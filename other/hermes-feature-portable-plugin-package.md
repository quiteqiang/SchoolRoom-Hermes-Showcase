# Hermes Feature 130: Portable Agent Plugin Packages

## Source and capability

Source page: Hermes **Build a Hermes Plugin** developer guide. Hermes can load a portable package containing a manifest, skills, references, and MCP definitions through a compatibility-oriented plugin format, separate from native Python plugins.

## Business value

SchoolRoom could distribute a reviewed “classroom comments” skill bundle without asking operators to fork Hermes. The package can document the intent schema, examples, and safety checklist while leaving the backend implementation in SchoolRoom.

## Proposed SchoolRoom integration

Create a public, credential-free skill package that teaches Hermes how to phrase requests and use the allowlisted SchoolRoom tools. Keep all secrets, endpoints, and deployment configuration outside the package.

Responsibilities:

- Hermes package: provide progressive-disclosure instructions, examples, and optional tool metadata.
- Integration layer: expose the actual typed operations and validate every call.
- SchoolRoom API: remain the only write authority.
- Database: no schema change.
- Frontend: document package version and show compatibility status to operators.

## End-to-end data flow

1. An operator installs a reviewed package.
2. Hermes loads the skill only when the user asks about classroom comments.
3. The skill guides Hermes to emit the structured intent or call an allowlisted tool.
4. The integration layer validates the request and calls the API.
5. SchoolRoom returns a pending or completed result for the normal frontend review flow.

## Interface and schema changes

Version the skill/tool contract and include only abstract operation names. Use a compatibility check against the backend API version. Do not put real URLs, tokens, local paths, student fixtures, or operational server settings in the package.

## Feasibility, dependencies, and risks

Feasible and useful for repeatable onboarding. Skills are untrusted instructions and can become stale or be replaced by a malicious package. Pin reviewed versions, require operator consent, scan package contents, and keep API authorization independent of skill text.

## Recommendation

Publish a small credential-free package after the typed API contract stabilizes. Treat it as documentation and orchestration guidance, never as a security boundary.
