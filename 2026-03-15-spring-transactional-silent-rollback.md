# Silent Spring `@Transactional` rollback dropped audit rows from a cron-driven workflow

**Date:** 2026-03-15 (approximate — investigation spanned about a week)
**Project:** A multi-datasource Spring Boot enterprise backend serving a regulated, compliance-audited workload
**Stack:** Java 17 · Spring Boot 3.5 · Hibernate 6 · multi-datasource relational backends · servlet container deployment · ShedLock-coordinated `@Scheduled` cron · Spring's `@Transactional` declarative transaction management
**Severity:** Compliance-impacting (audit records that downstream regulatory review depended on were silently missing from the database, with no error anywhere in the application logs)
**Time to root cause:** ~1.5 working days of investigation across several wrong hypotheses
**Resolution:** Extract audit + must-commit operations into a dedicated helper service annotated with `@Transactional(propagation = Propagation.REQUIRES_NEW)`. Consolidate duplicated transaction patterns across eight workflow services (~300 lines of duplication removed as a side effect).

---

## TL;DR

**The bug:** A scheduled cron job orchestrated a third-party-API workflow that, on every run, was supposed to persist two things — an audit record for compliance, and a status update on a related reference table. In production, audit rows for certain records were silently missing. The cron logged success. No exception was thrown anywhere. The external API call had returned 200. The repository `save()` line executed normally and returned without error. The row never appeared.

**The cause:** The audit save was being executed inside the **caller's** Spring `@Transactional` boundary (default propagation `REQUIRED` silently joins any existing transaction). Downstream — after the audit save but before the workflow returned — a separate **soft-validation** path could mark the transaction for rollback. Spring rolled the entire transaction back, including the audit save. From the cron's perspective the workflow had "handled itself" through a catch path and reported success. From the database's perspective, nothing had ever committed.

**The fix:** Extract the audit save and the related status-update writes into a dedicated helper service annotated with `@Transactional(propagation = Propagation.REQUIRES_NEW)`. Each call to those methods now opens a **fresh, independent transaction** that commits regardless of what the caller's transaction ultimately does. Also added `rowsAffected == 0` warning logs on every UPDATE path so any future "the SQL ran but updated nothing" case would at least be visible. While restructuring, consolidated the duplicated transaction pattern across eight workflow services into one reusable helper — netted ~300 lines of duplication removal as a side effect.

After fix: audit rows present in 100% of cron runs (verified across the next ~30 days of production runs). Logs explicitly call out any zero-row UPDATE the moment it occurs, instead of letting it disappear silently.

---

## How to remember this (plain-language mental model)

Skip this section if you want the technical depth. Re-read this section if you want a quick refresher.

### The bug in one paragraph (no jargon)

In Spring, `@Transactional` defaults to a mode called `REQUIRED`, which means *"if my caller already opened a transaction, I'll just join theirs. If not, I'll open my own."* It sounds reasonable until you write something like an audit log that **must** persist regardless of what the rest of the workflow does. Because if the caller's transaction rolls back later — for **any** reason — your audit write rolls back too, even though your own code never saw an error. You called `save()`. It returned cleanly. JPA staged the INSERT. But the COMMIT never happened, because the outer transaction was marked rollback-only somewhere downstream by a different piece of code, and at the end of the request Spring rolled the whole thing back as a unit. The audit row vanishes silently. The fix is to tell Spring "no, give me my OWN transaction for this — don't join the caller's" using `propagation = REQUIRES_NEW`. Now the audit commits on its own merits and nothing the caller does can undo it.

### The two-condition framing

Both have to be true at the same time for an audit row to silently disappear:

- **(A)** The audit-save method runs inside the caller's transaction (default `REQUIRED` propagation silently joins any existing transaction)
- **(B)** Some other code path in the same transaction marks it for rollback (a thrown-then-caught exception that hits `rollbackOn`, an explicit `setRollbackOnly()`, or a soft-validation handler that decides not to proceed)

