# Hermes Feature 120: Gateway Proxy Mode

## Source and capability

Source page: Hermes **Environment Variables** reference. Proxy mode lets the gateway own platform I/O while forwarding agent work to a separate Hermes API server. This separates channel adapters from model execution and can support a controlled network boundary.

## Business value

SchoolRoom can keep Telegram-facing ingress isolated from the service that holds the integration tools and database access. It also creates a clean place to add authentication, rate limits, and audit logging between messaging and business operations.

## Proposed SchoolRoom integration

Use the proxy boundary as the Hermes extension point, but keep the SchoolRoom API behind the agent service. The proxy must forward normalized messages and receive normalized responses; it must never receive database credentials or execute arbitrary tools.

Responsibilities:

- Hermes gateway: receive platform messages, enforce channel authorization, and forward requests.
- Integration-layer service: host the intent parser, validation policy, and typed SchoolRoom client.
- SchoolRoom API: accept only authenticated, allowlisted operations.
- Database: remain private to the backend; no direct gateway connection.
- Frontend: connect to the backend or a separately authenticated read path, not to the gateway proxy.

## End-to-end data flow

1. The channel adapter receives a message and normalizes it.
2. The gateway authenticates the proxy request and forwards the session context.
3. The remote Hermes agent extracts a typed intent and calls the integration layer.
4. The backend validates and persists a pending or approved comment.
5. The result travels back through the proxy to the originating chat.

## Interface and schema changes

Define an internal request envelope with correlation ID, bounded message content, channel scope, and an expiry. Use a private service authentication mechanism and replay protection. No student/comment schema change is required.

## Feasibility, dependencies, and risks

Feasible, but it adds latency and a second failure domain. Proxy compromise could expose conversations, while confused-deputy errors could let a channel invoke backend actions outside its scope. Enforce mutual authentication, allowlisted operations, short-lived correlation IDs, payload redaction, and explicit timeouts.

## Recommendation

Use proxy mode only when network isolation or independent scaling is required. For a small local deployment, a single gateway and backend is simpler; if proxy mode is adopted, make the integration service the only component with SchoolRoom write access.
