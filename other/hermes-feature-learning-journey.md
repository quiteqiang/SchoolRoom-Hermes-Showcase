# Hermes Feature 117: Learning Journey

## Source and capability

Source page: Hermes **Persistent Memory** guide. Hermes can present a timeline of saved memories and skills, showing how its reusable context changed over time. This is an explanation and review surface, not a replacement for the application database.

## Business value

A teacher or administrator can inspect why Hermes keeps preferring a class alias or a particular comment style. This makes personalization understandable and helps remove outdated classroom assumptions before they affect future commands.

## Proposed SchoolRoom integration

Expose the learning journey only through an admin-facing Hermes workflow. Convert approved, domain-specific preferences into versioned integration policy, not into student or comment rows.

Responsibilities:

- Hermes: collect and display memory/skill change history.
- Integration layer: classify entries as operational preference, temporary context, or prohibited student data.
- SchoolRoom API: optionally provide a read-only policy endpoint for approved class labels and comment templates.
- Database: no change to the two business tables; if needed, store policies in a separate configuration store owned by the integration layer.
- Frontend: show the source and approval status of any policy used during matching.

## End-to-end data flow

1. Hermes proposes a memory or skill change after repeated teacher corrections.
2. An administrator reviews the learning journey.
3. Approved operational preferences are exported as a versioned policy.
4. The integration layer loads only that policy when building an intent request.
5. Student records and comments remain independent of Hermes learning history.

## Interface and schema changes

No business schema change. Define a small policy contract with `version`, `allowed_aliases`, `comment_style`, and `approved_by`; keep free-form memory out of the runtime authorization path.

## Feasibility, dependencies, and risks

Feasible with a review gate. Unreviewed learning could retain sensitive student information, learn a wrong alias, or turn a one-off correction into a permanent rule. Apply retention limits, PII filtering, provenance, and explicit rollback.

## Recommendation

Use the learning journey as an audit and governance feature. Permit only reviewed operational preferences to influence SchoolRoom matching; never auto-promote student observations into persistent agent memory.
