# TanStack Table input-instability render loop freezing data tables

**Date:** 2026-05-18
**Project:** A Next.js admin panel I maintain (multi-tenant SaaS, fully client-rendered data tables)
**Stack:** Next.js 16.2.4 · React 19.2.5 · TanStack Query 5.100.5 · TanStack Table v8 · next-intl 4.9.1 · NextAuth v4.24.14
**Severity:** UX-breaking (any data table appeared frozen forever during initial load whenever a user interacted with the page during the loading window)
**Time to root cause:** ~3 hours of investigation across multiple iterations, after several wrong hypotheses
**Resolution:** Single infrastructure change in the shared `GlobalDataTable` component (~12 net lines)

---

## TL;DR

**The bug:** Any page that rendered the shared `<GlobalDataTable>` while its data was still loading silently entered a self-perpetuating React render loop. The loop alone made the page load ~2-3× slower than it should. When the user *also* interacted with anything during that loading window (clicked a language switcher, navigated via sidebar, etc.), the additional concurrent work pushed the main thread past saturation, microtasks could no longer drain, `response.json()` never got to complete, and the table appeared permanently frozen on the skeleton state.

**The cause:** `GlobalDataTable` passed inline-created references to `useReactTable` on every render (`data: data || []`, `getCoreRowModel()`, default `searchableColumns = []`, etc.). TanStack Table v8 treats these as "changed inputs" and updates its internal state via `setState`, which re-renders `GlobalDataTable`, which re-creates the inline references — a self-perpetuating render loop. The loop is the upstream pathology; concurrent interaction work is the trigger that makes it visibly fatal.

**The fix:** Move the defaults and factory calls to module-level `const` declarations so they're created exactly once. `data` fallback uses a shared `STABLE_EMPTY_DATA`; the four row-model factories (`getCoreRowModel`, `getSortedRowModel`, `getFilteredRowModel`, `getPaginationRowModel`) become module-level singletons; destructured defaults for `searchableColumns` and `rowActions` use shared empty arrays.

After fix: render count goes from **227+ renders climbing indefinitely** to **~6-10 renders** that flatline after the data arrives. The page also loads at its true latency (~500ms instead of 2-3s), regardless of whether the user interacts during loading.

---

## How to remember this (plain-language mental model)

Skip this section if you want the technical depth. Re-read this section if you want a quick refresher.

### The bug in one paragraph (no jargon)

`GlobalDataTable` was creating fresh empty arrays and fresh helper functions every single time it re-rendered. `useReactTable` checks whether its inputs are "the same object as last time" (identity, not value), saw "different every time," and reacted by triggering yet another re-render. That re-render created more fresh references, triggered another, etc. — a self-feeding loop running ~15+ times per second. The loop alone didn't crash anything (React 19 yields between renders, so the page still loaded — just slower). But when the user clicked the language switcher mid-load, the click triggered a cascade of additional re-renders across the rest of the page (NextAuth session updates, locale changes, `router.refresh()`). Those extra re-renders piled into the same already-busy main thread. The combined load saturated the single JavaScript thread enough that the tiny piece of waiting work (`response.json()`'s continuation, sitting in the microtask queue) never got its turn to run. So even though the server had already sent the full response and the bytes were in the browser, the JS side never parsed them. The skeleton appeared permanently frozen.

### The two-condition framing

Both have to be true at the same time for the freeze to show up:

- **(A)** A `GlobalDataTable` is mounted with `isFetching: true` (i.e., user is looking at a data-table page during its initial-load window)
- **(B)** Significant concurrent work happening on the main thread (user clicked something that fans out to many re-renders — typically a language switcher, but any cascading interaction would do)

Just (A) alone → silent slowdown (page loads in 2-3s instead of <500ms). Just (B) alone → totally fine. (A) + (B) → permanent freeze.

The fix kills (A) at its source. Without (A) ever happening, (B) can never cause harm.

### Why "the click flooded the main thread" was the right intuition

JavaScript runs on a single thread. One CPU lane. Nothing runs in parallel — not even microtasks (Promise callbacks). They all share that one lane.

```
Single lane:  [ render ] [ µtasks ] [ render ] [ µtasks ] [ render ] [ µtasks ] ...
                  ↑                       ↑                       ↑
              one render             µtasks drain            another render
              at a time              between renders
```

