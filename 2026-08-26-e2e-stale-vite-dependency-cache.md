# The E2E test that succeeded and failed at the same time — a stale Vite dependency pre-bundle cache

**Date:** 2026-08-26
**Project:** A customer-facing self-service onboarding app I work on — Vue 3 + Vite frontend, Node backend, pnpm workspace with a shared local package, run locally via Docker Compose.
**Stack:** Vue 3 + Vite (pnpm workspace) · Playwright E2E · Docker Compose (local dev)
**Severity:** Local development only — blocked reliable local E2E runs; never reproduced against the real deployed environment, and this suite isn't executed in CI at all (lint-only).
**Time to root cause:** ~2.5 hours, most of it spent on two reasonable-but-wrong hypotheses before the right instrumentation was added.
**Resolution:** Clear the frontend dev server's stale Vite dependency pre-bundle cache and restart the container — one command.

---

## TL;DR

**The bug:** a local Playwright E2E run would submit a record lookup, the backend would answer `200` with a valid record in the response body, and the UI would just... never advance. The test's own diagnostic logging showed success and failure reported by the same request at the same time.

**The cause:** the frontend dev server's Vite dependency pre-bundle cache (`node_modules/.vite/deps`) was **ten days stale**. A sibling workspace package it depends on had been rebuilt multiple times since, adding a new export along the way. Vite pre-bundles a locally-linked workspace package once at dev-server startup and doesn't deep-watch that package's *built output* the way it watches the app's own source files — so the dev server kept serving a pre-bundle snapshot from before the new export existed. Any code path that tried to import it hit a hard `SyntaxError` (ESM named exports are resolved statically, so this isn't a soft "undefined," it's a real module-resolution failure). That exception landed in a generic `try/catch` that discarded the actual error object before logging anything useful — so the one clue that would have identified the cause instantly was invisible in every log already being captured.

**The fix:** `rm -rf node_modules/.vite && docker compose restart <frontend-service>` — clear the stale cache, force Vite to re-optimize dependencies against the current build.

---

## How to remember this (plain-language mental model)

A bundler's dependency cache is a photograph, not a mirror. Vite pre-bundles a locally-linked workspace package once and then trusts that snapshot — it watches your own source files for changes, but it doesn't watch the *built output* of a symlinked sibling package the same way. Rebuild that package as many times as you like; the dev server keeps serving the old photograph until something forces a new one to be taken (a cache clear, a `--force` restart).

The failure mode is uniquely nasty because the browser genuinely does throw a real error — a `SyntaxError`, not a vague warning — but nobody was watching browser console output, and the application's own error handling had a bare `catch (error) { console.error("some generic message") }` that threw the real error away before logging anything useful. Two independently reasonable engineering decisions — "cache dependencies for speed" and "don't leak internal errors to a generic catch-all log line" — combine into a failure that looks like a flaky test and is actually a five-word fix, if only the real error had been visible.

**One sentence version:** the dev server was serving a ten-day-old snapshot of a package that had grown a new export since, the resulting import error was real but got silently swallowed, and the test just looked "stuck" instead of showing anyone why.

---

## Symptoms

Running the record-lookup step of the onboarding E2E flow locally, headed, against the local Docker stack:

```
[LookupHelper] Record lookup attempt 1 first submit: {
  recordReference: '...',
  response: {
    status: 200,
    url: 'http://localhost:8083/api/v1/onboarding/record-lookup',
    body: { code: undefined, message: undefined, items: 1 }
  }
}
[LookupHelper] Wait for /items fell back to detail-form detection; current URL: .../record-lookup
[LookupHelper] Detail form not visible before timeout; last URL: .../record-lookup
[LookupHelper] waitForItemsPage failed while still on record lookup - retrying attempt 2
```

A `200` response with a valid, non-empty item in the body — a genuinely successful backend call — followed immediately by the UI failing to advance and the test retrying. Success and failure, from the same request, simultaneously. This reproduced **deterministically**, on every local run, and against two unrelated backend integrations using completely different records — ruling out anything data- or integration-specific from the start. It never reproduced against the real deployed environment.

---

## What made this hard

