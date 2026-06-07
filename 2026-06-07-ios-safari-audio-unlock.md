# The "enable audio" button that errored instead of unlocking — iOS/Safari autoplay, three layers deep

**Date:** 2026-06-07
**Project:** A Next.js interview-prep web app I work on (browser-based mock interviews with a spoken AI interviewer; questions are read aloud via cloud text-to-speech)
**Stack:** Next.js 16.2 · React 19.2 · cloud TTS (MP3 over `fetch`) · `HTMLAudioElement` · iOS WebKit autoplay policy
**Severity:** Feature-breaking on Apple devices — the AI interviewer was mute on iPhone/iPad/macOS Safari, and the interview loop is unusable without spoken questions
**Time to root cause:** Three staged iterations against a real device on a staging preview; each fix uncovered the next layer
**Resolution:** Rewrite of the audio-unlock primitive in a single speech-synthesis hook, plus a redesign of the enable-audio modal

---

## TL;DR

**The bug:** On every Apple browser (iOS Safari, iOS Chrome/CriOS, macOS Safari) the interview's spoken questions never played. A modal asked the user to "enable audio"; tapping it did nothing or threw an error toast. The audio session was never actually unlocked.

**The cause — three layers, peeled one at a time:**

1. **The unlock never unlocked.** The "unlock" function loaded a silent source and flipped an `isAudioUnlocked` flag via a 1-second timeout, but **never called `play()`**. iOS only unlocks audio when a real `play()` *resolves inside a user gesture*. So the unlock was a lie, and the first real TTS clip — a fresh `new Audio()` per clip, played after a network fetch, outside the gesture — was rejected by the autoplay policy.
2. **The priming clip was the wrong kind of file.** After making the unlock actually call `play()`, it still failed on iOS **Chrome (CriOS)** with `AbortError`, because the silent clip was a **`data:` URL** (which iOS WebKit aborts) and was **zero-length** (no decodable samples). The app's own device-precheck — which *could* play sound on the same device — used a **`blob:` URL**. That was the tell.
3. **A normal interrupt was misread as permission loss.** Once audio played, a **second dialog + error toast** appeared whenever the interview advanced to the next question. Advancing interrupts the current clip; iOS rejects the interrupted `play()` with `NotAllowedError`; the catch misread that as "the unlock didn't stick," **flipped audio back to locked**, and re-summoned the enable-audio modal.

**The fix:** Unlock for real by playing a short, *valid* silent WAV delivered as a **`blob:` URL** via `play()` on a **single persistent `<audio>` element**, synchronously inside the tap — then **reuse that one unlocked element** for every TTS clip (never `new Audio()` again). Serialize concurrent unlock attempts, back the unlocked flag with a **ref** so the click→unlock→flush chain reads it synchronously, and treat a post-unlock `NotAllowedError` as a **transient interrupt** — never re-lock, never re-open the dialog.

After the fix: the modal appears once, one tap enables audio, the interviewer speaks, and advancing questions switches cleanly — no second dialog, no error toast. Non-Apple browsers are unchanged (they start unlocked, with no modal).

---

## How to remember this (plain-language mental model)

Skip this if you want the technical depth. Re-read it if you want to reload the whole bug in your head.

### The bug in one paragraph (no jargon)

Apple's browsers refuse to play sound unless a real human tap "wakes up" the audio. The way you wake it up is specific: the instant the user taps, you must *actually start playing a sound* — not prepare to play one, not play one a moment later, but play it right then, in the same heartbeat as the tap. The old code only *got ready* to play (it loaded a silent file and assumed that counted). It didn't count. So when the actual question arrived a second later — after a round-trip to the server — the browser said "no human asked for this" and stayed silent. Worse, the button reported success, so the user believed they'd enabled audio when nothing had happened. When we finally made it play a real sound on tap, we handed it a file in a format iOS quietly refuses (`data:`), and an empty one at that — so it errored. Once we used the right kind of file, audio worked — but a second trap remained: when the interview moved to the next question, it *interrupted* the current one, and the browser reported that interruption using the *same error code* it uses for "you're not allowed to play audio." Our code couldn't tell the two apart, panicked, decided audio had become locked again, and popped the enable dialog a second time.

### The two facts about iOS audio that everything hinges on

1. **Unlock = a real, resolved `play()` inside the gesture** — on the element you intend to reuse. Not a load. Not a muted play. Not a deferred play. The promise from `el.play()` must resolve, and the call must be reached with no `await` between the tap and the line that yields to the event loop.
2. **The unlock is per-element.** The *specific* `<audio>` element you played inside the gesture stays playable afterward — even across later async `play()` calls. A *new* `Audio()` created later is a stranger and gets blocked again. So you must keep and reuse the one element you unlocked.