The `response.json()` continuation lives in the microtask queue. It needs a turn. It only gets turns *between* renders. If renders come back-to-back-to-back without a meaningful gap, microtasks can squeeze in only briefly. If the gap is too small for the *several sequential* microtask hops `response.json()` needs to fully parse the body, the chain breaks and the Promise never resolves.

The render loop alone leaves gaps (small, but they exist). The cascade closes them.

### Why the fix isn't a null gate / early return

A `if (isLoading) return null;` at the page level (which one of our other pages happened to use for different reasons) **avoids the bug** by not running the buggy code. The bug is still there in `GlobalDataTable` — that other page just doesn't trigger it.

The actual fix **removes the bug** by making the code not buggy. Stable references mean `useReactTable` never sees "input changed," so it never schedules an internal re-render, so the loop never starts at all. Every consumer of `GlobalDataTable` benefits — no per-page gating needed.

### The mechanism in one sentence each

- **Bug:** render → new `[]` → `useReactTable` sees "input changed" → schedules re-render → new `[]` again → loop → microtasks starve → `response.json()` never finishes.
- **Fix:** every render uses the SAME `[]` (module-level constant) → `useReactTable` sees "input unchanged" → does nothing → no re-render → no loop → microtasks run normally → data loads cleanly.

### Mental model: closer to "busy-wait infinite loop on a single-CPU machine"

This is **not** a stack overflow (no memory exhaustion, no crash). It's a single-thread infinite-loop problem: the loop hogs the CPU so the queued work that *would* unblock everything never gets a slot to run. If you killed the loop, the queued continuation would fire immediately and everything would unfreeze. The fix kills the loop.

### Why some pages were safe

| Page | Why it didn't freeze |
|---|---|
| Page A (uses `GlobalDataTable`, but with `if (isLoading) return null` gate above) | Table never mounts during loading, so (A) never holds. |
| Page B (uses a custom inline loading UI, no `GlobalDataTable`) | No `useReactTable`, no loop possible. |
| Pages C, D, E (use `GlobalDataTable`, render immediately during loading) | None of the above. (A) holds. (B) plus (A) → freeze. |

After the fix: all pages, regardless of gating pattern, are safe.

---

## Symptoms (as originally reported)

The bug evolved through three distinct reports, each shifting the suspect surface:

1. **Initial report.** "Sidebar OK but main content freezes when I click something during a page load." Reproducible by navigating to a list page and clicking the language switcher while the table skeleton was still showing. The sidebar remained interactive; the main content area was stuck.

2. **After first fix attempt.** "Now language change during data-table loading freezes the page completely." A new symptom appeared after a commit that tried to address the initial report by adding a route-level Suspense fallback (`loading.tsx`) and removing mount-gates from layout components.

3. **After a second fix attempt that memoized props in one specific consumer page.** "Still happens. Also happens on several other list pages. Some pages are unaffected." Affected page screenshot showed the page header rendered in the new locale (so the language change *had* visually applied) but the table region remained a single skeleton block. The user's tenant had only **51 active rows on the primary table and 2 rows on another**, definitively ruling out data-volume hypotheses.

The freeze was reliably reproducible in both dev and staging. Server logs showed the API endpoint completed successfully (`GET /api/admin/<resource> 200 in 667ms`).

## What made this hard

- **Misleading partial fixes.** Every wrong hypothesis seemed plausible enough to ship a fix for. Each fix changed the symptom slightly without resolving the underlying issue, which made the problem look like multiple separate bugs.
- **No error.** No exception thrown, no 500, no toast. Just a skeleton that never resolved. Devtools showed the request as "stalled" but with a successful response in the buffer.
- **"Freeze" was misleading vocabulary.** The main thread was never genuinely locked — the sidebar remained interactive throughout. The symptom *looked* like a CPU freeze but was actually a stuck async state.
- **Modern stack.** Next.js 16 + React 19 + concurrent rendering + React StrictMode + Fast Refresh produce a lot of incidental noise (StrictMode double-renders, double effect mounts, Fast Refresh hot reload pulses) that obscured the actual signal.

---

## Hypotheses tried and ruled out

The user-facing chronology, with what evidence killed each one. Each was reasonable based on what was known at the time — they were all systematically eliminated by the next piece of evidence, which is the process working as intended even when it feels slow.

### Hypothesis 1: Hydration mismatch in sidebar mount-gating

