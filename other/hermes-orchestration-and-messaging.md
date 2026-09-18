# Hermes Orchestration and Messaging

This file contains the detailed ClassNote integration analyses for Hermes features 71–80. It is part of the public showcase documentation and contains no production credentials or deployment configuration.

### 71. Kanban Multi-Agent Board

Hermes Kanban is a durable task board shared across profiles. It supports named workers, dependencies, review states, comments, attachments, retries, and human handoffs through a persistent queue ([official guide](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban)).

**ClassNote integration analysis**

- **Business value:** Useful for coordinating a batch of review tasks across classes or routing imported observations through extraction, validation, and human review.
- **Extension point:** Use Kanban as an orchestration layer above the ClassNote API, not as a replacement for the student/comment tables.
- **Responsibilities:** Hermes dispatches named workers; the integration layer gives each task a fixed tenant/class scope; ClassNote API resolves students and owns comment state; the frontend displays the canonical review result.
- **Data flow:** `task created → worker reads scoped task → read/transform calls → draft or review handoff → teacher approval → idempotent ClassNote API write`.
- **Interfaces/schema:** Define task metadata for owner, class scope, acceptance criteria, request ID, and idempotency key. Keep Kanban state in Hermes; no new ClassNote table is needed initially.
- **Feasibility:** Medium; dispatcher supervision, worker profiles, retry rules, attachment handling, and cancellation must be operated reliably.
- **Privacy/security:** Treat board comments and attachments as student-sensitive. Isolate boards and tenants, limit worker tools, expire attachments, and never grant workers direct database access.

**Recommendation**

Use Kanban later for multi-class or multi-role review pipelines. Keep single-student Telegram comments on the simpler direct tool path, and require the ClassNote API plus teacher confirmation for every write.

### 72. Session Heartbeats

