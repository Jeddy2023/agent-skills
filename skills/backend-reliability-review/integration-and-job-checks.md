# Integration and Job Checks

Detailed checks for Steps 5 and 6 of `backend-reliability-review`. Load this file when the side-effect map contains a background job, an event consumer, a webhook handler or an outbound call to a third-party service.

## Outbound Calls

| Check | Pass condition |
|---|---|
| Timeout | Explicit connect and read timeouts, shorter than the caller's own deadline |
| Retry owner | Exactly one layer retries (app, SDK or queue), and the review names which |
| Retry scope | Only timeouts, connection errors, 5xx and 429 are retried; 4xx is not |
| Retry bound | Maximum attempts and total time are set; backoff with jitter |
| Idempotency | Retried non-idempotent calls send the same provider idempotency key on every attempt |
| Unknown outcome | A timeout leads to a status check or reconciliation, not to "failed" |
| Response parsing | Malformed or unexpected responses are rejected with a clear error, not passed on as `undefined` |
| Secrets | Credentials come from configuration and never appear in logs or error messages |

### Status Mapping

Write the provider's states next to the local states and check that every provider state has a mapping:

```
| Provider status   | Local status | Terminal? | Handled in        |
|-------------------|--------------|-----------|-------------------|
| succeeded         | paid         | yes       | webhook + poll    |
| processing        | pending      | no        | poll              |
| requires_action   | pending      | no        | (!) not handled   |
| canceled          | failed       | yes       | webhook           |
```

An unmapped provider state is a defect if the provider documents it, and an unresolved question if the documentation is not available.

## Webhooks

- The signature is verified against the raw request bytes, with the provider's documented algorithm and a constant-time comparison, before the body is parsed.
- Timestamps in the signature are checked against a tolerance window, to stop replays.
- The handler deduplicates on the provider's event id, stored with a unique constraint, before applying any side effect.
- The handler returns success only after the event is durably recorded; heavy work moves to a job.
- Out-of-order delivery is handled: the handler compares the event's state or timestamp with the current local state instead of applying it blindly.
- Unknown event types are acknowledged and logged, not rejected (rejecting causes endless redelivery).

## Background Jobs and Event Consumers

| Scenario | What to check |
|---|---|
| Duplicate delivery | Running the job twice produces the same result as once (dedupe key, conditional update, or upsert) |
| Crash mid-run | A job that dies after its external call but before recording it does not repeat the effect blindly on retry |
| Poison message | A job that always fails stops after its retry limit and lands in a dead-letter queue that someone monitors |
| Ordering | Jobs that must run in order are keyed or serialized; otherwise they tolerate any order |
| Fan-out | Batch jobs bound their concurrency and page through data instead of loading it all |
| Scheduling | Enqueueing happens in the same transaction as the write that requires it, or through an outbox |
| Visibility | Each run logs its job id, attempt number and outcome |

## Reconciliation

High-impact integrations (payments, payouts, bookings, inventory with an external system) need a scheduled comparison between provider state and local state:

- A job lists provider records changed since its last run and compares them with local records.
- Mismatches are corrected automatically when the provider is the source of truth, or alerted when a human must decide.
- The reconciliation window covers the longest retry and dispute period of the provider.
- The review names the reconciliation job, or reports its absence as a finding or a gap, depending on impact.

## Reporting

Every check above that could not be confirmed from code, tests or documentation is reported as a verification gap or an unresolved question under the [Review Evidence Standard](../../references/review-evidence-standard.md), not as a pass.