**Theory:** The `SidebarMenuButton` and `AppLayout` mount-gates (returning `null` until a `useEffect`-set `mounted` flag flipped) were causing a hydration flash that, combined with a navigation click during load, deadlocked the React commit.

**Action taken:** Removed both mount-gates. Added a route-level `loading.tsx` so suspended client routes had a proper Suspense fallback instead of falling through to upstream `null`.

**What killed it:** The original "click sidebar during load" symptom changed, but a *new* freeze appeared specifically on data-table pages during language change. New surface area, same underlying issue.

**Lesson:** Bundling a "while we're here" fix with a real fix poisons the diagnostic signal. The two changes should have been separated into independent commits so we could tell which one introduced the regression.

### Hypothesis 2: Large-payload (5000 rows) + react-table mounting cost

**Theory:** The main data table fetches with `limit=5000`. The response (~1.25 MB JSON) would block the main thread for ~100ms on `JSON.parse`, and `useReactTable` mounting 5000 row instances would block another ~200ms. Stacked with language-change re-renders, this would explain the freeze duration.

**Action taken:** Memoized `columns`, `queryKey`, `rowActions`, `apiRoute`, `searchableColumns` in the consumer page; conditionally mounted modals only when open; converted selection-set lookups from O(n²) `.includes()` to O(1) `Set.has()`.

**What killed it:** The maintainer pushed back: *"The table fetches 5000 rows, but it's not supposed to freeze when the user clicks something — it's supposed to be non-blocking even so. Dig deeper."* They were right. Async fetch is non-blocking, and the memoization didn't fix the freeze.

Then they added the killer data point: their tenant had only **51 rows** on the table they were testing. Data volume was conclusively not the cause.

**Lesson:** When the user insists a hypothesis is wrong and you can't see why, **trust them and dig further**. The "obvious cause" being wrong is a strong signal that the architecture has a non-obvious problem. Continuing to fix the obvious cause is whack-a-mole.

### Hypothesis 3: `loading.tsx` + `router.refresh()` Suspense flash

**Theory:** The `loading.tsx` added earlier created a Suspense boundary at the route level. `router.refresh()` (fired during the language change flow) might cause React 19's stricter Suspense rules to drop back to the fallback while it re-rendered the route, unmounting the in-flight `useQuery` and cancelling the fetch. The remount might leave the new query in a stuck state.

**What I proposed:** Wrap `router.refresh()` in `startTransition()` so it's a non-urgent update that doesn't trigger fallback display.

**What killed it (later, via instrumentation):** When I finally added diagnostic logs, the `[TABLE-DEBUG]` instance ID stayed the **same** across the entire freeze period. There was no unmount-remount cycle. The Suspense fallback was never activating. Same component instance throughout, just re-rendering itself 227+ times.

**Lesson:** Stop theorizing in the absence of evidence. I was making confident architectural arguments about Suspense behavior, when ten minutes of instrumentation would have killed that hypothesis instantly.

This was the turning point where I formally committed to **Phase 1: gather evidence at component boundaries before proposing fixes**.

---

## The systematic debugging process

The instrumentation phase is what actually solved this. Three diagnostic edits, each focused on a specific question.

### Round 1: Lifecycle and lifecycle-only

Added to `GlobalDataTable`:
- A `useRef<string>` instance ID generated once on first render, so we could tell across log lines whether we were looking at the same component instance.
- A `useRef<number>` render counter.
- Logs on: every render, mount, unmount, `queryFn` start, response received, JSON parsed, queryFn return, and queryFn throw.

Added to the language-change hook:
- Timestamped logs at each await step: before/after `mutateAsync`, before/after `update`, before/after `router.refresh`.

User reproduced and shared 200+ lines of console output. The findings, summarized:
- Same instance ID throughout. **Not a remount loop.**
- `queryFn:START` fires twice (StrictMode double-mount in dev, normal behavior).
- `queryFn:THROW {AbortError}` fires for the first invocation (StrictMode cleanup aborts the first fetch — normal).
- `queryFn:RESPONSE {status: 200, ok: true}` fires for the second invocation (server returned successfully).
- **`queryFn:JSON_PARSED` and `queryFn:RETURN` NEVER fire.** No `THROW` either. `response.json()` is a Promise that nothing settles.
- `[TABLE-DEBUG] render` keeps firing every ~30-80ms forever with the same state (`isFetching: true`).

