# Hermes Feature 115: Busy-Input Modes

## Source and capability

Source page: Hermes **Messaging Gateway** guide. When an agent is already working, a new message can interrupt the turn, wait in a queue, or steer the current run after the next tool boundary. Hermes can also send a short busy acknowledgement.

## Business value

Voice transcription and natural-language matching may take time. A teacher who sends a correction should be able to choose whether it replaces the active draft, waits behind it, or guides the current operation without creating an accidental duplicate comment.

## Proposed SchoolRoom integration

Configure the gateway mode per SchoolRoom channel. The integration layer remains idempotent and must treat every resumed or interrupted turn as a new candidate command until the API confirms the write.

Responsibilities:

- Hermes: control interrupt, queue, or steer behavior and communicate busy state.
- Integration layer: attach a request ID to every intent and reject stale writes.
- SchoolRoom API: accept idempotency keys and return the existing result on replay.
- Database: no schema change if request IDs are handled in an internal receipt store.
- Frontend: reflect pending, superseded, and approved states clearly.

## End-to-end data flow

1. Hermes receives a teacher message and begins intent extraction.
2. A second message arrives while the first is active.
3. The selected busy-input mode determines when Hermes processes the second message.
4. Each intent is validated independently and sent with a request ID.
5. The API persists at most one result per request ID; the review queue shows the final state.

## Interface and schema changes

Add a request ID to the integration envelope and support idempotent create-comment behavior. The API should expose a stable conflict/result response rather than allowing a retry to create a second comment.

## Feasibility, dependencies, and risks

Feasible, but steering can make the model reinterpret an in-progress task and queueing can delay urgent corrections. Interruption must not kill a database transaction halfway through. Use a transactional write boundary, explicit statuses, and an acknowledgement that says whether a message was queued or applied.

## Recommendation

Use queue mode as the default for comment creation, reserve interrupt/steer for conversational clarification, and require idempotency before enabling them for write-producing flows.