Just (A) alone: outer transaction commits successfully → audit row persists → no bug.
Just (B) alone: there's no outer transaction to join, so the audit method opens its own, commits or rolls back on its own merits → no bug.
**(A) + (B):** audit method "succeeded" from the JPA call's perspective, but the COMMIT is owned by the caller, who decided to roll back the whole transaction → audit row silently vanishes.

The fix kills (A) at its source. With `REQUIRES_NEW`, the audit method **always** opens its own transaction regardless of what the caller did, so the caller's rollback can never reach it.

### Why "the cron said it succeeded but the row isn't there" was the right place to start

When the *system* tells you "nothing went wrong" but the *data* tells you something is missing, **trust the data**. Frameworks have defaults. Defaults are usually right. But "usually" is not "always," and the audit-and-outbox case is one of the classic counterexamples that every backend engineer eventually has to learn.

The mental model: writes don't actually exist in the database until COMMIT. Everything between `save()` and COMMIT is staged. If anything reaches in and changes the COMMIT decision after your `save()` returned, your write retroactively never happened — but your code never finds out, because the failure is in the framework's transaction manager, not in your method's call stack.

### The mechanism in one sentence each

- **Bug:** caller opens `@Transactional` → audit-service `save()` joins caller's transaction (default `REQUIRED`) → caller's later code triggers rollback → Spring rolls back the entire transaction including the audit save → audit row never committed, but no exception ever surfaces in audit-service's call stack.
- **Fix:** audit-service method annotated `@Transactional(propagation = REQUIRES_NEW)` → forces a new, independent transaction every time it's called → audit commits or fails on its own merits, completely decoupled from whatever the caller does after.

### Mental model: writes-in-flight live in the caller's transaction by default

Spring's transaction propagation is essentially deciding the answer to one question per `@Transactional` boundary: *"is my work eligible to be undone if my caller's work fails?"* For business logic, the answer is usually yes (you want all-or-nothing semantics). For audit, outbox, error notifications, scheduled-job heartbeats, and "must-persist-regardless" telemetry, the answer is no. Default `REQUIRED` says yes. `REQUIRES_NEW` says no.

The footgun is that `REQUIRED` is the default and it works perfectly until the day a downstream rollback marker introduces silent data loss. There's no warning. There's no compiler check. There's no runtime exception. The data is just gone.

---

## Symptoms (as originally reported)

A QA / ops report flagged that for a subset of records processed by a particular nightly cron, the corresponding audit rows were missing from the database. The application had to produce a compliance audit trail, so missing rows were a real problem — not just a cosmetic bug.

What made it confusing:

- The cron logged completion: **`status=success`**, no anomalies.
- The external API the workflow integrated with had returned **HTTP 200** for every record.
- No exception had been thrown or logged anywhere in the workflow.
- Per-step debug logging showed the audit `save()` line being executed.
- The status update on the related reference table was also missing in the same cases, but again — no error.
- Re-running the cron manually against affected records sometimes "fixed" them, sometimes didn't (suggesting state-dependent behavior rather than a deterministic code path).

Repro in dev was possible only by deliberately triggering certain combinations of state that exercised a specific downstream branch. That should have been a hint — but it took me a while to read it correctly.

## What made this hard

- **The framework lied — or rather, was being silent in the way it was designed to be.** Spring's `@Transactional` is declarative. When you join a caller's transaction and the caller later rolls back, your method has no idea. Your method's call stack is long gone by then. The transaction manager just doesn't COMMIT.
- **The bug was in the absence of an action, not the presence of one.** Most bugs leave a trace — a stack trace, an exception, a wrong value somewhere. This bug's signature was a SQL statement that ran but didn't commit. Logs can't catch what doesn't happen.
- **The wrong layer of investigation.** Every hypothesis I tried for the first half-day was at the application layer — "is the conditional wrong?", "is the save being skipped?", "is the API contract returning weird data?". The actual fault was one layer up, in the transaction boundary.
- **Spring's defaults make this an easy trap to fall into.** `@Transactional` without arguments doesn't *feel* dangerous. There's no special syntax. Most tutorials use the default propagation. You have to specifically know that audit-style writes need different semantics.