This was the diagnostic moment that reframed the entire problem. **Not a network bug. Not a cancellation bug. A render loop starving the microtask queue.**

### Round 2: Is the loop in the parent or the child?

Added a parent render counter to the consumer page:
- Stored in `window.__parentRenders` so it survives across hot reloads
- Logged on every parent render

User reproduced again. Result:
- `[PARENT-DEBUG] render #1, #2` — and then never again. Parent rendered only twice (the StrictMode dev pair).
- `[TABLE-DEBUG] render` continued climbing past 225.

**Conclusive: the loop is entirely inside `GlobalDataTable`**. The parent stops re-rendering after its initial StrictMode pair; the child re-renders itself forever.

### Round 3: Network tab confirms the microtask-starvation hypothesis

User shared:
- Chrome DevTools Network tab showed two requests for the resource: one cancelled (StrictMode abort), one **stalled** (still listed as pending despite the server having long-since responded).
- Server log: `200 in 667ms`.

So the response existed in the browser buffer. The fetch Promise had resolved (we saw `queryFn:RESPONSE` fire). But `response.json()` was queued behind a render loop that wouldn't yield. The browser's Network tab interprets "JS never finished reading the body" as "stalled" — which is technically accurate.

---

## Root cause

The render loop was caused by `useReactTable` reacting to inline-created reference props in `GlobalDataTable`.

### The mechanism, step by step

1. `GlobalDataTable` renders. In its `useReactTable(...)` call, it passes:

   ```tsx
   useReactTable({
     data: data || [],                                  // fresh [] every render when data is undefined
     columns: columnsWithActions,                       // memoized — OK
     getCoreRowModel: getCoreRowModel(),                // factory called every render → fresh fn
     getSortedRowModel: getSortedRowModel(),            // same
     getFilteredRowModel: getFilteredRowModel(),        // same
     getPaginationRowModel: getPaginationRowModel(),    // same
     state: { sorting, columnFilters, globalFilter, pagination },  // fresh object every render
     onSortingChange, onColumnFiltersChange,            // setState from useState — stable
     onGlobalFilterChange, onPaginationChange,
     globalFilterFn,                                    // useCallback, but with unstable deps
   });
   ```

2. Plus two destructured defaults at the prop level:

   ```tsx
   function GlobalDataTable({
     searchableColumns = [],     // fresh [] every render
     rowActions = [],            // fresh [] every render
     ...
   }) {
   ```

3. `useReactTable` internally checks input identity for cache invalidation. When `data` reference changes, it invalidates its core row model cache and updates its internal state via `setState`.

4. That internal `setState` triggers a re-render of `GlobalDataTable`.

5. On re-render, step 1 happens again — `data || []` creates yet another fresh `[]`, the factories return new functions, the destructured defaults create new empty arrays. `useReactTable` sees changed inputs again. `setState` again. Re-render again.

6. Loop continues at React's render rate (~15 logical renders/sec). Each render takes ~5-10ms of CPU. With StrictMode in dev doubling that, the main thread is busy a large fraction of every 100ms — but React 19's concurrent renderer DOES yield between renders, so the thread isn't fully starved by the loop alone.

7. The Promise from `await response.json()` needs a microtask slot to run its continuation (the body parse). Microtasks run in the gaps between tasks. With the main thread in a tight render loop, those gaps shrink — but they don't disappear entirely. Without additional concurrent work, microtasks eventually scrape through and the page loads, just slowly (~2-3 seconds for an endpoint that would normally complete in ~500ms).

8. The full freeze only occurs when **additional concurrent work piles on top of the running loop** — typically a user interaction like clicking the language switcher, which fires several heavy operations back-to-back (a TanStack Query mutation, a NextAuth session update with its own re-broadcast, and `router.refresh()` triggering an RSC re-fetch). Each of those produces its own renders in unrelated components, all competing with the looping `GlobalDataTable` for main-thread time. At that point the thread saturates, the gap between tasks closes to zero, and microtasks can't run. `response.json()` is stuck waiting for a slot that never comes. **That** is when the user sees a permanent freeze.

### Why this is a two-condition bug (this matters)

The bug only **visibly manifests** as a freeze when both of these are true simultaneously:

| Condition | Source |
|---|---|
| **(A)** `GlobalDataTable` is on screen in `isFetching: true` state | Any list page during its initial load window |
| **(B)** Significant concurrent main-thread work is happening | A user interaction (language change, sidebar click triggering navigation, etc.) or anything else that fans out into a wave of re-renders across the tree |

If only (A) holds: the loop runs, main thread is stressed but not saturated, microtasks scrape through, page eventually loads — *slower than it should* but visibly fine. This is what happens when you sit and wait on a list page. The pathology is present, but invisible.

If only (B) holds: no loop is running because either `data` is already loaded (stable reference from TanStack Query cache, so `data || []` evaluates to `data`, not a fresh `[]`) or the page doesn't use `GlobalDataTable`. The interaction completes normally.

(A) + (B): main thread saturated, microtask queue cannot drain, `response.json()` stuck, permanent skeleton. Freeze.

This is why the bug looked timing-dependent and click-dependent. It IS timing-dependent — but the loop is the upstream condition that makes the click-triggered concurrent work fatal instead of just slow. **The fix eliminates condition (A) entirely**, so condition (B) becomes harmless regardless of how many things the user clicks during loading.

### Why specifically these pages

| Page | Uses `GlobalDataTable`? | Skeleton visible during load? | Pathology (loop) active? | Visible freeze on interaction? | Why |
|---|---|---|---|---|---|
| Heaviest data table | Yes | Yes | **Yes** | **Yes** | Table mounts in `isFetching: true` state, loop runs. Click anything → saturation → freeze. |
| Second table | Yes | Yes | **Yes** | **Yes** | Same |
| Third table | Yes | Yes | **Yes** | **Yes** | Same |
| Settings-style page | Yes | **No** — `if (isLoading) return null;` gates the entire page | No | No | `GlobalDataTable` isn't mounted during loading, so the loop never starts. When the table finally mounts, gating queries have resolved and `data` already has a stable reference from TanStack's cache. |
| Activity log | **No** — uses plain `<Card>` + inline `<Skeleton>` | Yes, but custom | No | No | `useReactTable` is never involved, so no loop is possible. |

The pages where (A) was true silently ran the loop even when the user wasn't interacting. They just loaded ~2-3x slower than they should have, which nobody noticed because "slow load" looks identical to "small backend latency." The freeze only became *visible* when condition (B) tipped the system into full saturation.

In other words: the bug existed in every page that used `GlobalDataTable`, but only manifested as a visible freeze when both the loading skeleton was visible to the user AND the user interacted during that window. Once data loaded, `data` was a stable TanStack Query cache reference and the loop stopped silently.

This is also why a prior memoization patch on the consumer side didn't fix it — that addressed the *parent* passing unstable props, but the loop was driven by `GlobalDataTable`'s *own* internal references to the row-model factories and the `data || []` fallback. The parent fix was good hygiene but solved a different (smaller) problem.

---

## The fix

In the shared `GlobalDataTable` component:

```tsx
// Module-level stable references — used as fallback for `data` and as singletons
// for react-table's row-model factory functions. Recreating these inline in the
// component body produces fresh references every render, which causes
// `useReactTable` to update its internal state, which triggers another render,
// in a self-perpetuating loop that starves microtasks and prevents Promise
// continuations (e.g., from `response.json()`) from ever resolving.
const STABLE_EMPTY_DATA: readonly any[] = [];
const STABLE_EMPTY_SEARCHABLE: readonly never[] = [];
const STABLE_EMPTY_ROW_ACTIONS: readonly never[] = [];
const STABLE_CORE_ROW_MODEL = getCoreRowModel();
const STABLE_SORTED_ROW_MODEL = getSortedRowModel();
const STABLE_FILTERED_ROW_MODEL = getFilteredRowModel();
const STABLE_PAGINATION_ROW_MODEL = getPaginationRowModel();
```

Then in the component:

```tsx
export default function GlobalDataTable<TData extends Record<string, any>>({
  // ...
  searchableColumns = STABLE_EMPTY_SEARCHABLE as unknown as (keyof TData)[],
  // ...
  rowActions = STABLE_EMPTY_ROW_ACTIONS as unknown as RowAction<TData>[],
  // ...
}: GlobalDataTableProps<TData>) {
  // ...
  const tableData = (data as TData[] | undefined) ?? (STABLE_EMPTY_DATA as TData[]);
  const table = useReactTable({
    data: tableData,
    columns: columnsWithActions,
    getCoreRowModel: STABLE_CORE_ROW_MODEL,
    getSortedRowModel: STABLE_SORTED_ROW_MODEL,
    getFilteredRowModel: STABLE_FILTERED_ROW_MODEL,
    getPaginationRowModel: STABLE_PAGINATION_ROW_MODEL,
    // ...
  });
}
```