### The three-strike framing

Each fix exposed the next problem, like peeling an onion:

- **Strike 1 — the fake unlock.** Never called `play()`. → "enable does nothing / errors, no sound."
- **Strike 2 — the wrong file.** Real `play()`, but a `data:` + zero-length clip. → `AbortError`, specifically on iOS Chrome.
- **Strike 3 — the misread interrupt.** Real unlock works, but advancing questions throws `NotAllowedError` that we misclassified. → "double dialog + error toast, but sound still plays."

Only after all three were fixed did the feature work end to end.

---

## Symptoms

What was observed, in the order it surfaced across debugging sessions:

1. On iOS and macOS the interview needed "permission" to play sound. A modal popped up asking to enable audio; tapping **Enable** errored instead of unlocking. (Also: the modal looked machine-generated and didn't match the app's design system.)
2. After the first fix shipped to staging: still no sound in the interview — **but the device-precheck screen could play an audible test tone on the very same device.** The error captured in monitoring:
   ```
   AbortError: The operation was aborted.
   [Audio Unlock] Unexpected error: AbortError: The operation was aborted.
   UI Click: <the gradient "Enable audio" button>
   User-Agent: …(iPad; CPU OS 26_4_2 like Mac OS X)… CriOS/148… Mobile/15E148 Safari/604.1
   ```
3. On that same build: tapping **Enable** did nothing visible, and the manual "Speak question" button also produced no sound (because audio was never actually unlocked).
4. After the blob fix: audio worked — but a **double confirmation** appeared. The first (restyled) dialog worked; a second dialog then surfaced with an error toast, even though sound was still playing.
5. Final state: resolved — audio plays cleanly, with the enable step no longer getting in the way.

The single most useful diagnostic in the whole investigation was **the contrast in symptom 2**: the precheck could play sound on the same device. That ruled out "the platform can't autoplay" and reframed the question as *"what is the interview path doing that the precheck isn't?"*

---

## What made this hard

- **The unlock lied about success.** Because the old code set `isAudioUnlocked = true` on a timeout, every downstream signal said "audio is enabled." The failure surfaced later and elsewhere (at the first real `play()`), far from the actual defect.
- **The error vocabulary is overloaded.** iOS uses `NotAllowedError` for both "autoplay blocked (no gesture)" *and* "a `play()` you started got interrupted by a newer one." One name, two completely different stories — and the correct response to each is opposite (prompt to unlock vs. ignore).
- **It was browser- and transport-specific.** The `data:`-URL abort only reproduced on iOS Chrome (CriOS), not on desktop or in any emulator I had at the desk. It could not be reasoned out purely from code; the device had to tell us.
- **Two UIs competed for one job.** A proactive enable-audio modal and a reactive permission dialog both existed for the same capability. In the happy path only one should ever appear; a state regression made both fire.
- **A gesture is a fragile, invisible context.** "Inside the user gesture" isn't visible in the code — it's a runtime property that any `await`, `setTimeout`, or network call can silently break. Three separate places in the original flow leaked outside it.

---

## Hypotheses tried (and ruled out)

| # | Hypothesis | Evidence that killed it |
|---|---|---|
| H1 | It's a microphone / `getUserMedia` permission problem. | The failing action was audio *output* (TTS), and the breadcrumb pointed at the audio-unlock path. Mic capture worked. |
| H2 | Apple just can't autoplay; nothing to be done. | The device-precheck plays an audible tone **on the same device**. The platform can play after a gesture — so could we. |
| H3 *(fix 1)* | The unlock just needs to actually call `play()` on a reused element. | Right direction, but staging still threw `AbortError` on iOS Chrome. Necessary, not sufficient. |
| H4 | The `AbortError` is two concurrent unlock calls aborting each other. | Partly real — the interview auto-speaks each question, firing a doomed non-gesture unlock that can collide with the tap — and worth fixing via serialization, but **not** the primary cause. The clip itself was. |
| H5 *(fix 2)* | iOS WebKit aborts `data:` audio loads, and the clip is zero-length — use a valid silent WAV as a `blob:` URL, like the precheck does. | This made sound play. Confirmed on-device. |
| H6 | The leftover second dialog is the separate mic-permission dialog. | Wrong: it was the **audio** enable flow re-triggering, because a post-unlock `NotAllowedError` flipped the unlocked flag back to false. |
| H7 *(fix 3)* | A post-unlock `NotAllowedError` is an *interrupt* (next question replacing the current clip), not a permission loss — stop re-locking on it. | Treating superseded requests as silent and not re-locking killed the double dialog and the toast. Resolved. |