Hermes `/heartbeat` adds one recurring instruction to the current conversation. When the session is idle and the interval elapses, Hermes injects a normal user-role turn using the same context and session state ([official guide](https://hermes-agent.nousresearch.com/docs/user-guide/features/heartbeat)).

**ClassNote integration analysis**

- **Business value:** A teacher or administrator can keep an active review conversation watching for meaningful changes without opening a new chat.
- **Extension point:** Use a heartbeat for supervised monitoring of a review queue, not for silent business mutations.
- **Responsibilities:** Hermes schedules the session turn; the integration layer calls a scoped read-only summary tool; ClassNote API checks current access; the frontend or messaging adapter delivers only meaningful changes.
- **Data flow:** `active review session → heartbeat prompt → current API read → change/no-change decision → visible summary or safe silence`.
- **Interfaces/schema:** Add a read-only queue-summary contract with owner, class scope, cursor, and last-seen request ID. No database schema change is required.
- **Feasibility:** High for an active supervised session; lower for long-lived unattended monitoring because session ownership and retention need explicit rules.
- **Privacy/security:** Re-check permissions on every tick, avoid embedding student data in the standing instruction, cap frequency, and never let a heartbeat infer confirmation for a comment.

**Recommendation**

Use heartbeats for short-lived, teacher-controlled review monitoring. Use the existing scheduled-task design for durable reports and require a fresh explicit confirmation before any write.

### 73. Recurring Loops

Hermes `/loop` repeats a prompt or slash command on a timer inside the current session, reading current state on each wakeup. Unlike a goal, it is timer-driven rather than judge-driven ([official guide](https://hermes-agent.nousresearch.com/docs/user-guide/features/loops)).

**ClassNote integration analysis**

- **Business value:** Periodically check queue depth, failed imports, or a long-running report and notify an administrator when something changes.
- **Extension point:** Connect loop prompts to read-only ClassNote reporting tools with a fixed scope and bounded result size.
- **Responsibilities:** Hermes owns cadence and stop behavior; the integration layer maintains the query contract; ClassNote API returns current authorized state; the frontend shows the latest result.
- **Data flow:** `loop timer → scoped API query → compare with last result → notify on change or remain quiet → repeat until stopped`.
- **Interfaces/schema:** Use a transient cursor or result hash for change detection. No core table change is needed; any durable monitoring state should remain separate from student/comment records.
- **Feasibility:** High for short monitoring windows; medium for continuous operation because timers, restarts, rate limits, and stale sessions must be handled.
- **Privacy/security:** Never place student names in loop titles, enforce per-run authorization, cap query volume, and stop when the session owner or class scope is unavailable.

**Recommendation**

Use loops for operational read-only monitoring, not for automatic comment creation or approval. For recurring teacher digests that must survive session changes, prefer a separately owned scheduled job.

### 74. Managed Scope

Hermes managed scope lets an administrator pin selected configuration and secret values so standard users cannot override them, while leaving unrelated settings user-controlled ([official guide](https://hermes-agent.nousresearch.com/docs/user-guide/managed-scope)).

**ClassNote integration analysis**

- **Business value:** An organization can enforce a safe baseline for the classroom assistant across profiles and machines.
- **Extension point:** Apply managed policy to Hermes tool availability, approval mode, privacy redaction, provider selection, and allowed ClassNote environment; keep business authorization in the API.
- **Responsibilities:** IT owns immutable runtime policy; Hermes enforces configuration precedence; the integration layer exposes the approved tool contract; ClassNote API enforces teacher, class, and record permissions.
- **Data flow:** `managed policy + local profile → Hermes resolved runtime → scoped ClassNote tool → API authorization → teacher response`.
- **Interfaces/schema:** Define a versioned policy contract listing required settings and prohibited capabilities. No student/comment schema change is required.
- **Feasibility:** Medium to high for organization-managed deployments; it depends on a reliable policy distribution and a clear support process when local settings conflict.
- **Privacy/security:** Managed scope can force safe values, but it can also centralize sensitive configuration. Keep secrets in a dedicated secret manager, audit policy changes, avoid exposing resolved credentials in diagnostics, and do not publish operational policy files.

**Recommendation**

Adopt managed scope for institutional deployments after the single-profile policy is stable. Pin safety-critical behavior, but keep tenant and record authorization exclusively in the ClassNote backend.

### 75. Tool-Use Enforcement

Hermes can inject model guidance that encourages an agent to make an actual tool call instead of merely describing an intended action. It is model-aware and can be enabled automatically or explicitly ([configuration reference](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**ClassNote integration analysis**

- **Business value:** Reduce false assurances such as “I added the comment” when the model never called the ClassNote API.
- **Extension point:** Apply enforcement to the Hermes ClassNote profile, while API results remain the only evidence of business success.
- **Responsibilities:** Hermes steers tool behavior; the integration layer validates arguments; ClassNote API executes and reports the operation; the frontend displays the returned state.
- **Data flow:** `teacher request → enforced tool call → API result → read-back or error → accurate confirmation message`.
- **Interfaces/schema:** Return structured result states such as `preview`, `confirmation_required`, `created`, and `rejected`. No database schema change is needed, but writes need request IDs and idempotency.
- **Feasibility:** High; it is a prompt/runtime policy with little application code, but it must be tested across the supported model set.
- **Privacy/security:** Enforcement is not authorization. Reject unknown tools and invalid arguments at the API boundary, avoid exposing raw backend errors, and never interpret a model’s prose as proof that a write happened.

**Recommendation**

Enable it for the production ClassNote profile as a reliability aid. Pair it with explicit confirmation, API-side validation, and read-back verification so the system remains correct when a model ignores or misinterprets the guidance.

### 76. Execution-Discipline Guidance

Hermes can add model guidance for persistent tool use, mandatory verification, count reconciliation, literal identifier handling, and verification-gated completion ([configuration reference](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**ClassNote integration analysis**

- **Business value:** Reduce incorrect claims such as treating a partial student lookup as complete or reporting a comment as saved without reading it back.
- **Extension point:** Apply the guidance to the ClassNote orchestration profile and encode the same invariants in typed API responses.
- **Responsibilities:** Hermes encourages the verification sequence; the integration layer preserves identifiers and interprets result states; ClassNote API performs authoritative checks and read-back; the frontend shows verified status.
- **Data flow:** `natural-language request → lookup and count reconciliation → preview → confirmed write → exact API read-back → user-facing completion`.
- **Interfaces/schema:** Return totals, pagination markers, canonical student identifiers, and post-write representations. No schema change is required if the existing API can expose these fields.
- **Feasibility:** High; it is complementary prompt guidance, but tests must cover empty, partial, malformed, and duplicated results.
- **Privacy/security:** Guidance cannot grant permissions. Keep raw student data out of prompts where possible, fail closed on count mismatches, and never normalize a malformed identifier into a different student.

**Recommendation**

Enable it for all ClassNote write-capable sessions. Make API read-back and idempotency mandatory so correctness does not depend on a model following the guidance perfectly.

### 77. Turn Liveness Watchdog

Hermes’s turn-liveness watchdog detects a conversation turn that has made no observable progress for too long, interrupts recovery, and allows stale-turn cleanup to reclaim a stuck session ([configuration reference](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**ClassNote integration analysis**

- **Business value:** A Telegram request cannot remain indefinitely “processing” because of a hung model call, tool, or network connection.
- **Extension point:** Configure the watchdog around the Hermes gateway and make the integration client use bounded API timeouts and cancellation.
- **Responsibilities:** Hermes detects no-progress turns; the integration layer maps interruption to a retry-safe status; ClassNote API commits transactions atomically; the frontend or messaging adapter tells the teacher whether retry is safe.
- **Data flow:** `message → Hermes turn → progress or timeout → cancellation/recovery → API outcome lookup → retryable error or confirmed result`.
- **Interfaces/schema:** Add request status and correlation IDs to the integration contract. No core schema change is needed if API writes are transactional and idempotent.
- **Feasibility:** High for operational resilience; thresholds need tuning so slow STT, approval waits, and long API reads count as progress rather than false stalls.
- **Privacy/security:** Do not retry a possibly committed write blindly. Read the target by idempotency key, avoid exposing internal timeout details, and ensure cancellation does not leave partially persisted comments.

**Recommendation**

Enable the watchdog for gateway sessions with conservative limits. Pair it with API idempotency, status read-back, and clear user-facing retry instructions.

### 78. Group Chat Session Isolation

Hermes can limit the number of active sessions and keeps messaging conversations scoped to their chat origin, preventing idle or unrelated sessions from consuming all available runtime capacity ([configuration reference](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**ClassNote integration analysis**

- **Business value:** Prevent one busy staff group from starving direct teacher conversations and reduce the chance of context crossing chat boundaries.
- **Extension point:** Map each approved chat or thread to a ClassNote tenant and role scope before creating a tool-enabled session.
- **Responsibilities:** Hermes owns session lifecycle and capacity; the integration layer binds chat identity to scope; ClassNote API re-checks authorization; the frontend distinguishes group review from private teacher work.
- **Data flow:** `chat/thread identity → isolated Hermes session → scoped student lookup → preview/confirmation → API`.
- **Interfaces/schema:** Define a session-to-actor/class mapping and correlation ID. No student/comment schema change is required.
- **Feasibility:** High, with capacity limits, reconnect behavior, and explicit handling for users who belong to multiple classes.
- **Privacy/security:** Never reuse a group transcript in a private session, do not allow cross-origin session listing for ordinary users, and fail closed when a chat cannot be mapped to an authorized scope.

**Recommendation**

Adopt strict chat and thread isolation before enabling ClassNote in group conversations. Start with direct teacher chats and add groups only with explicit membership and class-scope rules.

### 79. Gateway Streaming

Hermes can progressively edit a platform message as the response is generated, with platform-aware fallback when message editing is unavailable ([configuration reference](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**ClassNote integration analysis**

- **Business value:** Teachers receive quick feedback that a voice or text observation is being processed instead of waiting without status.
- **Extension point:** Stream only non-sensitive progress or a draft preview; keep the final structured result and confirmation as a separate authoritative message.
- **Responsibilities:** Hermes manages transport updates; the integration layer buffers partial model output; ClassNote API is called only after complete validated arguments; the frontend renders pending, confirmed, or failed states.
- **Data flow:** `message → progress stream → complete intent → student lookup → preview stream/final card → confirmation → API write → final status`.
- **Interfaces/schema:** Define message correlation, stream state, final version, and cancellation semantics. No database schema change is needed.
- **Feasibility:** Medium; Telegram editing limits, race conditions, partial Markdown, network retries, and duplicate final messages require testing.
- **Privacy/security:** Do not stream raw student data, credentials, or unvalidated names to a broad group. Redact intermediate tool output and ensure a partial stream cannot be mistaken for a saved comment.

**Recommendation**

Use streaming for short progress indicators and completed previews only. Keep write confirmation and API success as non-streamed, explicit states with delivery deduplication.

### 80. Human Delay

Hermes can add configurable human-like response pacing for messaging platforms, with natural or custom delay modes ([configuration reference](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**ClassNote integration analysis**

- **Business value:** A small pacing delay can make a teacher-facing assistant feel less abrupt and give the system time to combine rapid successive observations.
- **Extension point:** Apply delay only at the messaging presentation layer; do not delay API transactions, audit events, or safety checks.
- **Responsibilities:** Hermes controls outbound pacing; the integration layer debounces and correlates messages; ClassNote API validates each finalized request; the frontend shows an immediate processing state.
- **Data flow:** `teacher message → immediate receipt indicator → bounded intent processing → optional presentation delay → preview or result`.
- **Interfaces/schema:** No database change. Add a correlation ID and debounce window to the channel adapter so delayed replies cannot be attached to the wrong chat or thread.
- **Feasibility:** High, but the delay must stay short and be disabled for confirmations, errors, and urgent operational notifications.
- **Privacy/security:** Delayed messages can arrive after a teacher changes context or loses access. Re-check session scope before delivery, never use delay to hide errors, and avoid batching observations across students without explicit boundaries.

**Recommendation**

Use only a short, optional presentation delay for normal replies. Keep acknowledgements and safety-critical outcomes immediate, and never let pacing alter business ordering or authorization.