---

## Hypotheses tried and ruled out

### Hypothesis 1: A logical bug in *when* the audit was being written

**Theory:** Maybe the audit save was inside a branch that wasn't being entered for these records. Either the conditional was wrong, or the data was steering execution into a path that skipped the write.

**Action taken:** Traced the flow with per-step logging. Confirmed via logs that the audit `save()` line was reached and called for the affected records.

**What killed it:** Logs showed the save line executing without error. So this wasn't a "code didn't run" bug. It was a "code ran but produced no effect" bug.

**Lesson:** When the symptom is missing data, "did the code run?" is the first question, but "did the database actually commit?" is a separate question. The two are not the same. JPA `save()` doesn't COMMIT — it stages an INSERT into the persistence context that gets flushed and committed later when the transaction completes.

### Hypothesis 2: The external API was returning a 200 but with a payload signaling rejection

**Theory:** What if the third-party-API was returning HTTP 200 with a business-level "reject" indicator in the body, and our code was treating that as success when it should have been treating it as a no-op?

**Action taken:** Inspected the raw response payloads for the affected records via the integration logs.

**What killed it:** The response payloads were normal-success shapes. The downstream code was correctly recognizing them as successful. The audit save was supposed to fire for both success and failure branches anyway, with different content — so even if the response had been a soft-failure, the audit row should still exist (just with different fields). It wasn't.

**Lesson:** Don't optimize for the wrong fork in the road. The integration responses were a plausible enough place to look, but the symptom (missing row, not wrong row) didn't actually fit the integration-payload hypothesis. I should have noticed that earlier and moved on faster.

### Hypothesis 3: Repository / Hibernate flush-mode issue

**Theory:** Maybe Hibernate was deferring the actual SQL execution past the point where I expected, and something downstream was clearing the persistence context (e.g., `EntityManager.clear()` or a connection reset).

**Action taken:** Enabled SQL logging at `org.hibernate.SQL=DEBUG` to confirm INSERT statements were actually being emitted to the database connection. Checked for any explicit `EntityManager.clear()` or `flush()` calls in the workflow.

**What killed it:** The SQL log showed the INSERT statements being prepared and dispatched on the connection. No `clear()` calls in the relevant code path. So the writes WERE going out to the database — just not being committed.

**This was the lead I needed but didn't fully recognize yet.** "Going out but not committing" should have immediately pointed me to the transaction-boundary level. Instead I spent another couple of hours convincing myself this wasn't some quirk of multi-datasource Hibernate config.

---

## The systematic debugging process — when the wrong-layer theories collapsed

The breakthrough was deliberately backing out and asking a different question: **"if the INSERT was sent to the connection but never committed, what entity owns the COMMIT decision?"**

Answer: the transaction manager, which is governed by Spring's `@Transactional` annotation chain. The audit-service method had `@Transactional` on it (default `REQUIRED`). The caller method had `@Transactional` on it too. Per Spring's propagation rules, my audit-service method was joining the caller's transaction, not opening its own.

That immediately raised the question: *under what conditions would the caller's transaction get rolled back without the audit-service method seeing an exception?*

I went looking for places downstream of the audit save where the workflow could decide "don't proceed." There were several — soft-validation handlers, business-rule guard clauses that caught exceptions internally and turned them into "decline" outcomes, error-notification paths that themselves threw exceptions which were caught by the outer workflow but not before marking the transaction.

The smoking gun: a downstream guard clause that caught a particular validation failure and *internally handled it* — by intentionally throwing a checked exception that the outermost service method caught and translated into a "decline" return value. The Spring transaction manager, however, saw an exception propagate up the stack to the `@Transactional` boundary and marked the transaction for rollback per its default `rollbackOn` rules. The catch was too far up. The damage was already done at the transaction-manager level by the time the exception was caught at the application level.