---

## The systematic debugging process

1. **Read the entire audio path before touching anything** — the speech-synthesis hook (unlock + queue + playback), the enable-audio modal, the page wiring (including the fallback permission dialogs), and the interview state machine (who calls `speakText`, and when).
2. **Found the fake unlock by reading, not running.** The unlock set `audio.src`, listened for `canplaythrough`, and otherwise resolved on a `setTimeout(…, 1000)` — but **never called `play()`**, and set `audio.muted = true`. None of that unlocks iOS. Meanwhile each clip was a fresh `new Audio(blobUrl)`, and the real playback was deferred with `setTimeout(500)` *after* a fetch — three independent ways to land outside the gesture.
3. **Shipped fix 1 to a staging preview and let the device reveal the next layer.** The `AbortError` + `CriOS` UA + "precheck works" triad pointed straight at *how the silent clip was delivered*. Diffing against the working precheck surfaced the difference: the precheck plays a **`blob:` URL** (`URL.createObjectURL`), never a `data:` URL. iOS WebKit is known to abort `data:` audio loads — and our silent WAV was additionally **zero-length** (a 44-byte header with an empty `data` chunk: nothing to decode).
4. **Fix 2: a valid silent WAV as a blob.** Hand-built a real WAV (8 kHz, ~0.1 s of 8-bit silence) → `Blob` → object URL → `play()` on the reused element. Serialized concurrent unlock attempts and moved the unlocked flag into a ref. Sound played.
5. **Fix 3: classify the interrupt correctly.** With sound working, the residual double-dialog came from advancing questions: the interview does `stopSpeaking(); speakText(next)`, interrupting the in-flight clip. iOS rejected the interrupted `play()` with `NotAllowedError` (not the `AbortError` already being swallowed). The catch then ran its "unlock didn't stick → re-lock → re-prompt" branch. Fixed by (a) dropping any request that's been superseded, silently, and (b) once unlocked, treating `NotAllowedError` as a transient interrupt — no re-lock, no toast, no dialog.

---

## Root cause (with code references)

All in the speech-synthesis hook (`hooks/useSpeechSynthesis.ts`) unless noted.

### Layer 1 — the unlock never unlocked

```js
// BEFORE — never plays, fakes success on a timeout, and mutes
const audio = new Audio();
audio.muted = true;                       // muted plays don't count as an unlock
audio.src = "data:audio/wav;base64,…";    // only loads a source
setTimeout(() => resolveOnce(true), 1000); // resolves true regardless
setIsAudioUnlocked(canPlay);              // → true, but nothing was unlocked
```

Every clip was a brand-new element (iOS keeps only the gesture-unlocked element playable):

```js
// BEFORE — a stranger element per clip; iOS blocks it
const audio = new Audio(audioUrl);
```

…played after a deferral that leaves the gesture entirely:

```js
// BEFORE — real play() happens 500ms + a network fetch after the tap
setTimeout(async () => { await speakText(text, options); }, 500);
```

### Layer 2 — the silent clip was the wrong kind

The priming clip was a `data:` URL (aborted by iOS WebKit) **and** zero-length (`data` chunk size 0 — nothing to play). iOS Chrome surfaced this as `AbortError` from `el.play()`.

### Layer 3 — interrupt misclassified as permission loss

```js
// BEFORE — any NotAllowedError re-locks audio and re-opens the modal
if (error instanceof DOMException && error.name === "NotAllowedError") {
  markUnlocked(false);                 // ← re-summons the enable-audio dialog
  pendingTextRef.current = { text, options };
  callbacks?.onPermissionDenied?.();
  return;
}
```

When the next question interrupts the current clip, the interrupted `play()` rejects with `NotAllowedError`; this branch fired even though audio was perfectly unlocked and the newer clip was already playing.

---

## The fix

### 1. A real, valid, blob-delivered unlock on a reused element