### Why module-level (not `useMemo`)

The factories from `@tanstack/react-table` return functions of shape `(table) => () => RowModel<TData>`. The **outer** factory call (`getCoreRowModel()`) is what we cache once. The **inner** function gets called by `useReactTable` per table instance, creating a per-instance memoizer. So sharing the outer function across all `GlobalDataTable` instances is correct — each instance still gets its own memoizer state.

`useMemo(() => getCoreRowModel(), [])` would also work, but it'd allocate one closure per instance. Module-level allocates zero closures per instance. Functionally identical, slightly more efficient, clearer intent.

### What was deliberately NOT changed

- **API surface of `GlobalDataTable`** — drop-in compatible for all existing consumers. No prop renames, no required new props.
- **Behavior of `useReactTable`** — same table behavior, same UX, just fewer renders.
- **The earlier consumer-side memoization** — kept. It's good hygiene independent of the freeze fix.
- **The `loading.tsx`** added earlier — kept. It wasn't the cause.
- **Mount-gate removals** in layout components — kept. Independent of this issue.

---

## Verification

Console output from staging after the fix:

```
[TABLE-DEBUG 8f4jk r2 40329ms] MOUNT
[TABLE-DEBUG 8f4jk r2 40332ms] queryFn:START
[TABLE-DEBUG 8f4jk UNMOUNT 40340ms]                ← StrictMode cleanup
[TABLE-DEBUG 8f4jk r2 40343ms] MOUNT               ← StrictMode real mount
[TABLE-DEBUG 8f4jk r2 40345ms] queryFn:START
[TABLE-DEBUG 8f4jk r3 40359ms] render {isFetching: true, hasData: false}
[TABLE-DEBUG 8f4jk r4 40367ms] render {isFetching: true, hasData: false}
[TABLE-DEBUG 8f4jk r4 40494ms] queryFn:THROW {AbortError}           ← fetch #1 dying
[TABLE-DEBUG 8f4jk r4 41236ms] queryFn:RESPONSE {status: 200, ok: true}
[TABLE-DEBUG 8f4jk r4 41244ms] queryFn:JSON_PARSED                  ← ✅ NOW FIRES
[TABLE-DEBUG 8f4jk r4 41244ms] queryFn:RETURN {path: 'dataKey', count: 0}
[TABLE-DEBUG 8f4jk r5 41246ms] render {isFetching: false, hasData: true, dataLen: 0}
[TABLE-DEBUG 8f4jk r6 41250ms] render {isFetching: false, hasData: true, dataLen: 0}
[TABLE-DEBUG 8f4jk r7 41402ms] render {isFetching: false, hasData: true, dataLen: 0}
[TABLE-DEBUG 8f4jk r8 41405ms] render {isFetching: false, hasData: true, dataLen: 0}
(stops)
```

Render count flatlines at `r8`. `queryFn:JSON_PARSED` and `queryFn:RETURN` both fire — proving microtasks are no longer starved. `isFetching` transitions `true → false`. Data appears. Skeleton goes away.

Same result confirmed across multiple consumer pages with different table sizes (small datasets and the heaviest 51-row payload).

---

## Lessons learned

### About the bug itself

1. **`useReactTable` is reference-sensitive.** Documented in TanStack Table v8's [stable-references guidance](https://tanstack.com/table/v8/docs/guide/data#give-data-a-stable-reference) for `data` and `columns`, but the row-model factory functions and prop defaults are equally important and less obvious. The factories *look* like they should be called per-render because that's how TanStack's own docs show them — but calling them per-render with state-controlled tables can produce this loop.
2. **Microtask starvation is a real failure mode.** It's not theoretical. A tight render loop in one subtree can block Promise continuations elsewhere. Symptoms look exactly like a hung network request because that's the most visible thing that fails.
3. **The "freeze" wasn't a freeze.** The main thread was busy but not locked. Sidebar (different subtree) stayed interactive because its renders were rare. The looping subtree monopolized event-loop turns by re-rendering instead of yielding.
4. **React StrictMode dev double-renders, AbortErrors from cleanup, Fast Refresh pulses** — all added noise but none of them caused the bug. The bug existed identically in production builds without StrictMode. Don't blame dev-mode noise for behavior that reproduces in production.
5. **Microtask starvation has a threshold.** A render loop doesn't automatically equal a frozen UI — React 19 yields between renders, so a loop running alongside *nothing else* still leaves enough gaps for microtasks to scrape through. The visible freeze emerged only when concurrent interaction work piled on top of the running loop and pushed the main thread past saturation. Without that interaction trigger, the bug was a silent ~2-3× slowdown on initial page load that nobody had noticed. The first instinct — "render loop is the freeze" — was *almost* right but missed the threshold dynamic.

