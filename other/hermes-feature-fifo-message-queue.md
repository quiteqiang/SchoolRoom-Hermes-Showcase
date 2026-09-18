# Hermes Feature 116: Explicit FIFO Message Queue

## Source and capability

Source page: Hermes **Gateway Session Lifecycle** guide. Explicit queue commands preserve message order: each queued event produces its own full agent turn, and burst follow-ups are promoted one at a time instead of being silently merged.

## Business value

During a busy classroom, a teacher may send several short observations in succession. FIFO processing preserves the order of those observations and makes it possible to reconcile a later correction with the earlier draft.

## Proposed SchoolRoom integration

Use Hermes queueing at the gateway boundary and add a small integration-level event envelope. The SchoolRoom API should remain synchronous and simple; queue state belongs to Hermes or a durable message broker, not to the two business tables.

Responsibilities:

- Hermes: maintain per-session pending and overflow queues.
- Integration layer: assign sequence numbers, preserve the original text, and validate each event independently.
- SchoolRoom API: process one validated command at a time and return a durable result.
- Database: no student/comment schema change; optional internal receipt storage for sequence and retry state.
- Frontend: display queued, processing, and review states.

## End-to-end data flow

1. Gateway assigns a session key and sequence number.
2. Messages enter the single pending slot or FIFO overflow queue.
3. Hermes processes the head event and emits a typed intent.
4. The integration layer validates and calls the API.
5. The gateway promotes the next event only after the current event has a durable result.

## Interface and schema changes

Add `request_id`, `session_key_hash`, and `sequence` to an internal event envelope. Do not expose platform chat IDs in the SchoolRoom API. A durable queue is recommended if losing in-memory events is unacceptable.

## Feasibility, dependencies, and risks

Feasible for a single gateway, but memory-only queues can lose work on restart and unbounded overflow can consume resources. Duplicate delivery is possible after a crash, so the API still needs idempotency. Queue contents may include student names and comments and must be protected with the same access controls as messages.

## Recommendation

Adopt FIFO semantics for bursty classroom messages, with bounded depth, durable receipts, and a visible “queued” acknowledgement. Keep the business database free of queue implementation details.