From the cron's perspective, the workflow returned cleanly with a "decline" outcome. From the application's perspective, no exception escaped. From the database's perspective, every write inside the outer transaction — including my audit save — was rolled back at the end of the boundary.

That was the bug.

---

## Root cause

Spring's default `@Transactional` propagation mode (`REQUIRED`) silently joins any caller's transaction. When the caller's transaction is rolled back — for any reason, including a downstream catch-and-translate path that re-raises an exception across a transactional boundary before catching it again — every write in the same transaction is rolled back, including writes from inner `@Transactional` methods.

The audit-save method was correctly written from a "save the audit row" perspective. The repository call was correctly made. JPA correctly staged the INSERT. The database connection correctly received the INSERT statement. The only thing that didn't happen was the COMMIT — and the COMMIT decision was owned by the outer caller's transaction boundary, not the audit-save method.

This is a real but classic Spring footgun. The official docs document it. Most engineers find it the hard way once, then never again.

### The mechanism step by step

1. Cron fires. ShedLock acquires the lock for this run.
2. Cron handler iterates through eligible records, calls the workflow service for each one.
3. Workflow service method is annotated `@Transactional`. Spring opens a new transaction (or joins an outer one if one exists; in this case, a new one).
4. Workflow does its work: calls the third-party API, gets 200, computes the next state, calls `auditService.saveAudit(...)`.
5. Audit-service method is also annotated `@Transactional`. Default propagation = `REQUIRED`. Spring sees an active transaction → audit-service joins it. No new transaction is opened.
6. Audit-service calls `auditRepository.save(audit)`. JPA stages the INSERT inside the current (caller's) transaction's persistence context.
7. Audit-service returns. The audit row is now "saved" — visible to the same transaction, but not yet committed.
8. Workflow continues. Status-update operations run. They also stage UPDATE statements inside the same transaction.
9. Downstream: a soft-validation handler decides the workflow should not proceed for this record. It signals this by throwing a checked exception.
10. The outer workflow service catches the exception and translates it into a `WorkflowOutcome.DECLINED` return value. No exception escapes the workflow service method.
11. **BUT** — Spring's transaction manager, watching the `@Transactional` boundary, observed that an exception propagated up the stack past the boundary line before being caught. Per default `rollbackOn = RuntimeException.class | Error.class` (and custom rollback config in some places), the transaction is now marked rollback-only.
12. When the workflow service method returns and the transaction commit attempt occurs, Spring sees the rollback-only flag and rolls back instead. The audit INSERT and all UPDATEs are reverted.
13. Cron handler sees the workflow service returned `WorkflowOutcome.DECLINED`. Logs success ("workflow handled the record"). Moves on to the next record.
14. From outside: cron succeeded, no exceptions, API returned 200. From the database: audit row never existed.

The bug is the implicit transaction-boundary semantics of step 5 combined with the cross-boundary exception-propagation pattern in steps 9-11. Either could exist alone safely. Together they produce silent data loss.

---

## The fix

### Step 1: extract audit + must-commit writes into a dedicated helper service

Created an audit-and-reference-update helper service with explicit `REQUIRES_NEW` propagation on every method that must persist independently of the caller:

```java
@Service
public class WorkflowAuditService {

    private final AuditRepository auditRepository;
    private final CasesReferenceRepository casesReferenceRepository;

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveAudit(WorkflowRequest request, String reasonCode, String message, ...) {
        Audit audit = new Audit();
        // populate fields from request
        auditRepository.save(audit);
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void updateRelatedReference(String caseNo, String noticeNo, String decisionCode) {
        if (decisionCode != null && !decisionCode.isEmpty()
                && referenceCodeRepository.existsById(decisionCode)) {
            int rowsAffected = casesReferenceRepository.updateDecisionCode(
                caseNo, noticeNo, decisionCode, LocalDateTime.now());
            if (rowsAffected == 0) {
                log.warn("updateDecisionCode affected 0 rows for caseNo={} noticeNo={} — "
                       + "either the row doesn't exist or filter conditions excluded it",
                       caseNo, noticeNo);
            }
        }
    }
}
```

The key annotation is `@Transactional(propagation = Propagation.REQUIRES_NEW)`. This tells Spring: *"every time this method is called, open a fresh, independent transaction. Suspend any caller's transaction if one exists. Commit or roll back on this method's own merits, completely decoupled from the caller."*

Now when the workflow service calls `auditService.saveAudit(...)`:
- Spring suspends the caller's transaction.
- Spring opens a new, independent transaction for the audit-save method.
- Audit-save method runs its INSERT.
- Audit-save method returns normally.
- Spring commits the audit transaction (or rolls it back if the audit-save method itself throws).
- Spring resumes the caller's transaction and execution continues.

If the caller's transaction later gets rolled back, the audit transaction has **already committed**. The rollback cannot reach back across the transaction boundary. The audit row persists.

### Step 2: add `rowsAffected == 0` warning logs on UPDATE paths

While I was in there, I added explicit row-count check logs to every UPDATE that returns a row count. The pattern across the codebase was to call a repository UPDATE and not check whether it actually updated anything. If a filter clause silently excluded the target row (e.g., a status changed between read and update), the UPDATE would "succeed" with 0 rows affected and no one would know.

Now any UPDATE path that returns 0 rows emits a warning log immediately, with enough identifying fields to find the affected record. This is defense-in-depth: even if some future bug introduces a similar silent-loss pattern, it'll show up in the logs the moment it occurs.

### Step 3: consolidate duplicated transaction patterns across eight workflow services

While extracting the audit pattern, I noticed eight different workflow services (different feature areas) were each duplicating the same audit-save + status-update sequence with slightly different wrapping. Now that I had one helper service with the correct transaction semantics, I migrated each of the eight services to use it. ~300 lines of duplication removed. Every place that touches audit now goes through the same transaction boundary with the same semantics — no more "did this particular caller remember to do it right?" question.

### What was deliberately NOT changed

- **The workflow services' own `@Transactional` boundaries.** Their business logic still needs all-or-nothing semantics for the *business* writes. Default `REQUIRED` is correct for those. Only the audit-and-status writes were extracted.
- **The soft-validation pattern** (catch a downstream exception, translate to a workflow outcome). That's a legitimate pattern for clean workflow APIs. The fix wasn't to remove it; it was to make audit writes immune to it.
- **The `rollbackOn` configuration.** The defaults are reasonable. We just needed to opt some writes out of the rollback scope.

---

## Verification

Verification took roughly two weeks of production observation:

1. **Unit / integration tests.** Added an integration test that exercises the exact failure scenario: workflow service is called, audit-service is called (and commits its `REQUIRES_NEW` transaction), then the workflow throws an exception that triggers caller-transaction rollback, then the workflow's outer catch translates the exception into a non-error outcome. After the workflow returns, the test asserts that the audit row IS still present in the database. Pre-fix, this test failed. Post-fix, this test passed.

2. **Production check.** Inspected the audit table over the next ~30 days of cron runs. Every record that the cron handler reported as processed now had its corresponding audit row. The `rowsAffected == 0` warning logs fired a handful of times for genuinely-no-matching-row cases (which is the right behavior — the warning was the entire point).

3. **No regressions.** Other workflow paths that relied on the all-or-nothing transactional semantics for *business* writes were unaffected, because their own `@Transactional` boundaries weren't touched.

---

## Lessons learned

### About the bug itself

1. **`@Transactional` defaults to `REQUIRED`, which is a hazard for audit/outbox writes.** Required propagation silently joins any caller's transaction, which means an audit write you intended to be "fire-and-forget for compliance" actually has its commit decision owned by whoever invoked you. If the caller later rolls back, your audit row vanishes silently. This is a known footgun documented in Spring's reference docs — but it's the kind of thing you only really internalize after you've been bitten.

2. **`REQUIRES_NEW` is the right propagation for "must-commit-regardless" writes.** Audit logs, outbox patterns, error notifications, scheduled-job heartbeats, anything that needs to survive a caller's failure. The cost is a separate database connection from the connection pool for the inner transaction (it briefly suspends the caller's connection), so don't use it gratuitously — but for legitimate must-persist-regardless writes, it's the correct choice.