```js
// Build a SHORT, VALID silent WAV (real samples), served as a blob: URL —
// iOS WebKit aborts data: audio; the device-precheck unlocks the same way.
function createSilentWavBlob(): Blob { /* 8kHz, ~0.1s, 8-bit silence, real header */ }

const unlockAudioContext = async (): Promise<boolean> => {
  if (isUnlockedRef.current) return true;
  if (!needsAudioUnlock()) { markUnlocked(true); return true; }

  // Serialize: the interview auto-speaks each question (a non-gesture call that
  // fires a doomed unlock); without this it collides with the user's Enable tap.
  if (unlockInFlightRef.current) return unlockInFlightRef.current;

  const run = (async () => {
    try {
      const el = getAudioElement();                 // ONE persistent element
      if (!silentUrlRef.current)
        silentUrlRef.current = URL.createObjectURL(createSilentWavBlob());
      el.src = silentUrlRef.current;
      el.muted = false; el.volume = 0;
      await el.play();                               // ← the actual unlock
      el.pause(); el.currentTime = 0; el.volume = 1;
      markUnlocked(true);
      return true;
    } catch (error) {
      markUnlocked(false);
      const name = error instanceof DOMException ? error.name : "";
      if (name !== "NotAllowedError" && name !== "AbortError")
        captureException(error);                     // only the genuinely odd ones
      return false;
    }
  })();
  unlockInFlightRef.current = run;
  try { return await run; } finally { unlockInFlightRef.current = null; }
};
```

Every TTS clip now reuses that unlocked element (set `el.src` + assignment-based `onloadstart/onended/onerror`), never `new Audio()`.

### 2. A ref-backed unlocked flag

React's `setState` doesn't reach closures within the same async tick (the modal click → unlock → flush chain), so the flag lives in `isUnlockedRef` as the synchronous source of truth, with `markUnlocked(v)` keeping the React state in sync for the modal's render. Without this, the flush after a successful unlock read a stale `false` and redundantly re-unlocked.

### 3. Post-unlock interrupts are transient

```js
// Superseded by a newer speakText — drop quietly (no toast, no re-lock).
if (!isStillActive()) return;
…
if (error instanceof DOMException && error.name === "NotAllowedError") {
  if (!isUnlockedRef.current) {            // only meaningful if never unlocked
    pendingTextRef.current = { text, options };
    callbacks?.onPermissionDenied?.();
  }
  // Already unlocked → transient interrupt. Do NOT re-lock / toast / re-open dialog.
  return;
}
```

### 4. Gesture-preserving wiring (interview page)

The reactive audio permission dialog's retry used `setTimeout(() => initializeAudio(), 500)` — which pushes `play()` out of the click gesture. Changed to `await initializeAudio()` directly in the handler, and gated the modal on `isIOS() || isSafari()` so macOS Safari is covered too (it previously got no prompt).

### 5. The modal redesign

The original enable-audio dialog had the hallmarks of machine-generated UI: a thick arbitrary-hex border, a `🔇 → 🔊` icon row, two stacked colored info boxes with a bullet list, and six separate copy blocks. Replaced with a compact card matching the app's design system: one accent icon, a one-line title + description, and a single primary button. The now-unused localized strings were trimmed in both language files.

### Alternatives considered

