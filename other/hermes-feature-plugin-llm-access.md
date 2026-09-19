# Hermes Feature 124: Plugin LLM Access

## Source and capability

Source page: Hermes **Plugin LLM Access** developer guide. A plugin can make a single host-owned completion or structured extraction call outside the main agent conversation, using the active provider, model, authentication, fallback, timeout, and audit path.

## Business value

SchoolRoom can use a small deterministic classifier for name resolution, category suggestion, or language normalization without asking the main agent to perform another conversational turn. This can reduce prompt size and make the integration boundary easier to test.

## Proposed SchoolRoom integration

Build a narrow Hermes plugin for structured intent extraction, or keep the extraction in the existing integration layer if the deployment does not need plugin packaging. The plugin must return typed data and must never write the database directly.

Responsibilities:

- Hermes plugin: call the host-owned structured extraction surface and emit a schema-validated result.
- Integration layer: apply business rules, confidence thresholds, and roster matching.
- SchoolRoom API: persist only validated requests.
- Database: no schema change.
- Frontend: show the same pending/approved states and an optional “structured extraction” provenance label.

## End-to-end data flow

1. Hermes receives a natural-language message.
2. The plugin sends bounded input and a fixed intent schema to the host model.
3. Hermes validates the structured response and records an audit purpose.
4. The integration layer resolves the student and applies policy.
5. The API creates a pending comment or returns a clarification request.

## Interface and schema changes

Define a versioned intent schema with an explicit `operation` enum and no SQL, shell, or URL fields. Add contract tests for invalid JSON, missing student references, low confidence, and duplicate requests.

## Feasibility, dependencies, and risks

Feasible and useful, but every call consumes the configured model budget and may see sensitive text. Keep inputs minimal, redact unrelated content, set strict token/time limits, and prevent plugin-level provider or model overrides unless explicitly approved.

## Recommendation

Use structured plugin LLM access for bounded extraction only. Keep authorization, matching, and database writes outside the plugin so the business layer remains independently testable.
