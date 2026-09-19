# Hermes Feature 121: DM Pairing Authorization

## Source and capability

Source page: Hermes **Security** guide. Unknown direct-message senders can receive a one-time pairing code, while an operator approves the sender before normal conversation access is enabled. Alternatives include silently ignoring or declining unauthorized messages.

## Business value

Pairing gives a teacher a controlled way to connect a new Telegram identity without publishing a permanent allowlist in application code. It reduces accidental access to student data while keeping onboarding manageable for a small school team.

## Proposed SchoolRoom integration

Use Hermes pairing as the channel authorization layer. The integration layer must still enforce application roles and must not treat a paired chat as proof that the sender may view every class or approve every comment.

Responsibilities:

- Hermes: issue, expire, and verify the pairing code and bind the approved identity to a channel.
- Integration layer: map the approved channel identity to a SchoolRoom role and class scope.
- SchoolRoom API: authorize each read/write operation using the mapped role and scope.
- Database: optionally store a hashed external-identity reference in an access-control store, not in student/comment rows.
- Frontend: provide an admin-only view of paired identities and their scopes.

## End-to-end data flow

1. An unknown sender contacts the gateway.
2. Hermes returns a short-lived pairing code without loading student data.
3. An administrator approves the code through the operator interface.
4. The integration layer creates or updates a scoped principal.
5. Later messages carry that principal into intent extraction and API authorization.

## Interface and schema changes

Define a principal contract containing `channel`, `external_identity_hash`, `role`, `class_scope`, `expires_at`, and `status`. Keep this separate from the two business tables. Add an authorization check to every student/comment endpoint if one is not already present.

## Feasibility, dependencies, and risks

Feasible and low impact. Pairing codes can be forwarded, identities can change, and a paired user may still attempt prompt injection or over-broad requests. Use short expiry, one-time redemption, audit logging, explicit class scope, and immediate revocation.

## Recommendation

Adopt pairing for channel onboarding, but treat it as authentication only. Authorization and review permissions must remain enforced by the integration layer and SchoolRoom API.
