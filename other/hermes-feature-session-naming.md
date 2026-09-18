# Hermes Feature 112: Session Naming and Auto-Titles

## Source and capability

Source page: Hermes **Sessions** user guide. Hermes can assign a short title after the first exchange, accept an explicit title command, and resume a conversation by title. Titles are session metadata rather than part of the business records.

## Business value

Teachers may have several conversations about the same class in one day. Human-readable titles such as “Year 1 Otters — Monday comments” make it easier to find the conversation that produced a draft without putting chat history into the SchoolRoom database.

## Proposed SchoolRoom integration

Use the gateway session metadata as the extension point. The integration layer may suggest a safe title from the class and date, but it must not use a title as a student identifier or authorization credential.

Responsibilities:

- Hermes: create, update, and resume the conversation title.
- Integration layer: optionally propose a normalized, non-sensitive title from validated class/date context.
- SchoolRoom API: expose comment provenance as an opaque `source_session_id` only if audit needs it.
- Database: optionally add a nullable provenance field to comments; do not duplicate full transcripts.
- Frontend: show a compact source label and link to the review context, not the entire chat log.

## End-to-end data flow

1. Hermes receives the first classroom request and assigns a session.
2. The integration layer validates the class context and returns a title suggestion.
3. Hermes stores the title in session metadata.
4. A resulting comment stores only an opaque session reference if audit is enabled.
5. The frontend can display the reference beside a pending comment without exposing gateway internals.

## Interface and schema changes

The minimum change is none: titles can remain gateway-only. If audit is required, add an optional opaque `source_session_id` to the comment interface and database model. Never persist message text, chat IDs, or platform credentials in that field.

## Feasibility, dependencies, and risks

This is low effort and independent of the model provider. Auto-generated titles can accidentally include student names or sensitive observations, and title collisions can make resume ambiguous. Apply length limits, character filtering, and a rule that titles use class/date labels rather than individual student names.

## Recommendation

Enable title suggestions for teacher workflow and debugging, but keep them outside the two core tables unless an explicit audit requirement justifies one nullable provenance field.