- **The symptom actively pointed away from the real cause.** A `200` with valid data screams "the backend is fine, look at the frontend's rendering/routing logic" — which is true, but the actual defect wasn't in any application logic at all. It was one layer further out, in whether the frontend was even running the code its own source files said it should be running.
- **No code path implicated by the symptom had actually changed.** Diffing every file that plausibly explained a lookup-step stall against the base branch came back empty — ruling out "a recent code change broke this," but leaving no obvious next lead.
- **The real error was thrown away before it reached any log already being watched.** The lookup handler's own catch block deliberately discarded the caught error object (`void error`) and logged only a static, generic string. Every log stream already in use — Playwright's trace, the app's own diagnostic helper logging, Docker container logs — was blind to the one piece of information that would have solved this immediately, because the application itself never wrote it down anywhere.
- **The failure looked exactly like the shape of bug most likely to get dismissed as "local environment flakiness, not worth chasing."** Deterministic locally, never reproduced against the real deployed environment, not covered by CI at all. That combination is genuinely often a sign of a non-issue — which made it easy to under-invest in, right up until the fix turned out to be a one-line cache clear that unblocked every subsequent local test run.

---

## Hypotheses tried (and ruled out)

| Hypothesis | Why it seemed plausible | How it was ruled out |
|---|---|---|
| Stale local test data — leftover records from prior runs colliding with the new one | The shared local test account accumulates state across runs; the backend has an explicit duplicate-record dedup path that shows up in logs during failures like this | Cancelled 20 stale upstream records, TTL-expired 40 stale local records, reran the exact same scenario — identical failure, byte-for-byte same log signature |
| A stale/invalid AI verification model reference causing a downstream hiccup on an upload step, ahead of the lookup step | Local logs showed an unrelated 404 against a preview model id under a name that no longer existed | Found and fixed the (genuinely broken, worth fixing regardless) model reference, restarted the affected service, reran — no change to this specific failure |
| Backend-integration-specific timing or sandbox flakiness | First reproduced against one backend integration | Reproduced identically against a second, entirely different backend integration and a different record — ruled out anything data- or integration-specific |
| "It's fine, it's local-only and passes against the real deployed environment" | Deterministic-but-local-only failures are often genuinely not worth chasing | Rejected as a stopping point specifically *because* it was 100% reproducible and blocking every local run — a real, fixable defect, just not one visible from the outside |

Each ruled-out hypothesis moved the working theory from "something about this environment or data" toward "something in the app's own code, in a way invisible to every log already being captured" — which is what eventually justified reaching for raw browser instrumentation instead of continuing to poke at data and config.

---

## The systematic debugging process

**Step 1 — confirm nothing in the implicated code paths had actually changed.** `git diff` against the base branch for the lookup coordinator and the duplicate-record dedup service came back empty. This didn't explain the bug, but it correctly ruled out "a recent change caused this," which kept the investigation from chasing the wrong diff.

**Step 2 — let a failing run complete instead of stopping it early.** An earlier attempt had been interrupted mid-run to redirect debugging effort elsewhere, which meant no trace artifact existed to actually inspect afterward. Letting a subsequent run fail to completion produced a real `trace.zip`, video, and screenshot — the first genuine forensic evidence to work from, instead of live log-watching alone.

**Step 3 — the trace didn't show browser console output at useful fidelity, so instrument the browser directly.** Rather than keep reading source and guessing what the frontend does after a successful API response, the spec file was temporarily patched to listen at the browser level:

```ts
page.on("console", (msg) => console.log(`[DEBUG-CONSOLE:${msg.type()}]`, msg.text()));
page.on("pageerror", (err) => console.log("[DEBUG-PAGEERROR]", err.stack || err.message));
page.on("requestfailed", (req) =>
  console.log("[DEBUG-REQFAILED]", req.method(), req.url(), req.failure()?.errorText),
);
```

This is the instrumentation-at-the-boundary move: log what the browser is actually doing, unfiltered, rather than theorize about it from source code alone.

**Step 4 — the instrumentation surfaced a swallowed `console.error` immediately after the successful lookup response, but with no useful detail.** The app was already logging `"Record lookup error."` on every failed attempt — it had just been invisible until something was specifically listening for `console` events. The message itself carried zero information: the underlying `error` object had been explicitly discarded (`void error`) before the static string was logged.

**Step 5 — un-discard the error for one debug run.** Changing the catch block to log the real error object and its stack, temporarily, produced the answer on the very next run:

```
SyntaxError: The requested module '.../packages/shared/dist/esm/index.js'
does not provide an export named 'isManualOverrideWaived'
```

A module-resolution failure, not an application bug. This explained the entire symptom in one line: a genuinely successful backend response, followed by the frontend throwing while trying to run code that referenced a symbol its own bundled dependency graph claimed didn't exist — caught by generic error handling that hid the real cause from every log already in use.

