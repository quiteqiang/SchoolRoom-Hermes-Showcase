# Hermes Feature 129: Memory-Provider Session Flush

## Source and capability

Source page: Hermes **Gateway Internals** guide. When a session ends, resets, or an agent cache entry is evicted, Hermes can flush durable conversation state to the selected memory provider before releasing runtime resources.

## Business value

Explicit flush boundaries can preserve approved operational preferences while preventing half-finished classroom conversations from becoming reusable context. This is useful when the gateway is restarted or when a teacher changes from one class to another.

## Proposed SchoolRoom integration

Use flush events to trigger a privacy review, not to copy the memory provider into SchoolRoom. The integration layer should classify any candidate memory and allow only approved, non-student policy entries to influence future intent extraction.

Responsibilities:

- Hermes: emit session-end/reset/eviction lifecycle events and coordinate the provider flush.
- Integration layer: receive a bounded summary or event, run PII and class-scope checks, and version approved policy.
- SchoolRoom API: remain independent of memory-provider state.
- Database: no student/comment schema change; policy metadata belongs in a separate store.
- Frontend: show when a policy review is pending, without displaying raw memory content to ordinary teachers.

## End-to-end data flow

1. A session reaches an explicit boundary or is evicted from the cache.
2. Hermes flushes the provider and emits a lifecycle event.
3. The integration layer reviews candidate operational facts.
4. Approved aliases or style rules receive a version and expiry.
5. Future commands use only the approved policy version.

## Interface and schema changes

Define a lifecycle payload with opaque session reference, boundary type, policy candidates, and flush status. Keep it asynchronous and retryable. Do not add memory-provider IDs to student/comment rows.

## Feasibility, dependencies, and risks

Feasible if the provider supports reliable lifecycle callbacks. Flush failures must not block a comment transaction, and stale memory can cause incorrect matching. Use retries, dead-letter handling, retention limits, explicit approval, and policy expiry.

## Recommendation

Adopt flush events for governance and cleanup, not automatic personalization. The SchoolRoom backend remains authoritative for students and comments.
