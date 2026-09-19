# Hermes Feature 125: Middleware Policy Enforcement

## Source and capability

Source page: Hermes **Middleware** developer guide. Middleware can rewrite LLM requests or tool arguments, wrap execution, attach trace metadata, and preserve the normal retry, approval, and hook pipeline.

## Business value

Middleware can provide a single policy point for SchoolRoom: enforce bounded prompts, normalize structured tool arguments, attach correlation IDs, and block any attempt to call an unapproved endpoint before execution.

## Proposed SchoolRoom integration

Implement a minimal policy middleware in Hermes only for request shaping and correlation metadata. Keep business validation in the integration layer; middleware should not become a second database authorization system.

Responsibilities:

- Hermes middleware: reject oversized or malformed intent requests and add trace metadata.
- Integration layer: perform roster matching, role checks, and comment policy validation.
- SchoolRoom API: enforce the final authorization and persistence boundary.
- Database: no schema change; correlation metadata may go to logs or an audit store.
- Frontend: use correlation IDs to explain a failed or pending request without showing raw model prompts.

## End-to-end data flow

1. Hermes constructs the model request or tool call.
2. Middleware applies size, operation, and destination constraints.
3. Normal Hermes approvals and hooks run on the effective arguments.
4. The integration layer validates the result and calls the API.
5. Trace metadata links the gateway turn to the backend result.

## Interface and schema changes

Define an internal trace envelope with a correlation ID and policy decision. Do not let middleware create arbitrary new business fields or rewrite a student ID after matching. Add tests proving rejected requests never reach the API.

## Feasibility, dependencies, and risks

Feasible, but middleware runs before some downstream guardrails, so a bad rewrite could weaken policy. Keep it deterministic, fail closed for SchoolRoom write tools, log the effective arguments safely, and review every plugin update.

## Recommendation

Use middleware for transport and shape controls, not as the source of business truth. Make the backend API the final authority and keep the middleware small enough to audit.