**Step 6 — verify the export actually exists where it's supposed to.** `grep` confirmed the shared package's source *and* its built output both correctly define and export the symbol. So the artifact on disk was fine — which meant whatever was serving the running frontend wasn't reading the current build.

**Step 7 — compare what's being served against what's on disk.** Checking the mtime of the frontend dev server's Vite dependency pre-bundle cache against the shared package's most recent rebuild made the gap immediately obvious: the cache was **ten days old**; the shared package had been rebuilt (multiple times) since. Vite's dependency optimizer doesn't deep-watch a symlinked local workspace package's build output the way it watches the app's own first-party source — it snapshots once at startup and reuses that snapshot until something forces a re-optimize.

---

## Root cause

The frontend is a pnpm workspace member that imports a sibling `shared` package via a standard pnpm workspace symlink (`node_modules/@shared -> ../../../shared`). Vite's dependency pre-bundler (`node_modules/.vite/deps`) treats that linked package as a dependency to optimize once at dev-server startup — and, critically, does not deep-watch its *built output* for changes the way it watches the app's own first-party source files.

The shared package had been rebuilt multiple times across the ten days between the frontend dev container's last restart and this investigation, adding new exports along the way (including the one implicated here). The running dev server kept serving a pre-bundle snapshot from *before* one of those exports existed.

When a code path attempted to import that newly-added export, the browser threw a hard `SyntaxError` at import time — not a soft "undefined is not a function," a genuine module-resolution failure, because ESM named-export bindings are resolved statically at parse time, not looked up dynamically at call time. That exception propagated into a generic `try/catch` wrapping the lookup submit handler, which discarded the error object (`void error`) and logged only a static string — hiding the one clue that would have identified the cause immediately.

None of this had anything to do with any feature being worked on at the time. It was a pure artifact of local dev-container lifecycle: the shared package gets rebuilt independently of the frontend dev server's own restart cycle, and nothing forces the two to stay in sync automatically.

---

## The fix

```bash
rm -rf packages/frontend/apps/<app>/node_modules/.vite
docker compose restart <frontend-service>
```

Clearing the stale pre-bundle cache and restarting the dev server forces Vite to re-optimize its dependency graph against the shared package's current build output on next start.

### Why not just fix the swallowed-error catch block too?

Worth doing as a follow-up, but deliberately out of scope for the immediate fix: the catch block's `console.error("Record lookup error.")` without the underlying error is defensible as a *user-facing* safeguard (don't leak internals into a customer-facing toast/log). The actual gap is that there's no *developer-facing* channel — a dev-only verbose log, a Sentry-style captured exception, anything — that preserves the real error for debugging while still showing users a clean message. That's a small, separate, low-risk change; bundling it into an urgent unblock-local-dev fix would have been unrelated scope creep.

---

## Verification

Cleared the cache, restarted the frontend container, reran the exact same local scenario that had failed on the four prior attempts: the record lookup succeeded and the flow advanced to the next screen on the very next attempt — no retry needed, no console errors of any kind, and the full onboarding flow (identity verification, item assignment, contact details, completion) passed end to end in 2.3 minutes. Reran twice more from a clean state to confirm it wasn't a one-off — consistent pass both times.

---

## Lessons learned

### About the bug

1. **A bundler's dependency pre-bundle for a locally-linked workspace package is not automatically invalidated by rebuilding that package.** This is a general pnpm-workspace + Vite trap, not specific to any one feature — any dev container that stays up across multiple rebuilds of a sibling package is exposed to it. Worth checking for in any pnpm/Vite monorepo where a shared package gets rebuilt independently of the app that consumes it.
2. **Discarding an exception before logging it (`void error` + a static message) is a debugging tax paid by someone else, later, usually at the worst time.** It's a defensible choice for a genuinely user-facing surface — but the fix there is a separate dev-only or telemetry channel for the real error, not making it unrecoverable everywhere, including local development where there's no "user" to protect from the details.
3. **ESM named-export resolution failures are hard failures, not soft ones.** A missing named export doesn't quietly resolve to `undefined` and let the calling code fail later on a property access — it throws immediately, at import time, for the whole module. That's actually useful once you know to look for it: a `SyntaxError` mentioning "does not provide an export named X" is a very specific, very traceable signature once it's actually visible.

### About the process

