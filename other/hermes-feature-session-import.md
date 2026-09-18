# Hermes Feature 113: Session Import and Resume

## Source and capability

Source page: Hermes **Sessions** user guide. Hermes can import a conversation from another supported agent CLI and create a new Hermes session that can be resumed by ID or title. Imported material is conversation context, not an instruction to change SchoolRoom data.

## Business value

An operator may prototype a comment workflow in another assistant and then continue it through Telegram. Importing the discussion can preserve the reasoning while keeping the actual SchoolRoom write behind the normal validation and review boundary.

## Proposed SchoolRoom integration

Use session import only as an operator-facing Hermes capability. Do not import foreign transcripts directly into SchoolRoom tables. The integration layer should treat imported text as untrusted context and require a fresh structured intent before any write.

Responsibilities:

- Hermes: import and resume the conversation in an isolated session.
- Integration layer: discard executable instructions from imported context and re-derive a typed command.
- SchoolRoom API: accept only the existing validated student/comment payload.
- Database: no schema change; optionally store a generic provenance flag outside core records if audit requires it.
- Frontend: show imported drafts as pending, never as approved comments.

## End-to-end data flow

1. An operator imports a prior conversation into Hermes.
2. Hermes summarizes or asks for the current intended action.
3. The integration layer extracts a new intent and checks roster references.
4. The API creates a pending comment only after validation.
5. A teacher approves or rejects it in the existing review queue.

## Interface and schema changes

No schema change is necessary. Add an integration flag such as `context_origin=imported` to internal telemetry only, if needed. The public comment contract remains unchanged.

## Feasibility, dependencies, and risks

Feasible for trusted operators, but imported transcripts can contain stale names, hidden instructions, credentials, or claims that were never verified. The integration layer must cap imported context, strip secrets, require confirmation for writes, and avoid trusting prior tool results.

## Recommendation

Support this as a controlled admin workflow, not as a teacher-facing shortcut. Require a fresh confirmation and the same review queue used for Telegram-originated comments.