### About the debugging process

1. **Theorizing without evidence is a trap, especially when you sound confident.** Three wrong hypotheses in a row, each backed by plausible-sounding architectural reasoning, none of them correct. The fourth round — instrumentation — solved it in two iterations.
2. **No fixes without root cause investigation first.** Each premature fix changed the symptom without resolving the cause, and made the next investigation harder by adding moving parts.
3. **Instrumentation cost is low; theorizing cost is hidden.** Adding `console.log` to a hook takes 30 seconds. Arguing about React 19 Suspense semantics takes hours and produces zero evidence. Choose instrumentation early.
4. **The user's pushback was correct.** When the maintainer said "5000 rows shouldn't freeze it, dig deeper," they were exactly right. Async fetch *isn't* supposed to block. Trusting that and digging further was what eventually led to the right answer.
5. **Three failed fixes = stop and question the architecture.** After the third unsuccessful patch, I stopped proposing patches and committed to gathering data. That re-framing was the actual fix.
6. **The user's intuition was the corrective signal a second time.** After the fix was deployed and verified, I initially explained the bug as "the click is incidental — the freeze happens regardless." The maintainer pushed back: "no, if I just wait it loads in 2-3 seconds; only when I click does it actually freeze." That correction reframed the mechanism from "loop ⇒ freeze" to "loop + concurrent interaction ⇒ saturation ⇒ freeze," which is what the evidence actually shows. The first model was a simplification that obscured the threshold dynamic. Listen carefully when a user describes behavior that contradicts your tidy explanation — they're seeing something you missed.

### About the code itself

1. **Stability of references matters more than most React tutorials let on.** "Just put it in useMemo" is fine advice for components you control, but library hooks that compare references internally need *module-level* singletons or they'll churn.
2. **Destructured defaults create new objects every call.** `function F({ items = [] })` is a fresh `[]` on every invocation. For arrays/objects passed to reference-sensitive hooks, that's a hazard.
3. **React Query and React Table both use internal `useState`.** They look pure-functional from the outside but they trigger their own re-renders. Treating them as "just hooks" misses this.
4. **`useReactTable` should arguably document this gotcha more prominently** in v8 — or use `Object.is` on each input internally before invalidating cached row models. Right now the danger is implicit.

---

## Timeline (engineering-day chronology)

- **Day 1, morning:** Maintainer reports "sidebar clicks during page load freeze the page."
- **Day 1, midday:** Diagnosed as mount-gate hydration issue. Removed mount-gates from sidebar + app-layout, added `loading.tsx`. Shipped to staging.
- **Day 1, afternoon:** Maintainer reports the freeze pattern has changed — now language change during data-table load freezes. Sidebar still works.
- **Day 2, morning:** Hypothesized 5000-row JSON + react-table mount cost. Memoized the consumer page's columns/queryKey/rowActions and conditionally mounted modals. Shipped.
- **Day 2, midday:** Maintainer reports the fix didn't work. Pushed back on the 5000-row theory: "non-blocking even so."
- **Day 2, afternoon:** Hypothesized `loading.tsx` Suspense flash. Maintainer shared screenshot showing the affected page in the frozen state.
- **Day 2, late afternoon:** Maintainer added the killer evidence — only 51 rows on the heavy table. Data volume conclusively ruled out.
- **Day 2, evening:** Stopped theorizing. Added detailed instrumentation logs. Asked maintainer to reproduce.
- **Day 3, morning:** First instrumented reproduction showed: same component instance throughout (no remount), `queryFn:RESPONSE` fires (200 OK), `queryFn:JSON_PARSED` never fires, 200+ renders climbing.
- **Day 3, midday:** Added parent render counter to consumer page. Second reproduction confirmed: parent renders 2 times (StrictMode), child renders 200+ times. Loop is entirely inside `GlobalDataTable`. Maintainer also confirmed via DevTools Network tab that the response had reached the browser (server log: 200 in 667ms).
- **Day 3, afternoon:** Identified the unstable inputs to `useReactTable`. Applied module-level stable references for `data` fallback, row-model factories, and destructured defaults.
- **Day 3, late afternoon:** Verification reproduction. Render count flatlines at r8. `JSON_PARSED` + `RETURN` logs fire. Data loads. Skeleton goes away. Done.
- **Day 3, evening:** Removed instrumentation. Added gotcha note to project conventions doc. Wrote this postmortem.

