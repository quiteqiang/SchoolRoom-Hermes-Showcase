# Hermes Feature 114: Session Lineage After Compression

## Source and capability

Source page: Hermes **Sessions** user guide. Hermes stores a parent-session relationship when long conversations are split or compressed. The active session can therefore continue with a smaller context while retaining a navigable lineage.

## Business value

Long teacher conversations may contain many observations and corrections. Session lineage helps operators understand which compacted context produced a draft without forcing the SchoolRoom database to store a second copy of the chat.

## Proposed SchoolRoom integration

Use the gateway session store as the extension point and expose only an opaque lineage reference to SchoolRoom. The integration layer should use the active session ID for idempotency and keep parent IDs out of natural-language matching.

Responsibilities:

- Hermes: create child sessions and preserve parent relationships.
- Integration layer: attach the active opaque session reference to a command envelope and prevent duplicate writes across retries.
- SchoolRoom API: accept an optional idempotency key or provenance token.
- Database: optionally add a unique idempotency record or nullable source token; do not mirror the session graph.
- Frontend: show “continued session” as audit metadata only.

## End-to-end data flow

1. Hermes compresses a long chat and starts a child session.
2. The child session emits a new command envelope with a stable idempotency token.
3. The API checks the token before inserting a pending comment.
4. Retries from either session resolve to the same pending item.
5. The frontend presents one review item with a compact provenance indicator.

## Interface and schema changes

Prefer an idempotency header or request field at the API boundary. If durable deduplication is required, add a small internal write-receipt table; do not add parent-session columns to the student or comment tables unless audit needs them.

## Feasibility, dependencies, and risks

Feasible and useful for reliability. Session IDs are operational identifiers and must not be displayed as authorization. A lost or regenerated token could create duplicates, while over-trusting lineage could make stale context look authoritative. Deduplication must be scoped to the authenticated teacher and intended operation.

## Recommendation

Adopt the idempotency part first. Treat lineage as observability metadata and keep the core business schema minimal.