- **Web Audio `AudioContext.resume()` + a silent buffer source** (what the precheck uses for its tone). More robust in theory, but the TTS plays through an `HTMLAudioElement`, and resuming an `AudioContext` does **not** unlock a *separate* media element — we'd still have to prime the element. Kept the reused-element approach as the lower-risk change; the AudioContext route remains the fallback if WebKit ever resists the element approach again.
- **iOS-only gating** (don't prompt on desktop Safari). Rejected on a product call — cover the whole Apple ecosystem — but it's a one-line change if the desktop modal proves too heavy.

---

## Verification

- **Type-check + lint** clean on every iteration; a localization-structure validator confirmed both language files stayed in parity after trimming the modal copy.
- **On-device staging (iPad, iOS Chrome/CriOS)** — the exact device that produced the `AbortError`: after the blob fix, enabling audio plays the first question; after the interrupt fix, advancing questions switches with no second dialog and no error toast.
- **Error monitoring** no longer receives `AbortError`/`NotAllowedError` from this path (reclassified as expected); only genuinely unexpected decode/fetch failures are captured.
- **Non-Apple regression check:** the unlock predicate is false on Android / desktop Chrome / Firefox → the unlocked flag initializes `true`, no modal renders, and the playback path is byte-for-byte unchanged.

---

## Lessons learned

1. **iOS audio unlock is a *real, resolved `play()` inside the gesture, on the element you'll reuse* — nothing less counts.** Loading a source, a muted play, or a deferred play all silently fail. If you're flipping an "unlocked" boolean without an `await el.play()` having resolved first, it's a lie that will surface far from its cause.
2. **`data:` audio URLs are a trap on iOS WebKit; prefer `blob:`** — and never prime with a zero-length clip; give the decoder real (silent) samples. When something audio-related works in one place but not another on the *same device*, **diff the delivery mechanism** (`blob:` vs `data:`, reused vs fresh element) before anything else.
3. **The User-Agent is a first-class debugging signal for media bugs.** iOS Chrome (`CriOS`), mobile Safari, and desktop Safari behave differently under the same WebKit; the UA in the error trace narrowed the search instantly.
4. **`AbortError` and `NotAllowedError` are different stories — don't collapse them.** A post-unlock `NotAllowedError` from an interrupted clip is *not* permission loss. Treating it as such caused a self-inflicted re-lock and a phantom second dialog.
5. **In a gesture → async → callback chain, read state from a ref, not `useState`.** A flag that must be synchronously truthful across click→unlock→flush hops lags a render behind if read from React state.
6. **Ship the unknown-environment fix to a real device early, and let each layer surface the next.** This bug couldn't be fully reasoned out at the desk — iOS Chrome's `data:`-abort behavior only appeared on the device. Three small, verifiable changes beat one big speculative rewrite.
7. **Two UIs for one capability is a smell.** A proactive enabler and a reactive permission dialog for the same thing will eventually both fire. Once a proactive unlock exists, suppress the reactive fallback in the happy path.

---

## Timeline

Compressed engineering chronology (interleaved with normal release work):

- **Day 0 — report & analysis.** User reports the interview is mute on iOS/macOS and the enable-audio button errors; also that the modal looks off. Read the full audio path; identify the fake unlock (no `play()`), per-clip `new Audio()`, and the `setTimeout`-deferred playback as the suspects. Redesign the modal to match the design system.
- **Day 0 — fix 1 to staging.** Real `play()` inside the gesture on a reused element; broaden gating to all Apple browsers; route unexpected failures to monitoring.
- **Day 0 — fix 1 fails on-device.** `AbortError` on iOS Chrome with the precheck-works contrast. Diff against the precheck → `data:` vs `blob:` and the zero-length clip.
- **Day 0 — fix 2 to staging.** Valid silent WAV as a `blob:` URL; serialize unlock attempts; ref-backed unlocked flag. Audio plays.
- **Day 0 — residual double dialog.** Advancing questions throws `NotAllowedError`; the catch re-locks and re-prompts.
- **Day 0 — fix 3.** Drop superseded requests silently; treat post-unlock `NotAllowedError` as a transient interrupt. Resolved on-device and confirmed by the reporter.

---

## Appendix — the silent-WAV builder

A short, valid silent clip is the crux of the fix (zero-length / `data:` clips are what failed). Building it in code keeps it dependency-free and lets us hand it to the element as a `blob:` URL:

```ts
// 8 kHz, ~0.1s of 8-bit mono silence. Real samples (0x80 = silence for unsigned
// 8-bit), a proper RIFF/WAVE header, and a non-zero data chunk — so iOS actually
// has something to decode and play() resolves.
function createSilentWavBlob(): Blob {
  const sampleRate = 8000;
  const numSamples = 800;            // 0.1s
  const dataSize = numSamples;       // 8-bit mono → 1 byte/sample
  const buffer = new ArrayBuffer(44 + dataSize);
  const view = new DataView(buffer);
  const writeStr = (off: number, s: string) => {
    for (let i = 0; i < s.length; i++) view.setUint8(off + i, s.charCodeAt(i));
  };
  writeStr(0, "RIFF");
  view.setUint32(4, 36 + dataSize, true);
  writeStr(8, "WAVE");
  writeStr(12, "fmt ");
  view.setUint32(16, 16, true);      // PCM header size
  view.setUint16(20, 1, true);       // format = PCM
  view.setUint16(22, 1, true);       // channels = mono
  view.setUint32(24, sampleRate, true);
  view.setUint32(28, sampleRate, true); // byte rate (1 byte/sample)
  view.setUint16(32, 1, true);       // block align
  view.setUint16(34, 8, true);       // bits per sample
  writeStr(36, "data");
  view.setUint32(40, dataSize, true);
  for (let i = 0; i < dataSize; i++) view.setUint8(44 + i, 128); // silence
  return new Blob([buffer], { type: "audio/wav" });
}
```

The element is unlocked once with this clip inside the tap, then reused for every real TTS clip for the life of the session.

---

*Maintained by [Alvalens](https://github.com/Alvalens). PRs and issues welcome if you spot inaccuracies or want to discuss a technique used in a postmortem.*