1. **When a generic `catch` block sits directly in the failure path, temporarily un-discarding its error is one of the highest-value five-minute changes available.** Significant time went into data-cleanup and unrelated-config hypotheses that this single change would have made unnecessary from the start. Reading the code and reasoning about it is the right first move; it should yield quickly to "attach a listener and look" the moment a suspicious `catch` block is spotted.
2. **Letting a failing run complete, rather than killing it early to save time, produces the trace artifact that makes the next round of debugging faster.** A partial/interrupted run leaves no forensic evidence; a completed failing run leaves a trace, a video, and a screenshot — worth the extra wall-clock time whenever the failure is genuinely under investigation rather than just being reproduced for the third time.
3. **"Deterministic locally, absent in the real environment, not covered by CI" is not, by itself, evidence that a bug isn't worth chasing.** It's a common shape for both real-but-environment-specific bugs and non-issues, and the only way to tell them apart is to actually find the mechanism — which in this case turned out to be a one-command fix that had been silently blocking every local E2E run for over a week.

### Prevention / detection

- Document (and ideally script, e.g. a `dev:rebuild-shared` helper that also clears the consuming app's Vite cache) "rebuilt a linked workspace package but the dev server didn't pick it up" as a known, named class of problem, so it isn't independently rediscovered from scratch the next time it happens.
- Consider whether the shared package's rebuild step could touch a file the frontend's dev server *does* watch (even a trivial timestamp file), to trigger Vite's own change detection instead of relying on manual cache-busting.
- Add a dev-only (non-user-facing) verbose error log at the site of any `catch` block that currently discards its error before logging — cheap insurance against the next multi-hour chase for a bug that a real stack trace would have named in one line.

---

## Timeline

- **T+0:00** — Local E2E run stalls at the record-lookup step; backend response is `200` with valid data, UI never advances.
- **T+0:15** — Confirm via `git diff` that no recently-changed code touches the implicated lookup or dedup paths.
- **T+0:20 – T+1:15** — Rule out stale local test data (cleanup performed, no change) and a genuinely-broken-but-unrelated AI model config reference (fixed on its own merits, no change to this symptom).
- **T+1:20** — Reproduce identically against a second, unrelated backend integration — rules out data- or integration-specificity.
- **T+1:35** — Let a run fail to completion instead of interrupting it early; recover a full Playwright trace as the first real forensic evidence.
- **T+1:50** — Add raw `page.on('console'/'pageerror'/'requestfailed')` listeners directly in the spec; immediately surfaces a swallowed `console.error("Record lookup error.")` on an otherwise-`200` response.
- **T+2:00** — Un-discard the caught error for one debug run; the real cause appears instantly — a `SyntaxError` on a missing named export from the shared package's build.
- **T+2:10** — Confirm the export exists in both source and the built artifact on disk, ruling out a bad shared-package build.
- **T+2:20** — Compare the frontend's Vite pre-bundle cache mtime against the shared package's last rebuild — ten days stale.
- **T+2:30** — Clear the cache, restart the frontend container; rerun passes end to end on the first attempt; two additional clean reruns confirm it wasn't a fluke.

---

## Appendix — diagnostic instrumentation used

**Browser-level instrumentation added temporarily to the failing spec (removed before commit):**

```ts
page.on("console", (msg) => console.log(`[DEBUG-CONSOLE:${msg.type()}]`, msg.text()));
page.on("pageerror", (err) => console.log("[DEBUG-PAGEERROR]", err.stack || err.message));
page.on("requestfailed", (req) =>
  console.log("[DEBUG-REQFAILED]", req.method(), req.url(), req.failure()?.errorText),
);
```

**Un-discarding the swallowed error for one debug run (removed before commit):**

```diff
  } catch (error) {
-   void error;
+   console.error("Record lookup error. DEBUG:", error, error instanceof Error ? error.stack : "");
    console.error("Record lookup error.");
    ...
```

**Confirming the shared package's build was actually current on disk, ruling out a bad build as the cause:**

```bash
grep -c "isManualOverrideWaived" packages/shared/dist/esm/domain/index.js
# -> present, correctly exported

ls -la packages/frontend/apps/<app>/node_modules/.vite/deps/_metadata.json
# -> ~10 days older than the shared package's last rebuild
```

**The fix, applied:**

```bash
rm -rf packages/frontend/apps/<app>/node_modules/.vite
docker compose restart <frontend-service>
```

---

*Maintained by [Alvalens](https://github.com/Alvalens). PRs and issues welcome if you spot inaccuracies or want to discuss a technique used in a postmortem.*
