# Implementation plan

## Ordered steps

1. Capture the baseline count of avoidable results-status calls (R8, measure from DECIDE-2) before any change, then add the release-gate check (R1) in the proposed new file `src/service/notify.mjs`, so only doctor-released results continue (starts once ASSUME-1, the release signal, is confirmed); write the `gating_only_released_results_notify` test first.
2. Build the privacy-safe notification payload and logging (R2, R3) in the proposed new file `src/service/notify.mjs`, building the link by filling the result ID into the route from the existing `src/portal/result-route.mjs`, which stays unchanged; write the `privacy_no_clinical_content_in_payload_or_logs` and `portal_link_uses_existing_route` tests first.
3. Derive the idempotency key from the released result version in the proposed new file `src/service/notify.mjs` and enqueue through the existing `enqueueNotification` in `src/service/queue.mjs` (R4, R7), after step 2 and once ASSUME-3 (delivered-record storage) is confirmed; write the `duplicate_retry_same_version_delivers_once` test first.
4. Add bounded retry (maximum from DECIDE-1), batches of at most 20 messages and pacing of at most 50 requests per second (R5, R6) in the proposed new file `src/service/notify.mjs`, after step 3, with sending in a dry run until OPEN-1 is approved; write the `retry_bound_then_failed_state_logged` and `throughput_boundary_50_rps_20_per_batch` tests first.
5. Compare the status-call count with the step 1 baseline once sending is approved and live (R8), run all tests and the path check, then review that the archived advice stays rejected and not used.

## Paths and dependencies

- Existing path `src/portal/result-route.mjs` is reused unchanged for the patient link; it has no dependency on the other steps.
- Existing path `src/service/queue.mjs` is reused for enqueueing via `enqueueNotification`; steps 3 and 4 depend on it.
- Proposed new file `src/service/notify.mjs` holds the gate, key, batching and retry logic; it is the only new source file, and the tests are described by name only and not created in this lab.
- Dependency order: step 1 (gate) before step 2 (payload), step 2 before step 3 (key and enqueue), step 3 before step 4 (retry, batching and sending), and step 5 last.
- The archived helper and its portal rebuild are rejected and not used; no step depends on them, and nothing here notifies on upload.
- OPEN-1 (channel, owners: product and privacy) stays unresolved and blocks real sending in step 4 until approved; OPEN-2 (referring doctors, owners: clinical and privacy) is out of scope and blocks no step.
- DECIDE-1 (retry maximum, timeout handling) must be set before step 4, ASSUME-3 (delivered-record storage) before step 3, and ASSUME-1 (release signal) and ASSUME-2 (opaque result ID, message wording) before steps 1 and 2; DECIDE-2 (success measure) and DECIDE-3 (release retraction) are open and block no step; owners are in `spec.md`.

## Risks and verification

- Risk: a notification leaks clinical content or identifiers; mitigated by R2, the fixed neutral message and the privacy test, and clinical content stays out of notifications and logs.
- Risk: a retry delivers the same released version twice; mitigated by the idempotency key (the released result version) and the duplicate-retry test.
- Risk: the provider limit of 50 requests per second or 20 messages per batch is exceeded; mitigated by batching and pacing and the boundary test at 20, 21, 50 and 51.
- Risk: an unreleased result is notified; mitigated by the release gate and the gating test.
- Risk: a provider timeout with an unknown outcome double-sends; mitigated by the idempotency key and the answer to DECIDE-1 on whether the provider honours it.
- Risk: a release is retracted after a notification is queued; open under DECIDE-3 (owner: clinical lead).
- Risk: the delivered record is lost on restart, which would break idempotency; mitigated by confirming ASSUME-3 before step 3.
- Risk: sending goes live before OPEN-1 and DECIDE-1 are resolved; mitigated by keeping it disabled or in a dry run until they are.
- Risk: error or retry logs leak provider response bodies or payloads; mitigated by the privacy test also covering failed and retried sends.
- Risk: pacing holds only inside one process, not across several workers, so 50 requests per second could be exceeded; open for the engineering lead under DECIDE-1.
- Risk: open items stay unresolved; each goes to its owner in `spec.md` before the step that depends on it, and none is decided in this plan.
- Tests in this lab are descriptions by name only, and any fake provider needed to run them is a proposed new item, not created here.
- Verification: run `grep -on 'src/[A-Za-z0-9_./-]*\.mjs' spec.md plan.md`, check each path with `ls`, run `grep -n '^## ' spec.md plan.md` to check the headings, and confirm `git status` shows only `spec.md` and `plan.md` changed, with `intent.md` and `inputs/` untouched.

| Requirement | Step | Existing path | Test |
| --- | --- | --- | --- |
| R1 release gate (I-1) | 1 | - | `gating_only_released_results_notify` |
| R2 privacy (I-2) | 2 | - | `privacy_no_clinical_content_in_payload_or_logs` |
| R3 portal reuse (I-3) | 2 | src/portal/result-route.mjs | `portal_link_uses_existing_route` |
| R4, R7 idempotent enqueue (I-4, I-1) | 3 | src/service/queue.mjs | `duplicate_retry_same_version_delivers_once` |
| R5 bounded retry (I-4) | 4 | - | `retry_bound_then_failed_state_logged` |
| R6 provider limits (I-4) | 4 | - | `throughput_boundary_50_rps_20_per_batch` |
| R8 fewer status calls (I-1) | 1, 5 | - | `status_call_reduction_measured` |