3. **The silent failures are the dangerous ones.** When the system tells you "nothing went wrong" but the data tells you something is missing, the failure is almost certainly in a layer that doesn't surface errors via the call stack: transactions, async event handlers, persistence-context lifecycle, connection-pool acquisition, message broker acks. Code-level debugging won't catch it because no code-level exception was thrown.

4. **`save()` is not the same as COMMIT.** JPA's `save()` stages an INSERT into the persistence context. The COMMIT happens at the transaction boundary, possibly far above the `save()` call in the call stack. The two are independent. Watching `save()` execute without error tells you nothing about whether the row will actually be persisted.

5. **`rowsAffected == 0` is information.** Most engineers (myself included, before this incident) write UPDATE code that calls the repository method and moves on. The row count is silently discarded. Adding a warning log on `rowsAffected == 0` is one of the cheapest forms of defense-in-depth there is, and it would have caught this bug class earlier in any number of nearby code paths.

### About the debugging process

1. **Trust the data when the system says "nothing went wrong."** The cron's success log was correct — the cron handler did succeed at its job per its own definition. The missing data was real. The conflict between those two truths is a signal that the failure is in a layer you haven't looked at yet.

2. **When debugging silent failures, "did the COMMIT happen?" is a separate question from "did the code run?"** I burned a half-day on the first question before realizing I needed to ask the second. With Hibernate SQL logging enabled, the answer was right there: INSERT statements were being dispatched but never committed. That's the signature of a transaction-level failure, not a code-level one.

