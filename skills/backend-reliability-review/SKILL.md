---
name: backend-reliability-review
description: Reviews a change for data integrity and reliability under failure — transactions, concurrency, duplicate execution, background work, external integrations and schema migrations. Use when a change being reviewed or shipped writes to a database, adds or alters a migration, enqueues jobs or events, handles webhooks, or calls a third-party service. Use when asked whether a backend change is safe to merge or to operate in production.
---

# Backend Reliability Review

## Overview

Most backend defects that reach production are not wrong on the happy path. They break when a request is retried, when two requests overlap, when the second of three writes fails, when a provider times out, or when a migration meets real rows. A five-axis review reads the code as it is written; this review traces what the change *does to the world* and asks what happens when each step fails or runs twice.

It complements `code-review-and-quality` rather than replacing it, and it reports to the same [Review Evidence Standard](../../references/review-evidence-standard.md): stated baseline, review only, verified defects kept apart from questions and gaps.

## When to Use

- Reviewing or shipping a change that writes to a database, or changes a schema or migration
- The change enqueues a job, publishes an event, handles a webhook, or calls a third-party API
- Payments, bookings, orders, inventory, credits, or any record where a duplicate or a lost write costs money or trust
- Asked "is this backend change safe to merge?" or "can this run in production?"

**When NOT to use:**
- Designing a new API contract (use the `api-and-interface-design` skill)
- Planning a migration or deprecation before it is written (use the `deprecation-and-migration` skill)
- A change with no persistence, jobs or outbound calls — a pure UI or formatting change (use the `code-review-and-quality` skill alone)

## The Process

### Step 1: Map the Side Effects

Before judging anything, list every effect the change adds or alters, in execution order. Everything after this step works from this table.

```
| # | Operation                      | Durable? | Inside txn? | Retried by        | If it fails here...            |
|---|--------------------------------|----------|-------------|-------------------|--------------------------------|
| 1 | INSERT order                   | yes      | T1          | client retry      | nothing persisted — OK         |
| 2 | POST provider /charges         | external | T1 (!)      | SDK, 2 attempts   | charge may exist, txn rolls back |
| 3 | UPDATE order status = paid     | yes      | T1          | —                 | charged but order unpaid       |
| 4 | enqueue send-receipt job       | no (!)   | after T1    | queue, 5 attempts | order paid, no receipt, silent |
```

Find each row in the code, not in the PR description. Flag every `(!)`: those are where reviews find defects.

### Step 2: Correctness of the Persisted Result

- The happy path produces the intended rows, with the intended values. Trace it end to end once.
- Business invariants that must never break (one active subscription per user, balance ≥ 0) are enforced by a **database constraint**, not only by application code. Code checks race; constraints don't.
- Missing records, invalid state transitions, duplicates and boundary values return defined errors, not 500s or silent no-ops.
- Dates, time zones, currency and precision follow the project's rules (money in integer minor units or decimal, never float; stored timestamps in UTC).
- PATCH handlers distinguish omitted, `null` and empty values (see the PATCH section of the `api-and-interface-design` skill).
- Client-controlled fields cannot set protected values: owner, role, price, status, tenant.

### Step 3: Transactions and Partial Failure

- **Related writes are atomic.** If rows 1 and 3 must agree, they share one transaction, or the design explains how they converge.
- **No network call inside a transaction.** An HTTP call held in a transaction keeps locks and a connection for the provider's latency, and its effect cannot roll back. Call out before or after; record the intent first.
- **A commit must not lose its follow-up.** If a write commits and the event or job that should follow it is published separately, a crash in between loses it silently. Use a durable handoff: an outbox row written in the same transaction and relayed afterwards, or a job enqueued transactionally.
- **Report success only when the work is done or durably scheduled.** A `200` returned before the job is persisted is a lie under a crash.
- **Errors propagate.** A `catch` that logs and continues turns a failed write into a "successful" request. Each catch either handles the failure meaningfully or rethrows.

### Step 4: Concurrency and Duplicate Execution

Name the two interleaved operations and the invariant they break; otherwise it is an unresolved question, not a finding.

- **Check-then-insert** (`SELECT` none → `INSERT`) races under concurrent requests. Fix with a unique constraint and handling of its violation.
- **Read-modify-write** (load balance, add, save) loses updates. Fix with an atomic update (`SET balance = balance - :amount WHERE balance >= :amount`), a version column, or a row lock.
- **State transitions** (`pending → paid`) are guarded atomically: `UPDATE … WHERE id = :id AND status = 'pending'`, then check the affected row count.
- **Repeated requests** that create consequential records need an idempotency key, honoured as in the `api-and-interface-design` skill — do not re-derive that design here.

