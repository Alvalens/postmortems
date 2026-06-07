# Engineering Postmortems

A public collection of detailed engineering postmortems from production bugs I've debugged.

Each postmortem is a self-contained markdown document. The intent is to share both the *how* (mechanism of the bug, exact root cause, fix applied) and the *why* (debugging process, wrong hypotheses tried, lessons learned). They're written for engineers who appreciate the *journey* of solving a hard problem, not just the punchline.

## Why postmortems?

Most engineering knowledge dies the moment a bug gets fixed. The commit message is one line. The Slack thread scrolls away. The author moves on. Six months later, someone hits the same class of bug and starts the investigation from zero.

Writing postmortems is a small habit that converts the cost of debugging into a reusable artifact:

- **For yourself** — you remember the lesson, not just the fix.
- **For your team** — future engineers can search for "the bug where X" and find the prior art.
- **For the broader community** — public postmortems give engineers earlier in their career a window into how senior engineers actually think when a tidy explanation falls apart.

This repo is also a personal portfolio. Each postmortem is written so a hiring manager or interviewer can read it and get a real signal about how I debug, communicate, and reason about systems.

## Conventions used in each postmortem

Every postmortem follows a similar structure:

| Section | Purpose |
|---|---|
| **TL;DR** | The bug, cause, fix in 3 lines. For people who only want the punchline. |
| **How to remember this** | Plain-language mental model + memorable analogies. Re-reading this section is enough to reload the whole bug in your head. |
| **Symptoms** | What users / observers actually saw. The original report. |
| **What made this hard** | Why the bug resisted easy diagnosis — the confounding factors, the misleading vocabulary, etc. |
| **Hypotheses tried (and ruled out)** | The wrong turns, with the specific evidence that killed each one. Useful both for credibility and for showing the debugging discipline. |
| **The systematic debugging process** | How instrumentation actually broke the case open after theory-only investigation stalled. |
| **Root cause** | What was actually wrong, with code references and a mechanism walkthrough. |
| **The fix** | Exact change applied, plus alternatives considered and rejected. |
| **Verification** | How the fix was proven correct. |
| **Lessons learned** | Generalizable takeaways — about the bug, about debugging, about the code itself. |
| **Timeline** | Engineering-day chronology so the size of the investment is visible. |
| **Appendix** | Diagnostic instrumentation code, log excerpts, etc. |

Code references use file paths relative to the project root. Stack versions are listed at the top of each postmortem so the reader knows what era of tooling these patterns apply to.

## Sanitization note

These postmortems describe bugs from real production systems I work on. Company-specific names, internal URLs, customer counts, and similar private details have been genericized. All technical content — the mechanism, the code patterns, the fix — is preserved verbatim. The result is something I'd be comfortable sharing with anyone while still being faithful to the actual engineering work.

## Index

| Date | Title | Stack |
|---|---|---|
| 2026-06-07 | [The "enable audio" button that errored instead of unlocking — iOS/Safari autoplay, three layers deep](./2026-06-07-ios-safari-audio-unlock.md) | Next.js 16 · React 19 · cloud TTS · HTMLAudioElement · iOS WebKit autoplay |
| 2026-05-18 | [TanStack Table input-instability render loop freezing data tables](./2026-05-18-tanstack-table-render-loop.md) | Next.js 16 · React 19 · TanStack Query v5 · TanStack Table v8 |
| 2026-03-15 | [Silent Spring `@Transactional` rollback dropped audit rows from a cron-driven workflow](./2026-03-15-spring-transactional-silent-rollback.md) | Java 17 · Spring Boot 3.5 · Hibernate 6 · multi-datasource · ShedLock |

---

*Maintained by [Alvalens](https://github.com/Alvalens). PRs and issues welcome if you spot inaccuracies or want to discuss a technique used in a postmortem.*