3. **Wrong-layer hypotheses can eat hours.** My first three hypotheses were all at the application layer (conditional logic, API payload semantics, Hibernate persistence-context quirks). The actual fault was one layer up at the Spring transaction manager. The mental model fix was to ask: *who owns the COMMIT decision for this code path?* Once I asked that, the bug became obvious.

4. **Read the transaction graph, not just the call graph.** For any `@Transactional` boundary, ask: which caller opened the transaction? What other code paths are in the same transaction? What could mark this transaction for rollback before it commits? In retrospect this is a debugging discipline I now apply by default in any Spring multi-service workflow.

### About the code itself

1. **Cron-driven workflows are particularly vulnerable to this class of bug.** Crons run in background threads without a user-facing error surface. If audit writes silently fail in a request-driven flow, a user might notice "I clicked save but nothing happened." If they silently fail in a cron, no human will notice until the missing data is needed weeks later — usually during an audit or a regulatory review. That's the worst possible time.

2. **Duplicated transaction patterns are a smell that this bug is waiting to happen somewhere.** I found the audit+status pattern duplicated across eight workflow services. Each instance had subtly different transaction-boundary conventions. The fix was to consolidate them into one helper with the correct semantics enforced in one place. Every duplicated transaction pattern in a Spring codebase is an opportunity for one of them to silently use the wrong propagation.

3. **`@Transactional` placement is a contract that should be reviewable.** The team has adopted a code-review checklist item for any new `@Transactional` annotation: *"is the default `REQUIRED` propagation correct here, or should this be `REQUIRES_NEW`, `NESTED`, or `NOT_SUPPORTED`?"* Most of the time `REQUIRED` is right. Forcing the reviewer to think about it explicitly catches the cases where it isn't.

### About working in a regulated environment

1. **Compliance-impacting bugs deserve more than a code fix.** The fix isn't done when the code change ships. It's done when you've verified production no longer exhibits the failure for at least one observation window, AND when you've added telemetry that would catch a recurrence of the same bug class even from different code paths. The `rowsAffected == 0` warning logs were as important as the propagation fix.