### Step 5: Retries, Jobs and Webhooks

- Retries are bounded, with backoff and jitter, and only for failures that can succeed on retry (timeouts, 5xx, 429), never for 4xx validation errors.
- Every job and webhook consumer is safe to run twice: delivery is at-least-once in every queue and every provider.
- Webhook signatures are verified against the **raw** request body, before parsing.
- Event ordering is handled where correctness depends on it (a `refunded` arriving before `paid`).
- High-impact flows reconcile local state with the provider's state on a schedule, not only on the happy-path callback.

The detailed checks for jobs and providers are in [integration-and-job-checks.md](integration-and-job-checks.md); load it when the side-effect map has a job, a webhook or an outbound call.

### Step 6: External Integrations

- Every outbound call has an explicit timeout. A missing timeout is a thread or connection held forever when the provider hangs.
- Unavailable providers, malformed responses and unexpected status codes are handled, and a timeout is treated as **unknown outcome**, not failure.
- Endpoints, fields, signatures and status mappings match the provider's official documentation (see the `source-driven-development` skill). If the docs are not available, mark the integration **Not verified**.

### Step 7: Migrations

Run the "Pre-flight against existing data" checks in the `deprecation-and-migration` skill: constraints against existing rows, table rewrites and locks, old and new app versions overlapping during deploy, and the recovery plan when there is no safe down path.

### Step 8: Operability

- Unexpected failures are logged once, at the layer that handles them, with the request, job or correlation id and without secrets or personal data (see the `observability-and-instrumentation` skill).
- Operators can tell which request, job or integration failed from the logs alone.
- Health and readiness checks cover any new hard dependency, and do not fail on optional ones.

### Step 9: Report

Report findings with the [finding contract](../../references/review-evidence-standard.md#finding-contract) and close with the [readiness checklist](../../references/review-evidence-standard.md#readiness-checklist). Include the side-effect map from Step 1: it is the evidence for every reliability finding.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "The queue guarantees delivery" | No queue gives exactly-once across a consumer crash. Design for at-least-once and make consumers idempotent. |
| "It's wrapped in a try/catch" | A catch that logs and continues converts a failure into a false success. Ask what state the system is in after the catch. |
| "The migration ran fine locally" | Local databases have no duplicate rows, no nulls and no traffic. Production has all three. |
| "The provider SDK handles retries" | Then two layers may retry the same non-idempotent call. Know who retries, how often, and with what key. |
| "Two requests at the same instant is unlikely" | Double-clicks, client retries and load balancer replays make it routine. Unlikely per request is certain per month. |
| "We can reconcile manually if it happens" | Only if someone can detect it. Without a reconciliation job or an alert, nobody knows it happened. |
| "The transaction makes it safe" | A transaction does not roll back an HTTP call or an email, and holding it open across one invites lock contention. |

## Red Flags

- An HTTP, email or queue call between `BEGIN` and `COMMIT`
- `SELECT` followed by `INSERT` to prevent duplicates, with no unique constraint behind it
- A balance, counter or status loaded, modified in code, and saved back
- Success returned before a job or event is durably recorded
- A job or webhook handler with an external side effect and no deduplication
- An outbound call with no timeout, or a timeout handled as a definite failure
- A webhook parsed as JSON before its signature is checked
- A `NOT NULL` or unique constraint added to a populated table with no data check
- An empty `catch`, or a `catch` that only logs, around a write

## Verification

- [ ] The side-effect map is in the report and every row was located in the code
- [ ] Every multi-write path is atomic or its convergence is explained
- [ ] No network call is held inside a transaction, or the finding is reported
- [ ] Every consequential create or state change is safe under concurrent and repeated execution, or the finding names the interleaving
- [ ] Every job, event and webhook consumer is idempotent, and its retries are bounded
- [ ] Every outbound call has a timeout, and its unknown outcome is handled
- [ ] Provider contracts were checked against official docs, or marked Not verified
- [ ] Migrations passed the existing-data pre-flight, or the gaps are reported
- [ ] Findings follow the evidence levels; suspicions are listed as unresolved questions
- [ ] No files were modified during the review
