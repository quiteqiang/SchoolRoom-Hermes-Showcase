# Hermes Feature 119: Model Context and Tool-Calling Capability Gate

## Source and capability

Source pages: Hermes **Quickstart** and **Run Hermes Locally with Ollama** guides. Hermes requires a large enough context window for multi-step tool use, and a model without tool-calling support can chat but cannot reliably perform agent actions.

## Business value

This gate prevents a misleading deployment where Hermes can answer a teacher but cannot safely resolve a student or call the SchoolRoom API. It also makes local-model selection explainable to operators.

## Proposed SchoolRoom integration

Run a capability check when the Hermes profile starts and before enabling write tools. The integration layer should fail closed if the model cannot emit the required structured intent or if the effective context budget is below the tested threshold.

Responsibilities:

- Hermes: expose model metadata and tool-calling behavior.
- Integration layer: run a harmless capability probe and verify the intent schema.
- SchoolRoom API: no change; it must not infer capability from user-agent text.
- Database: no change.
- Frontend: show “read-only” or “write-enabled” status and the reason for a disabled write path.

## End-to-end data flow

1. The profile starts and Hermes identifies the selected model.
2. The integration layer runs a non-mutating structured-output probe.
3. If the probe passes, comment creation tools are enabled; otherwise only read-only help is exposed.
4. Every write still goes through roster validation and review.

## Interface and schema changes

Add a capability result to the internal health contract: `structured_output`, `tool_calling`, `context_capacity`, and `checked_at`. Do not store it in student or comment records.

## Feasibility, dependencies, and risks

Feasible and inexpensive. A probe can pass while real Chinese classroom phrasing still performs poorly, and model updates can change behavior. Cache results only briefly, test representative utterances offline, and keep a manual read-only fallback.

## Recommendation

Make capability gating mandatory for write-producing profiles. Treat model eligibility as an operational precondition, not as a guarantee of correct student matching.
