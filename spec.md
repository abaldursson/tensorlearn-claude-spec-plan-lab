# Specification

## Requirements

- R1 (I-1, release gate): notify a patient only after a doctor has reviewed and released the result; a corrected result with a new released version notifies again (release signal: ASSUME-1).
- R2 (I-2, privacy): clinical content stays out of notifications and logs and is shown only in the authenticated portal; notifications carry no patient identifier (the link has an opaque result ID, ASSUME-2) and logs hold event IDs and delivery states only.
- R3 (I-3, portal reuse): the link uses the existing route in `src/portal/result-route.mjs` (`/portal/results/:resultId`), unchanged, with no portal redesign.
- R4 (I-4, idempotency): the idempotency key is the released result version, so one released version is delivered at most once; the delivered record is kept behind the proposed new file `src/service/notify.mjs` because `src/service/queue.mjs` holds no state (storage: ASSUME-3).
- R5 (I-4, provider failure): retries are bounded (maximum: DECIDE-1), never duplicate a delivery, and log a failed state when exhausted.
- R6 (I-4, provider limits): at most 50 requests per second and 20 messages per batch; one batch counts as one request.
- R7 (I-1, queueing): jobs go through the existing `enqueueNotification` in `src/service/queue.mjs`, reused as is.
- R8 (I-1, outcome): avoidable results-status calls are compared before and after release, with the measure and target set under DECIDE-2.

## Design

- D1: a gate-and-key step, proposed as new file `src/service/notify.mjs`, checks release, builds the idempotency key and calls `enqueueNotification` in `src/service/queue.mjs`.
- D2: the body is a fixed neutral message plus the existing portal link from `src/portal/result-route.mjs`, with no result details or patient identifiers; sending waits for OPEN-1.
- D3: the sender in the proposed new file `src/service/notify.mjs` batches at most 20 messages and paces at most 50 requests per second, with bounded retries keyed by the idempotency key.
- D4: logs hold only event ID and delivery state (queued, sent, failed).
- D5: the archived proposal (unapproved helper, notify on upload, result details in messages, portal rebuild) is rejected, excluded and not used.

## Open decisions and owners

- OPEN-1: channel choice for patient notifications; no channel is decided here. **Owner**: product owner and privacy officer. **Status**: open, needs approval.
- OPEN-2: referring-doctor notifications; not in scope. **Owner**: clinical lead and privacy officer. **Status**: open, out of scope.
- DECIDE-1: retry maximum and backoff within the provider contract, how a provider timeout with unknown outcome is handled, and whether the provider honours the idempotency key; no value invented. **Owner**: engineering lead. **Status**: open.
- DECIDE-2: measure and target for fewer results-status calls (I-1); none invented. **Owner**: product owner. **Status**: open.
- DECIDE-3: whether a queued notification is cancelled if a doctor retracts the release. **Owner**: clinical lead. **Status**: open.
- ASSUME-1: source of the doctor-release signal and version is not in current source. **Owner**: engineering lead and clinical lead. **Status**: to confirm before step 1.
- ASSUME-2: result ID in the link is opaque and identifies no patient, and the fixed message wording is acceptable. **Owner**: privacy officer. **Status**: to confirm before step 2.
- ASSUME-3: where the delivered record per released version is stored is not in current source. **Owner**: engineering lead. **Status**: to confirm before step 3.

## Traceability

| Intent | Requirement | Test |
| --- | --- | --- |
| I-1 notify after review and release | R1, R7, R8 | `gating_only_released_results_notify`: unreleased results enqueue nothing, released ones enqueue one job; `status_call_reduction_measured`: the DECIDE-2 measure is recorded before and after release |
| I-2 clinical content stays in the portal | R2 | `privacy_no_clinical_content_in_payload_or_logs`: payload and logs, including failed and retried sends, hold no identifiers or clinical content |
| I-3 reuse the portal route | R3 | `portal_link_uses_existing_route`: link built from `/portal/results/:resultId` with an opaque ID; `git diff` shows the route file unchanged |
| I-4 no duplicates, provider failure handled | R4, R5, R6 | `duplicate_retry_same_version_delivers_once`, `retry_bound_then_failed_state_logged`, `throughput_boundary_50_rps_20_per_batch`: one delivery per version, retries stop at the bound, 20 messages fit one batch and 21 split, 50 requests per second pass and 51 are delayed |
