# Review Evidence Standard

The evidence bar every review in this pack meets: `code-review-and-quality`, `backend-reliability-review`, the `code-reviewer`, `security-auditor` and `test-engineer` personas, and `/ship`. It decides what a review may claim, how sure it must be, and how it reports what it could not check. The five-axis content lives in the skills; this file governs the evidence behind it.

## Table of Contents

- [Establish the Baseline](#establish-the-baseline)
- [Review Only](#review-only)
- [Introduced vs Pre-existing](#introduced-vs-pre-existing)
- [Evidence Levels](#evidence-levels)
- [Finding Contract](#finding-contract)
- [Contract Checks](#contract-checks)
- [Severity Calibration](#severity-calibration)
- [Noise Rules](#noise-rules)
- [Verification Report](#verification-report)
- [Readiness Checklist](#readiness-checklist)

## Establish the Baseline

Before judging anything, name exactly what is under review:

1. **Pick the baseline.** A PR diff against its base branch, a branch against the merge-base with the default branch, a commit range, or uncommitted working-tree changes. State it in the report as a commit SHA or an explicit range.
2. **Say so when it is ambiguous.** Staged and unstaged changes mixed together, a branch several merges behind its base, or no clear base: describe the ambiguity and what you assumed before drawing any conclusion about scope.
3. **List the change set.** Added, modified, deleted and renamed files.
4. **Read the full diff and the final state of each changed file.** A diff hunk without its surrounding function hides the bug. Then read what the change touches: callers, middleware, shared utilities, schemas, migrations and tests.
5. **State the intended behavior** from the spec, task or PR description. If none exists, say the review judged the change against its apparent intent.

## Review Only

A review reports; it does not edit. Do not modify, stage, commit or format files while reviewing. Propose each fix as a minimal change in the finding. Apply fixes only when the user asks for that as a separate step.

## Introduced vs Pre-existing

Report problems the change **introduces or materially worsens**. A pre-existing problem belongs in the report only when it directly affects the change (the new code calls the broken helper, or relies on the missing constraint), and it is labelled **Pre-existing**. Do not suggest drive-by refactors or rewrites of working code outside the change.

## Evidence Levels

Every finding sits in exactly one level:

| Level | Meaning | Allowed severity |
|---|---|---|
| **Verified defect** | Traced through the code (or reproduced) with a concrete trigger and impact | Any |
| **Unresolved question** | Plausible, but depends on something the review could not establish (intent, config, runtime data) | None; ask the question |
| **Verification gap** | A check that should happen and did not: no environment, no test, no docs | None; say whether it blocks approval |

A suspicion is never reported as a verified defect. "This might race" is a question until you can name the two interleaved operations and the invariant they break.

## Finding Contract

Every Critical or Required finding states:

- **Location**: `file:line`, plus the function, handler, service or migration
- **Trigger**: the inputs or conditions that cause it
- **Impact**: what goes wrong, concretely (wrong record persisted, data exposed, request fails)
- **Evidence**: the code path, test output or documentation that proves it
- **Minimal fix**: the smallest change that resolves it, with no unrelated improvements

## Contract Checks

Compare routes, methods, status codes, request fields, response shapes and third-party provider fields against an authoritative source: the project's schema or OpenAPI file, its contract tests, or the provider's official documentation (see the `source-driven-development` skill). When no authoritative source is available, mark the behavior **Not verified**. Do not assume it is correct, and do not report it as wrong.

## Severity Calibration

Severity is realistic trigger multiplied by impact. A severe outcome behind an unreachable trigger is not Critical; a modest outcome on every request may be Required. Keep the pack's labels and map other scales onto them:

| Common scale | Pack label | Blocks merge |
|---|---|---|
| Critical: demonstrated path to severe compromise or major data loss | **Critical** | Yes |
| High: material security, correctness or reliability defect | **Required** | Yes |
| Medium: meaningful, narrower impact | **Required** or **Optional** | Say which, and why |
| Low: small, actionable improvement | **Optional** or **Nit** | No |

A missing environment or test result is a verification gap, not a defect. State whether the gap prevents approval.

## Noise Rules

Do not report these as problems:

- Names such as `response`, `result`, `data` or `payload` when their meaning is clear in their local scope
- `unknown` for untrusted input or caught errors, when it is narrowed before use
- Explicit validation at a trust boundary, even when the types already look safe
- An abstraction the project already uses for the same job

Report the concrete maintenance or correctness problem, never a taste preference or a guess about whether a person or a model wrote the code. Include praise only when specific evidence supports it; otherwise omit it.

## Verification Report

For every command or check, report three things:

```
- Command/check: npx tsc --noEmit
  Outcome: passed (0 errors)
  Limitation: type-checks the app only; scripts/ is excluded from tsconfig
```

Discover the project's own commands (build, type check, lint, unit and integration tests, migration validation) instead of assuming them. Static inspection does not prove that an integration or end-to-end flow works; say so when nothing ran it. Never run destructive commands against production or shared data.

## Readiness Checklist

Close a production-readiness review with this checklist. Mark each line **Verified**, **Failed**, **Not verified** or **Not applicable**; mark checks that do not fit the stack Not applicable rather than skipping them.

- Scope stayed controlled
- API contracts and provider fields match authoritative sources
- Authentication and authorization are correct
- Ownership and tenant boundaries are enforced
- Input validation and response exposure are appropriate
- Database constraints and transaction boundaries are correct
- Migrations are safe for the deployment approach
- Concurrency and duplicate execution are handled
- External side effects and retries are reliable
- Errors and logs are useful and do not expose secrets
- Queries and resource usage are appropriate
- Existing flows remain compatible
- Required build and checks pass
- Tests support the conclusions