Total: **~3 working days** of investigation. Of that, the actual root-cause-to-fix transition took about **2 hours** once instrumentation began. Most of the time was spent on the three wrong hypotheses.

---

## Code references

**Files changed in the final fix:**
- The shared `GlobalDataTable` component (`components/layout/data-table/global-data-table.tsx`) — module-level stable constants + updated `useReactTable` call

**Stack versions at the time:**
```json
{
  "next": "16.2.4",
  "react": "19.2.5",
  "@tanstack/react-query": "^5.100.5",
  "@tanstack/react-table": "(v8.x)",
  "next-intl": "^4.9.1",
  "next-auth": "^4.24.14"
}
```

---

## Appendix: the diagnostic instrumentation (since removed)

For reference, the logs that broke the case. These were added then removed once the fix was confirmed.

**In the language-change hook** (now removed):

```ts
const t0 = performance.now();
const log = (step: string, extra?: unknown) => {
  console.log(`[LANG-DEBUG ${(performance.now() - t0).toFixed(0)}ms] ${step}`, extra ?? "");
};
log("change:start", { from: current, to: next });
// ... log before/after each await ...
```

**In `GlobalDataTable`** (now removed):

```tsx
const instanceIdRef = React.useRef<string>("");
if (!instanceIdRef.current) {
  instanceIdRef.current = Math.random().toString(36).slice(2, 7);
}
const renderCountRef = React.useRef(0);
renderCountRef.current += 1;
const dlog = (event: string, extra?: unknown) => {
  console.log(
    `[TABLE-DEBUG ${instanceIdRef.current} r${renderCountRef.current} ${performance.now().toFixed(0)}ms] ${event}`,
    extra ?? ""
  );
};

useEffect(() => {
  dlog("MOUNT", { apiRoute, queryKey });
  return () => console.log(`[TABLE-DEBUG ${instanceIdRef.current} UNMOUNT ${performance.now().toFixed(0)}ms]`);
}, []);

// In queryFn:
queryFn: async ({ signal }) => {
  dlog("queryFn:START", { apiRoute });
  try {
    const response = await fetch(apiRoute, { signal });
    dlog("queryFn:RESPONSE", { status: response.status, ok: response.ok });
    const json = await response.json();
    dlog("queryFn:JSON_PARSED");
    // ... extraction ...
    dlog("queryFn:RETURN", { /* ... */ });
    return /* ... */;
  } catch (err) {
    dlog("queryFn:THROW", { name: (err as Error)?.name, message: (err as Error)?.message });
    throw err;
  }
},

// In render body:
dlog("render", { isFetching, isError, hasData: data !== undefined, dataLen: Array.isArray(data) ? data.length : "n/a" });
```

**In the consumer page component** (now removed):

```tsx
const parentRenderCount = (typeof window !== "undefined"
  ? ((window as any).__parentRenders = ((window as any).__parentRenders || 0) + 1)
  : 0);
console.log(`[PARENT-DEBUG ${performance.now().toFixed(0)}ms] ParentTable render #${parentRenderCount}`);
```

The instance-ID-based logs were the critical innovation: by stamping every log with a unique-per-mount ID, we could tell at a glance whether logs were coming from the same component instance (re-renders) or different instances (remount loop). That single piece of metadata is what definitively killed the Suspense-flash hypothesis and pointed us at internal-state oscillation as the cause.

---

*Postmortem authored by [Alvalens](https://github.com/Alvalens), after a multi-day staging-environment debugging session. All findings reproduced and verified.*