2. **Working under regulatory audit pressure shifts which bugs feel urgent.** A missing audit row in a non-regulated system is annoying. A missing audit row when downstream regulatory or compliance review depends on it is a real problem. The investigation here was higher-priority than a typical bug because the consequences of leaving it unfixed were bigger than "users see a glitch."

---

## Timeline (engineering-day chronology, approximate)

- **Day 1, morning:** QA / ops report: missing audit rows for certain records in nightly cron output. Initial assumption: logical bug in the workflow's audit-write branch.
- **Day 1, midday:** Added per-step debug logging to the workflow. Confirmed audit `save()` line was being executed for affected records. Hypothesis 1 ruled out.
- **Day 1, afternoon:** Investigated whether the external API was returning a soft-failure inside an HTTP 200 envelope. Confirmed responses were normal-success. Hypothesis 2 ruled out.
- **Day 1, late afternoon:** Enabled Hibernate SQL logging. Confirmed INSERT statements were being dispatched to the database connection but the corresponding rows weren't appearing in the table. Hypothesis 3 inspected but not yet correctly understood.
- **Day 2, morning:** Stepped back and asked the right question: *if the INSERT was sent but not committed, who owns the COMMIT?* That led directly to the transaction-boundary level. Traced the propagation chain from the cron handler → workflow service → audit service. All `@Transactional` with default `REQUIRED`. Started looking downstream for code that could trigger rollback.
- **Day 2, midday:** Found the soft-validation handler that threw a checked exception which the outer service caught and translated to a workflow outcome. Identified that the exception had already crossed the inner `@Transactional` boundary before being caught — marking the transaction rollback-only. **Root cause confirmed.**
- **Day 2, afternoon:** Designed the fix. Extracted audit-save and related status-update operations into a dedicated helper service annotated `@Transactional(propagation = REQUIRES_NEW)`. Wrote the failing-then-passing integration test. Senior review approved.
- **Day 3, morning:** Discovered the same audit + status update pattern duplicated across eight workflow services. Consolidated all of them onto the new helper. Added `rowsAffected == 0` warning logs on every UPDATE path while restructuring. Net ~300 lines of duplication removed.
- **Day 3, afternoon:** Tested in staging. Audit rows now persist correctly in the same scenario that previously lost them. Shipped to production with the next release.
- **Following 2 weeks:** Production verification. All cron runs produced complete audit rows. No regressions. Closed the incident.

Total: **~3 working days from report to fix shipped**, **~2 weeks** to full production verification.

---

## Code references and patterns

**Pattern that emerged from this incident** (now a team convention):

```java
@Service
public class XxxAuditService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveAudit(/* ... */) {
        // INSERT audit row — commits independently of caller's transaction
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void updateXxxReference(/* ... */) {
        int rowsAffected = repository.updateXxx(/* ... */);
        if (rowsAffected == 0) {
            log.warn("updateXxx affected 0 rows for ...");
        }
    }
}
```

**Code-review checklist additions** (now applied team-wide for any PR that adds or modifies `@Transactional`):

- Is default `REQUIRED` propagation correct for this method, or does it need `REQUIRES_NEW`?
- Does this method belong to the "audit / outbox / must-persist-regardless" category? If yes, `REQUIRES_NEW`.
- Are there any UPDATE paths returning row counts that are being silently discarded? Add `rowsAffected == 0` warning logs.
- For any new cron / scheduled-job workflow: explicitly trace the transaction graph end-to-end before merging.

**Stack versions at the time:**

```
Java 17
Spring Boot 3.5.x
Hibernate 6 (via Spring Boot 3.5)
Multiple relational datasources with HikariCP per datasource
ShedLock for distributed cron coordination
Servlet-container deployment of the production WAR
```

---

*Postmortem authored by [Alvalens](https://github.com/Alvalens), based on a real production incident investigation. Employer name, project name, geography, class names, and internal identifiers have all been removed or genericized; the technical mechanism, fix pattern, and lessons learned are faithful to the actual engineering work.*
