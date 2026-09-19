# Hermes Feature 128: Durable Board Events and Live Updates

## Source and capability

Source page: Hermes **Kanban Multi-Agent Board** guide. The board records append-only task events with monotonic IDs and can stream new events to a dashboard over a WebSocket, falling back to a board reload when many events arrive together.

## Business value

The same pattern can make the SchoolRoom review queue feel live: a teacher sees a new pending comment, approval, rejection, or clarification without repeatedly refreshing the page.

## Proposed SchoolRoom integration

Reuse the event-stream idea for review notifications, but keep the core student/comment tables simple. The event stream is a projection and delivery mechanism, not a second source of truth.

Responsibilities:

- Hermes: emit a normalized business-event request after a command result.
- Integration layer: publish only safe event types and map them to opaque comment IDs.
- SchoolRoom API: persist the comment transaction and expose an ordered event cursor.
- Database: add an append-only internal event table or outbox if live delivery is required; keep it separate from the two core tables.
- Frontend: subscribe, reconcile by cursor, and reload on gaps or bursts.

## End-to-end data flow

1. A comment transaction commits.
2. An outbox/event record is written atomically with the result.
3. The backend streams events after the frontend’s last cursor.
4. The frontend updates the queue or performs a full reload when a gap is detected.
5. Reconnects resume from the cursor without duplicating visible items.

## Interface and schema changes

Define events such as `comment.created`, `comment.approved`, and `comment.rejected`, each with `event_id`, `cursor`, `comment_id`, and `status`. Do not include raw student text in broadcast payloads unless the viewer is authorized.

## Feasibility, dependencies, and risks

Feasible, but live channels add authentication, reconnect, ordering, and backpressure concerns. A leaked subscription could reveal classroom activity. Scope each stream to the authenticated class, expire connections, and make the API reload authoritative.

## Recommendation

Implement an outbox plus cursor-based polling first; add WebSocket delivery when the review queue needs lower latency. Keep event history bounded and non-sensitive.
