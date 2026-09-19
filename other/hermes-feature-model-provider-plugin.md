# Hermes Feature 126: Model Provider Plugin Surface

## Source and capability

Source page: Hermes **Build a Hermes Plugin** developer guide. Hermes can load a model-provider plugin that declares an inference backend while leaving the agent planner, tool registry, memory, and CLI surface unchanged.

## Business value

SchoolRoom may need to switch between a local model and a managed provider without changing the natural-language integration contract. A provider plugin can isolate provider-specific transport details and keep the business workflow stable.

## Proposed SchoolRoom integration

Keep provider selection in Hermes. The integration layer should consume the same structured intent regardless of provider and should record only a safe provider class such as `local` or `managed`, not credentials or raw endpoint details.

Responsibilities:

- Hermes provider plugin: translate Hermes requests to the selected inference backend.
- Integration layer: validate provider capability and maintain model-neutral intent tests.
- SchoolRoom API: remain provider-agnostic.
- Database: no schema change; safe provider category may be audit metadata.
- Frontend: show operational status, not provider secrets or raw endpoints.

## End-to-end data flow

1. Operator selects a provider implementation in Hermes.
2. Hermes sends the same intent-extraction prompt through that provider.
3. The provider returns a structured response.
4. The integration layer validates and matches it.
5. SchoolRoom persists the result through the existing API.

## Interface and schema changes

Add a provider capability health check and a stable structured-output contract. No student/comment schema change is needed. Provider configuration must remain outside tracked showcase files.

## Feasibility, dependencies, and risks

Feasible, but provider plugins can introduce incompatible tool-calling behavior, telemetry, or data residency. Require capability tests, timeouts, redaction, plugin review, and an explicit fallback to read-only mode when the provider is unavailable.

## Recommendation

Use the provider plugin surface to support local/private inference, but make the integration contract and acceptance suite provider-neutral. Never couple business logic to one model SDK.
