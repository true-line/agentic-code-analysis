# ACA-0095: One deadline and one publication boundary per check

**Status:** Proposed
**Date:** 2026-09-22
**Issue:** #95; downstream qwts/overlook#1176
**Extends:** [ACA-0003](ACA-0003-suite-contracts.md) D3 (exit codes) and
D7 (verdict cache); preserves the frozen D2 JudgeClient interface

## Context

Overlook bounds ACA process groups at 120 seconds, but several checks still
finish without reports. The evidence and exact downstream jobs are recorded in
[#95](https://github.com/qwts/agentic-code-analysis/issues/95). At source
`1eb89a1`, the Qwen adapter supplies neither an explicit request timeout nor
retry policy to its OpenAI-compatible SDK. This is a missing control and
observation boundary, not proof that retries caused the delays.

A timeout per request restarts the allowance for every request. A timer in the
same JavaScript thread cannot interrupt synchronous preparation. The current
cache also writes individual verdicts immediately, before the check finishes.
A total deadline therefore needs a supervised execution and publication
contract, rather than only an SDK option.

## Proposed decision

### One supervised budget

A normal CLI check has a default total budget of 120,000 milliseconds, starting
before check loading, configuration, scope selection, or preparation. A positive
`--timeout-ms` may request a smaller budget, never a larger one. Help, version,
and argument validation do not start a check. Calibration self-tests use the
same limit and report timeout as missing qualification evidence.

The CLI supervisor owns a monotonic deadline outside the execution worker's
JavaScript event loop. Preparation, judge calls, retry waits, result validation,
and publication all consume the same remaining allowance. It stops admitting
work when that allowance is exhausted. Cooperative request cancellation starts
before the deadline, reserving a bounded cleanup interval inside the total
budget; unresponsive execution and its descendants are terminated at the
boundary. No cleanup or retry receives a fresh allowance. Tests must cover the
supported operating systems, not assume POSIX process groups work on Windows.

The downstream process-group guard remains defense in depth. Measured elapsed
time includes scheduler tolerance; no mechanism promises real-time scheduling.

### Preserve the judge port and qualified routes

Checks keep calling the frozen `JudgeClient.judge(JudgeRequest)` interface.
The CLI creates an execution context once and supplies it when constructing
adapters; deadlines and cancellation do not become prompt fields or check-owned
inference knobs. Standalone adapter consumers retain their existing interface;
the supervised CLI is the boundary that guarantees the total check limit.

Adapters pass the remaining budget and cancellation signal to the underlying
transport. Disable hidden SDK retries explicitly. Permit at most two explicit
retries of retryable transport failures, with bounded backoff, within the same
remaining budget. Do not retry account rejection or schema/refusal outcomes.
Stop retry/backoff immediately on cancellation. A noncooperative transport must
not keep the supervisor alive after the deadline.

Provider, model, prompt, token allowance, and cache semantic keys stay unchanged.
Changing one of these requires separate route qualification; a shorter answer
or cheaper route is not an implicit timeout remedy.

### Timeout is absent evidence

An expired check returns exit **124** in advisory, enforce, and self-test modes.
It prints a redacted timeout receipt to stderr and emits no verdict JSON or
partial report to stdout. This explicitly extends the advisory-default exit
contract: timeout cannot look like a successfully completed advisory judgment.
Existing usage, findings, and account-unavailable classifications retain their
current exit codes. A transport failure that completes before the deadline
retains its existing per-request degradation semantics.

Keep pending cache writes private to the run. Publish only validated,
cache-eligible verdicts from a fully completed check, and publish no new entries
from a timed-out, cancelled, or unavailable run. Previously committed entries
remain readable and unchanged. The implementation must define a commit point
that cannot expose a partially published run when termination races publication;
an in-memory queue followed by individual visible writes is insufficient.
A check with legitimate fail findings can complete and publish cache entries.

### Redacted timing receipts

Opt-in `--timings` writes structured stage and request/attempt timing events to
stderr, never verdict stdout. Include elapsed and remaining milliseconds,
stage, request sequence, attempt number, retry delay, outcome category, and
HTTP status where available. Timeout always emits its final receipt. Do not
include credentials, URLs, prompts, source text, model output, paths, or raw
exception messages. Concurrent requests have distinct sequence numbers but
share one deadline. Timing distinguishes preparation, transport waiting, and
backoff; it cannot assert whether a remote provider was queueing or computing.

## Why

One outer boundary covers synchronous preparation and transports that ignore
cancellation. Explicit adapter limits make underlying work stoppable and retry
cost observable. Delayed publication prevents unfinished runs from contributing
new cached evidence. None of these mechanisms establishes useful evaluation
latency; they make failures bounded and diagnosable.

## Qualification and delivery

After acceptance, deliver focused implementation PRs for supervision/exit
behavior, adapter cancellation/retries and timing, and cache publication. Keep
#95 open until the integrated behavior is verified. Deterministic tests must
cover blocked preparation, a never-responding transport, repeated transient
failures, cancellation during backoff, concurrent and sequential requests
sharing one deadline, and expiration during publication. Assert process cleanup,
no partial stdout, unchanged existing cache, no new visible cache entries, and
redaction even when an error contains a fake credential or prompt.

Then record useful representative Overlook `doc-drift` and
`agent-rule-conflict` evaluations within 120 seconds, using the unchanged
qualified routes and timing receipts. Only that evidence supports a downstream
pin update and restoring the two suspended checks. A timeout-only patch does
not complete qwts/overlook#1176.

## Consequences and tradeoffs

- Supervision and atomic publication add process and filesystem complexity;
  cross-platform cleanup and crash recovery become explicit test obligations.
- A strict budget can leave large checks and calibration suites unqualified.
  Report that limitation; do not silently sample away required evidence.
- Explicit retry bounds may expose transient failures previously hidden by SDK
  retries, while preventing hidden backoff from exhausting the entire run.
- Staging cache writes uses temporary storage and needs bounded cleanup; the
  implementation must preserve atomic visibility without accumulating abandoned
  run directories indefinitely.
- This record is a proposal, not accepted architecture or completed restoration.
