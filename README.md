# ClassNote

A sanitized showcase of the ClassNote business architecture, selected business-code modules, and its integration approach for Hermes Agent.

This repository contains no production deployment code, real user data, API keys, credentials, server configuration, database credentials, or local file paths.

## Overview

ClassNote helps teachers turn short classroom observations into reviewable student comments. Hermes interprets the teacher's natural-language request and orchestrates only approved business-safe tools; the ClassNote API remains the boundary for validation and data access.

## Documentation

- [Business and Hermes integration architecture](docs/business-architecture.md)
- [Detailed business flow and Hermes integration notes](other/hermes-integration-notes.md)

## Code snapshot

The public snapshot contains the core backend business modules and teacher-facing frontend workflow pages. Deployment files, environment files, database snapshots, provider adapters, messaging credentials, and local development settings are intentionally excluded.

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

### 81. Smart Approvals

Hermes Smart Approvals evaluates potentially dangerous terminal commands in smart, manual, or disabled modes. Smart mode can auto-approve low-risk commands, deny risky commands, and escalate uncertain cases; the denial circuit breaker stops repeated variations of a denied command ([official configuration guide](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**Concrete ClassNote integration analysis**

- **Capability and business value:** Reduce the chance that a natural-language classroom request can trigger an unsafe host operation while avoiding approval fatigue for routine, read-only maintenance.
- **ClassNote extension point:** Apply the policy to the Hermes profile used for maintenance or diagnostics. The normal teacher profile should expose approved ClassNote business tools and no general terminal tool.
- **Responsibilities:** Hermes evaluates terminal risk and presents approvals; the integration layer separates business actions from host commands; the ClassNote API remains responsible for teacher, class, record, and write authorization; the frontend shows pending approval versus completed business state.
- **End-to-end flow:** `teacher request → Hermes intent classification → approved ClassNote tool or flagged operational command → Smart Approval decision → optional teacher approval → API validation → verified result`.
- **Interfaces and schema:** Tool calls should carry a request ID, actor scope, action type, and idempotency key. No new student or comment columns are needed; an external audit stream may retain approval decisions without storing raw student text.
- **Feasibility and dependencies:** High feasibility because this is primarily a Hermes policy boundary. It depends on separate Hermes profiles/toolsets and a typed ClassNote tool contract; it must not be treated as a replacement for API authorization.
- **Privacy and security risks:** An auxiliary approval model may see command text; an overly broad terminal tool can expose local files or secrets; a false approval could still authorize a harmful operation. Keep terminal disabled for ordinary classroom conversations and deny unrestricted shell patterns.
- **Recommendation:** Enable Smart or Manual Approvals only for an isolated operations profile. Keep teacher-facing comment creation on dedicated API tools, require explicit confirmation for writes, and use API read-back as the final proof of success.

### 82. PII Redaction

Hermes can redact personally identifiable information from gateway context before it reaches the language model. On supported messaging platforms it deterministically hashes user, phone, and chat identifiers while preserving internal routing values; user-chosen names are not automatically changed ([official configuration guide](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**Concrete ClassNote integration analysis**

- **Capability and business value:** Reduce exposure of messaging identifiers in prompts while preserving enough stable identity for group-session separation and audit correlation.
- **ClassNote extension point:** Enable redaction in the Hermes gateway boundary, before system context and routing metadata are assembled for the model.
- **Responsibilities:** Hermes transforms supported identifiers for model context; the integration layer keeps the private-to-redacted mapping in memory or protected gateway state; the ClassNote API authenticates the original actor and receives only the minimum required business identifiers; the frontend displays teacher-safe names and statuses.
- **End-to-end flow:** `Telegram event → private actor/chat identity → redacted Hermes context → intent and tool selection → scoped API request using an internal authorization context → canonical student/comment result → channel response`.
- **Interfaces and schema:** Add a correlation contract that distinguishes `public_actor_ref` from the internal authorization subject. No student or comment schema change is required; never persist the reversible mapping in the two-table business database.
- **Feasibility and dependencies:** High; this is a gateway configuration and adapter-boundary change. It depends on keeping authorization and delivery routing outside the model prompt.
- **Privacy and security risks:** Deterministic hashes still permit repeated-user linkage; names, free-form comments, and attached files may contain PII; a model-generated identifier must never be trusted as an authorization subject.
- **Recommendation:** Enable redaction for all teacher messaging sessions, pass opaque short-lived tool references to Hermes, and let the API resolve the real actor and student IDs from authenticated context. Add explicit redaction tests for group chats, aliases, and voice transcripts.

### 83. STT Vocabulary Hints

Hermes supports an optional `stt.prompt` vocabulary hint for prompt-capable speech-to-text backends. Plugins can extend the base hint through the `pre_transcription` hook, which is useful for proper nouns, aliases, and domain terminology that Whisper-family models may mis-hear ([official configuration guide](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**Concrete ClassNote integration analysis**

- **Capability and business value:** Improve recognition of student names, aliases, class labels, and classroom vocabulary before Hermes interprets a voice observation.
- **ClassNote extension point:** Add a bounded vocabulary provider in the Telegram voice adapter or Hermes plugin layer, generated from the currently authorized class scope.
- **Responsibilities:** Hermes/STT performs transcription; the integration layer selects and limits vocabulary hints; the ClassNote API remains authoritative for student matching and ambiguity handling; the frontend lets the teacher correct a transcript or matched student before saving.
- **End-to-end flow:** `voice message → scoped vocabulary selection → local STT with hints → transcript normalization → Hermes intent extraction → student lookup → teacher preview → API write`.
- **Interfaces and schema:** Define a transient `transcription_context` with locale, allowed aliases, and request ID. Reuse existing comment text and student alias fields; do not add a transcript-only table or persist the full vocabulary in business records.
- **Feasibility and dependencies:** High for local faster-whisper or another prompt-capable backend; medium when the active backend ignores prompts. It depends on class scoping, bounded prompt size, and a fallback path without hints.
- **Privacy and security risks:** A roster-derived prompt can leak names to an external STT provider if the backend is not local; stale aliases can bias recognition; a correct transcript can still map to the wrong student.
- **Recommendation:** Use local STT with short, authorized vocabulary hints, never send the whole school roster, and keep matching plus teacher confirmation in the ClassNote layer. Record only the final corrected text and matched student in the existing comment flow.

### 84. File Read Safety

Hermes limits the amount returned by a single `read_file` call and requires the agent to use `offset` and `limit` for larger content. It also deduplicates unchanged file regions, reducing repeated context injection ([official configuration guide](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**Concrete ClassNote integration analysis**

- **Capability and business value:** Prevent a large export, attachment, or generated report from overwhelming the conversation and make file-backed review predictable.
- **ClassNote extension point:** Use the safety limit at the Hermes file-tool boundary, while keeping ordinary student and comment reads behind paginated ClassNote API tools rather than filesystem access.
- **Responsibilities:** Hermes bounds file reads and asks for ranges; the integration layer maps safe attachment references to permitted content; the ClassNote API controls data scope and pagination; the frontend renders bounded previews with an explicit “load more” action.
- **End-to-end flow:** `teacher asks about attachment → authorization check → bounded file preview → Hermes extracts relevant fields → API validates student/class scope → frontend shows preview or review draft`.
- **Interfaces and schema:** Attachment tools should accept an opaque attachment ID plus `offset` and `limit`, and return a content range, total-size hint, and request ID. No student/comment schema change is needed.
- **Feasibility and dependencies:** High; it requires a read-only attachment adapter and consistent range semantics. It is independent of the two-table model.
- **Privacy and security risks:** Files may contain rosters, unrelated student information, or embedded credentials; path disclosure can expose local layout; repeated reads can bypass a weak quota if each call is not audited.
- **Recommendation:** Keep file access disabled in the basic teacher profile until attachment authorization is available. When enabled, expose only opaque, class-scoped attachments, enforce byte and page quotas, and redact sensitive fields before model context.

### 85. Tool-Result Spillover

Hermes spills oversized tool results to its managed cache instead of silently cutting them off. The model receives a preview and an internal reference that can be read in bounded ranges; MCP results use a tighter spillover threshold ([official configuration guide](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**Concrete ClassNote integration analysis**

- **Capability and business value:** Preserve complete report or roster results without flooding the model context, while making incomplete previews explicit to the orchestration layer.
- **ClassNote extension point:** Prefer API pagination and filtering first; use Hermes spillover only as a safety net for unusually large, authorized exports or multi-student reports.
- **Responsibilities:** Hermes stores and retrieves oversized tool output; the integration layer marks previews as incomplete and requests the next range; the ClassNote API enforces class scope, pagination, and maximum result size; the frontend shows a progress state and a complete-result indicator.
- **End-to-end flow:** `report request → scoped API query → bounded page or spillover preview → Hermes requests additional range → integration reconciles pages → API verifies snapshot/version → frontend renders report`.
- **Interfaces and schema:** Tool responses need `complete`, `next_cursor`, `snapshot_ref`, and `request_id` fields. Keep spillover state outside the student/comment tables; optionally retain only a short-lived job reference.
- **Feasibility and dependencies:** Medium to high. The integration must support pagination, retention cleanup, and safe retrieval from the same session; large reports may also need an asynchronous export path.
- **Privacy and security risks:** Full results written to cache remain sensitive; internal file references must not be exposed to teachers or other chats; stale spillover can outlive authorization.
- **Recommendation:** Cap normal API responses and use spillover only for short-lived, class-scoped report work. Encrypt or isolate the managed cache, expire results quickly, bind retrieval to the originating session, and never treat a preview as a complete roster.

### 86. Context Pressure Warnings

Hermes tracks conversation size relative to the compaction threshold and emits informational or warning notifications. On messaging platforms the notice is plain text, does not modify the message stream, and does not inject extra content into the model context ([official configuration guide](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**Concrete ClassNote integration analysis**

- **Capability and business value:** Warn a teacher before a long-running class conversation loses older context, reducing mistaken student matches and incomplete follow-up comments.
- **ClassNote extension point:** Handle the warning in the Hermes gateway presentation layer and offer a safe reset or summary action; keep canonical student/comment state in the API.
- **Responsibilities:** Hermes measures context pressure and sends the notice; the integration layer suggests a scoped summary or new session; the ClassNote API returns current authoritative data; the frontend distinguishes a context warning from a business error.
- **End-to-end flow:** `long teacher session → pressure threshold reached → gateway notification → teacher chooses summarize/new session → fresh scoped lookup → preview → API write`.
- **Interfaces and schema:** A notification event should contain only level, percentage bucket, session reference, and suggested action. No database change is required; summaries should reference student IDs only through authorized opaque handles.
- **Feasibility and dependencies:** High because the feature is automatic and presentation-only. It depends on a reliable session reset path and on tool calls rereading current API state after compaction.
- **Privacy and security risks:** A warning containing model, session, or working-context details can leak operational information; compression may omit a prior instruction or consent state.
- **Recommendation:** Enable the warning for messaging sessions, keep the copy generic, and force a fresh student lookup plus confirmation after compression. Never use conversation history as the sole source of truth for a write.

### 87. Unauthorized DM Behavior

Hermes can pair unknown direct-message senders, silently ignore them, or politely decline them. The policy can be global or overridden per platform, which lets a deployment choose a safe default for private teacher conversations ([official configuration guide](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**Concrete ClassNote integration analysis**

- **Capability and business value:** Prevent an unknown messaging account from reaching student data or learning whether a particular teacher/class account exists.
- **ClassNote extension point:** Use Hermes DM admission as the first gate, followed by ClassNote account and role authorization before any business tool is exposed.
- **Responsibilities:** Hermes handles the initial pair/ignore/decline behavior; the integration layer maps an approved channel identity to a teacher account; the ClassNote API rechecks role, class membership, and record access; the frontend shows only authorized data.
- **End-to-end flow:** `incoming DM → Hermes sender admission → pairing or decline → approved identity mapping → API authorization → scoped student lookup/comment workflow`.
- **Interfaces and schema:** Maintain an external identity binding with platform, opaque sender reference, teacher account reference, status, and created/expired timestamps. Do not copy messaging IDs into student rows or comment text.
- **Feasibility and dependencies:** High. It requires an allowlist or pairing workflow and a clear account-provisioning owner; the API must reject missing or stale bindings.
- **Privacy and security risks:** Pairing codes can be forwarded; decline messages can reveal that a bot is active; identity changes and shared teacher accounts can create cross-user access.
- **Recommendation:** Use explicit pairing for controlled pilots and a quiet ignore/decline policy for public-facing endpoints. Keep bindings revocable, expire pairing codes, rate-limit attempts, and require API authorization on every read and write.

### 88. Quick Commands

Hermes Quick Commands provide deterministic `exec` commands or aliases that run without invoking the LLM. They work across messaging platforms and are intended for predictable utility actions, with a short execution timeout ([official configuration guide](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**Concrete ClassNote integration analysis**

- **Capability and business value:** Give teachers fast, predictable access to low-risk reads such as a review-queue count, without spending a model turn on a command whose behavior is already known.
- **ClassNote extension point:** Register aliases or wrapper commands at the Hermes gateway boundary that call a fixed, read-only ClassNote status endpoint; do not expose arbitrary shell input.
- **Responsibilities:** Hermes dispatches the deterministic shortcut; the integration wrapper validates the authenticated channel and calls the API; the ClassNote API applies authorization and returns a compact result; the frontend or chat adapter renders a plain status response.
- **End-to-end flow:** `/queue → Hermes quick-command dispatch → fixed integration wrapper → authenticated read-only API call → compact queue summary → teacher response`.
- **Interfaces and schema:** Define a command name, allowed platform, actor scope, timeout, and response schema such as `pending_count` plus `generated_at`. No student/comment schema change is needed.
- **Feasibility and dependencies:** High for read-only summaries. It depends on a small stable endpoint and a wrapper that cannot accept or concatenate user-provided shell fragments.
- **Privacy and security risks:** Exec commands run with the gateway's host privileges; queue output can reveal student information; aliases can collide with built-in commands or be misconfigured.
- **Recommendation:** Add only fixed, read-only shortcuts such as queue health or help. Prefer a dedicated API wrapper over shell commands, enforce channel authorization, return aggregate data by default, and keep all student-specific writes on the normal Hermes confirmation flow.

### 89. Background Sessions

Hermes `/bg` starts an isolated asynchronous agent session while the originating chat remains responsive. The background session inherits the current configuration and delivers a completion or failure message back to the same chat ([messaging gateway guide](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/)).

**Concrete ClassNote integration analysis**

- **Capability and business value:** Let a teacher continue chatting while Hermes prepares a multi-student summary, imports a bounded file, or assembles a draft report.
- **ClassNote extension point:** Route long-running, non-urgent workflows from the gateway to a background task, while keeping single-comment creation synchronous and confirmation-driven.
- **Responsibilities:** Hermes owns isolated session execution and result delivery; the integration layer creates a task envelope and correlates results; the ClassNote API provides snapshot reads and idempotent writes; the frontend shows queued, running, review-required, completed, or failed states.
- **End-to-end flow:** `/bg report request → background session with scoped prompt → paginated API reads → draft artifact/result → teacher notification → explicit review → idempotent API write → final read-back`.
- **Interfaces and schema:** Use a task ID, originating chat reference, actor scope, source snapshot/version, idempotency key, and expiry. Keep task state in Hermes or an external queue; do not add a third business table to the minimal student/comment model.
- **Feasibility and dependencies:** Medium to high. It needs durable task tracking, retry policy, bounded result delivery, and a way to resume or cancel a task without losing authorization context.
- **Privacy and security risks:** Background sessions inherit tools and configuration; a delayed result may arrive after access is revoked; retries can duplicate comments; completion notifications can expose details in a shared chat.
- **Recommendation:** Start with read-only summaries and draft generation. For writes, require a fresh teacher confirmation, enforce API idempotency and version checks, revalidate authorization at commit time, and deliver only a minimal notification before opening the review view.

### 90. Per-Platform Progress Overrides

Hermes supports platform-specific display settings for tool progress, interim assistant messages, and streaming. A platform can be verbose while another stays quiet, and interim messages remain independent from tool-progress bubbles ([official configuration guide](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**Concrete ClassNote integration analysis**

- **Capability and business value:** Match feedback volume to the channel: useful progress in a private teacher chat, concise updates in a shared space, and a clean final result everywhere.
- **ClassNote extension point:** Map Hermes progress events to a safe presentation contract in the gateway adapter; keep the business API call and final state independent from progress rendering.
- **Responsibilities:** Hermes applies per-platform display policy; the integration layer converts tool activity into redacted stage events; the ClassNote API returns canonical pending or saved state; the frontend renders progress, preview, and verified completion separately.
- **End-to-end flow:** `teacher message → Hermes tool-progress event → platform-specific filter → safe “matching/reviewing/saving” update → API result → explicit final status`.
- **Interfaces and schema:** Define progress events with `stage`, `correlation_id`, `safe_summary`, and `terminal_state`. No database change is required; the frontend should treat progress as ephemeral and the API response as authoritative.
- **Feasibility and dependencies:** High. It depends on adapter support for editing or sending interim messages and on a small allowlist of non-sensitive progress phrases.
- **Privacy and security risks:** A progress bubble can be mistaken for a completed write; raw tool arguments can expose names or identifiers; shared chats may show another teacher's processing state.
- **Recommendation:** Use concise, platform-specific progress in private teacher chats and suppress detailed tool output in groups. Allow only redacted stages, attach correlation IDs internally, and send one final message that states whether the ClassNote API actually committed the change.

### 91. Gateway Turn Lease Timeout

Hermes serializes gateway turns by resolved session ID so concurrent routing keys cannot load and write the same transcript. If the lease wait expires, Hermes fails closed and asks the user to resend instead of automatically requeueing a potentially duplicate message ([official configuration guide](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**Concrete ClassNote integration analysis**

- **Capability and business value:** Prevent two rapid teacher messages from racing against one another and creating duplicate or incorrectly ordered comments.
- **ClassNote extension point:** Bind the Hermes session lease to the normalized teacher-and-chat scope before any student lookup or write tool is called.
- **Responsibilities:** Hermes serializes conversation turns; the integration layer carries correlation and idempotency metadata; the ClassNote API performs atomic validation and write decisions; the frontend shows “busy, resend” rather than a false success.
- **End-to-end flow:** `message A → session lease → scoped lookup/write → verified result → lease release; message B waits → timeout or next turn`.
- **Interfaces and schema:** Tool requests require `request_id`, `idempotency_key`, actor scope, and client timestamp. The existing comment record can use its current idempotency or source metadata; no new table is required.
- **Feasibility and dependencies:** High, provided the API write is idempotent and the adapter preserves message ordering. A queue is preferable if the product later requires automatic reordering.
- **Privacy and security risks:** A timeout message may cause a teacher to resend; without idempotency that resend can duplicate data. Do not expose internal lease details or cross-chat identifiers.
- **Recommendation:** Enable a bounded lease wait for every write-capable session, fail closed on contention, and make retry safe through API idempotency plus a final read-back. Keep read-only queue checks available while a write turn is busy.

### 92. Session Stall Watchdog

Hermes can monitor a busy session whose shared activity clock has been idle while an inbound follow-up is waiting. The gateway emits a warning and a one-shot user notification suggesting a session reset; it is a notify-and-recover aid rather than a business-data timeout ([official configuration guide](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**Concrete ClassNote integration analysis**

- **Capability and business value:** Make a stuck Telegram conversation visible before a teacher retries blindly or assumes a comment was saved.
- **ClassNote extension point:** Map the watchdog notification to the integration layer’s pending-request state and provide a retry-safe reset action.
- **Responsibilities:** Hermes detects inactivity; the integration layer records correlation and retry eligibility; the ClassNote API commits atomically and exposes final status; the frontend marks a request as stalled, retryable, or unknown.
- **End-to-end flow:** `teacher request → Hermes tool/model activity stops → watchdog notice → operator or teacher resets session → API status lookup → safe retry or review`.
- **Interfaces and schema:** Add a transient request state machine: `processing`, `stalled`, `retryable`, `committed`, `failed`. Persist only the request ID and idempotency key if operational audit is needed; student/comment tables need no new columns.
- **Feasibility and dependencies:** High for detection and messaging. Reliable recovery depends on an API status endpoint and idempotent retries.
- **Privacy and security risks:** A shared chat may reveal that another request is stalled; a blind retry can duplicate a comment; watchdog logs must not include full student text.
- **Recommendation:** Enable the watchdog for teacher sessions, send generic notices, and require a status/read-back check before retrying any write. Treat the watchdog as an operational signal, never as proof that a database transaction failed.

### 93. Reconnect Attention Escalation

Hermes retries failed messaging-platform connections with capped backoff. It classifies clearly permanent adapter failures and can mark a continuously retrying platform as needing attention, while continuing retries for eventual recovery ([official configuration guide](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**Concrete ClassNote integration analysis**

- **Capability and business value:** Distinguish a temporary channel outage from a persistent integration problem so teachers do not assume their Telegram request reached ClassNote.
- **ClassNote extension point:** Feed gateway connection health into the application’s operational status surface and message-delivery state, without changing student or comment business data.
- **Responsibilities:** Hermes owns adapter retry and fatal-error classification; the integration layer maps health to a channel status; the ClassNote API remains usable through an alternate authenticated frontend; the frontend shows channel unavailable versus API unavailable.
- **End-to-end flow:** `adapter disconnect → Hermes retries/classifies → attention flag → integration health endpoint → frontend warning or alternate entry point → queued request reconciliation after reconnect`.
- **Interfaces and schema:** Define a health payload with platform, status, last-success time bucket, retry state, and safe reason code. Store delivery attempts in the existing delivery mechanism or operational log, not in student/comment rows.
- **Feasibility and dependencies:** High. It requires health polling or event hooks and a reconciliation policy for messages received during an outage.
- **Privacy and security risks:** Raw adapter errors can reveal tokens, account identifiers, or infrastructure details; retry logs can grow indefinitely; a recovered connection must not replay stale writes without idempotency.
- **Recommendation:** Surface only coarse health states to teachers, alert operators on persistent attention, and reconcile any pending messages by request ID after reconnect. Keep credentials and provider errors inside Hermes logs with redaction.

### 94. Gateway Agent Cache

Hermes caches one agent per session so prompt prefixes and the full transcript can be reused. It supports bounds by entry count, idle time, memory pressure, and recent-session protection; evicted sessions reload from durable session state on the next turn ([official configuration guide](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**Concrete ClassNote integration analysis**

- **Capability and business value:** Improve response latency for active teacher chats while preventing a busy multi-class gateway from exhausting memory.
- **ClassNote extension point:** Treat the cached Hermes transcript as a convenience layer only; every business write and important read must still use the current ClassNote API state.
- **Responsibilities:** Hermes manages cache lifecycle and eviction; the integration layer binds session keys to authorized scopes; the ClassNote API remains the source of truth; the frontend handles a cache reload without changing visible business status.
- **End-to-end flow:** `message → cached session lookup → Hermes intent/tool call → API validation → result → session cache update; eviction → durable session reload → fresh scope check`.
- **Interfaces and schema:** Cache entries need an opaque session key, scope fingerprint, last-used time, and transcript version. No student/comment schema change is needed; invalidate or refresh the fingerprint when authorization changes.
- **Feasibility and dependencies:** High, with memory limits chosen from the actual runtime budget. It depends on durable session storage and a scope check after eviction or reconnect.
- **Privacy and security risks:** Cached transcripts contain sensitive observations and tool output; cross-profile key collisions could mix conversations; memory pressure can cause unpredictable latency.
- **Recommendation:** Use bounded caching for responsiveness, isolate caches by profile and chat origin, avoid storing unnecessary student data in prompts, and always reauthorize the API request after a cache hit or reload.

### 95. Verify-on-Stop

Hermes can refuse to accept a final answer after code edits unless the turn has fresh verification evidence such as a test, build, or lint result. The guard is bounded and does not trigger for documentation-only edits ([official configuration guide](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**Concrete ClassNote integration analysis**

- **Capability and business value:** Prevent Hermes from claiming that a backend or frontend change is complete without evidence, which is especially useful when it edits the ClassNote repository.
- **ClassNote extension point:** Apply the guard to engineering or maintenance sessions, not to ordinary teacher conversations that only create comments through API tools.
- **Responsibilities:** Hermes requests fresh verification after code edits; the integration layer defines the minimum checks for API contracts and UI builds; the ClassNote API and database tests prove runtime behavior; the frontend review surface displays verified versus unverified changes.
- **End-to-end flow:** `engineering request → Hermes edits workspace → verifier detects missing evidence → tests/build/read-back → result summary → human review → merge or rollback`.
- **Interfaces and schema:** Define a verification record with request ID, changed areas, command category, exit state, and evidence timestamp. Keep it in code-review or operational metadata; no student/comment schema change is required.
- **Feasibility and dependencies:** High for repository maintenance. It depends on stable test commands and a distinction between documentation changes, code changes, and business-data writes.
- **Privacy and security risks:** Verification commands can print environment values or fixture data; unrestricted shell verification can mutate the database; a passing build does not prove authorization correctness.
- **Recommendation:** Enable this for engineering profiles with a safe verification allowlist. Require API contract tests, migration checks, and read-back tests where relevant; keep it separate from teacher-facing comment entry and never use a green build as proof of a saved comment.

### 96. Auxiliary Models

Hermes routes side tasks such as image analysis, title generation, vision, compression, and approval review through auxiliary model slots. The default can inherit the main model, while each task can be routed independently to control cost, latency, or capability ([official configuration guide](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**Concrete ClassNote integration analysis**

- **Capability and business value:** Use an economical model for transcript cleanup, title generation, or summarization while reserving the primary model for ambiguous student matching and final tool decisions.
- **ClassNote extension point:** Configure auxiliary routing in the Hermes ClassNote profile, with the integration layer labeling auxiliary output as non-authoritative.
- **Responsibilities:** Hermes selects the model for each side task; the integration layer validates and normalizes its output; the ClassNote API performs authoritative matching and writes; the frontend shows whether text is an automatic draft or a teacher-confirmed result.
- **End-to-end flow:** `voice/text input → optional auxiliary normalization or summary → primary Hermes intent extraction → scoped student lookup → teacher review → API commit`.
- **Interfaces and schema:** Auxiliary results should include task type, model-independent confidence or uncertainty, source request ID, and normalized text. Reuse existing comment fields; do not store provider-specific reasoning or credentials.
- **Feasibility and dependencies:** High, especially for title and compression tasks. It is medium for transcript cleanup because model routing must respect local-only processing requirements and language quality.
- **Privacy and security risks:** A lower-cost external model may receive student names or classroom observations; compression can omit consent or scope instructions; model-specific output can bias matching.
- **Recommendation:** Keep speech transcription local when required, use auxiliary models only for bounded non-authoritative transformations, and route student resolution plus writes through the primary contract and teacher review. Document which tasks may leave the local boundary.

### 97. Fast Mode

Hermes can request provider-side fast or priority processing for supported models. It is opt-in and may carry a premium cost; automatic modes can choose fast processing after a configured waiting window ([official configuration guide](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**Concrete ClassNote integration analysis**

- **Capability and business value:** Reduce perceived latency during live classroom use, especially when a teacher is waiting for a student match or a short review preview.
- **ClassNote extension point:** Apply fast mode selectively to interactive, low-payload intent turns; keep batch reports and background work on normal routing.
- **Responsibilities:** Hermes requests the chosen service tier; the integration layer sets a request class and cost policy; the ClassNote API handles the same validation and transaction path regardless of model speed; the frontend shows processing state without promising completion time.
- **End-to-end flow:** `teacher message → request priority classification → Hermes fast or normal model call → validated tool arguments → API lookup/write → final status`.
- **Interfaces and schema:** Add an internal request class such as `interactive`, `batch`, or `background`, plus a correlation ID and latency budget. No student/comment schema change is necessary.
- **Feasibility and dependencies:** Medium; availability and pricing vary by provider, and fast mode does not improve a slow database or network hop. It should be measured against end-to-end latency.
- **Privacy and security risks:** A priority route may use a different provider or data-processing boundary; cost spikes can occur under message bursts; faster output can increase premature or unreviewed writes.
- **Recommendation:** Pilot fast mode only for short interactive drafts, with a spending cap and provider allowlist. Keep API authorization, confirmation, and idempotency identical to normal mode, and measure total request-to-confirmation latency before broad adoption.

### 98. Clarify Timeout

Hermes can wait for a bounded period when it asks the user a clarifying question. The canonical gateway setting controls how long an incomplete request remains open, with an explicit option for unlimited waiting ([official configuration guide](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**Concrete ClassNote integration analysis**

- **Capability and business value:** Handle ambiguous names, classes, dates, or comment intent without guessing, while preventing abandoned Telegram sessions from remaining active forever.
- **ClassNote extension point:** Use clarification before student resolution or any write-capable tool call; keep the pending request correlated with the originating chat and teacher scope.
- **Responsibilities:** Hermes asks and times out the question; the integration layer stores a short-lived pending intent; the ClassNote API validates the completed answer; the frontend displays the missing field and resumes the preview flow.
- **End-to-end flow:** `natural-language request → ambiguity detected → Hermes clarification → teacher answer → merged intent → API lookup/validation → preview → confirmed write`.
- **Interfaces and schema:** Define a pending-intent envelope with request ID, missing field, allowed answer shape, expiry, actor scope, and idempotency key. Store it in session/task state or a short-lived queue, not in student/comment rows.
- **Feasibility and dependencies:** High. It requires deterministic expiry handling and a resume path that rereads current authorization and student data.
- **Privacy and security risks:** The prompt may repeat a student name in a group chat; an expired answer could be applied to a newer request; unlimited pending state can consume memory and keep stale authorization.
- **Recommendation:** Use bounded clarification with concise, privacy-aware choices. Expire stale requests, reject answers without a matching request ID, and require a fresh preview plus API authorization before saving.

### 99. Website Blocklist

Hermes can reject URLs matching configured domain patterns before web-search, extraction, browser, or other URL-access tools execute. Rules can include exact domains, wildcard subdomains, and shared rule files ([official configuration guide](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**Concrete ClassNote integration analysis**

- **Capability and business value:** Prevent an agent with web access from reaching administrative portals, private systems, or unrelated sites during a classroom workflow.
- **ClassNote extension point:** Apply the blocklist to Hermes web and browser tools, while exposing only an allowlisted source set for optional curriculum or policy research.
- **Responsibilities:** Hermes blocks matching URL requests; the integration layer tags the tool call with purpose and class scope; the ClassNote API remains the source for student and comment data; the frontend explains that an external lookup was blocked without revealing rule details.
- **End-to-end flow:** `teacher asks for external context → Hermes chooses web/browser tool → URL policy check → block or fetch → sanitized evidence → draft/preview → ClassNote API write only after review`.
- **Interfaces and schema:** Define a URL-policy result with `allowed`, safe reason code, tool name, and correlation ID. No business schema change is needed; evidence references should be optional and sanitized.
- **Feasibility and dependencies:** High for a deny boundary; medium for an allowlist because school policy and curriculum sources need ownership and maintenance.
- **Privacy and security risks:** Domain patterns can reveal internal topology; URL query strings may contain personal data; blocked access is not a substitute for network egress controls or API authorization.
- **Recommendation:** Start with web/browser tools disabled for student-record sessions. If research is needed, use a reviewed allowlist plus a denylist, strip query parameters before logging, and keep external content informational rather than authoritative for student comments.

### 100. Global Toolset Disable

Hermes supports a global list of disabled toolsets that is applied across the CLI and every gateway platform. A globally disabled toolset stays removed even when a per-platform configuration still lists it ([official configuration guide](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**Concrete ClassNote integration analysis**

- **Capability and business value:** Establish a simple institution-wide safety baseline, such as disabling web, terminal, or memory tools for every teacher-facing Hermes channel.
- **ClassNote extension point:** Use the global policy as a coarse guardrail, then expose only the minimum approved ClassNote business tools through the integration profile.
- **Responsibilities:** Hermes removes disabled capabilities before tool selection; the integration layer publishes the allowed tool contract; the ClassNote API enforces record permissions and transaction rules; the frontend hides controls for unavailable capabilities and provides a clear fallback.
- **End-to-end flow:** `managed policy → Hermes resolved toolsets → natural-language intent → approved ClassNote tool only → API authorization → frontend result`.
- **Interfaces and schema:** Maintain a versioned capability manifest containing toolset name, action, read/write class, and required role. No student/comment schema change is required.
- **Feasibility and dependencies:** High and configuration-driven. It depends on separate teacher and operations profiles if administrators still need terminal, web, or diagnostic capabilities.
- **Privacy and security risks:** Disabling memory can reduce continuity but improve privacy; disabling a needed tool may push users toward unsafe workarounds; policy drift across profiles can create inconsistent behavior.
- **Recommendation:** Disable terminal, web, and unrelated external toolsets in the teacher profile by default. Keep the ClassNote API toolset narrow, version the manifest, and review any capability expansion as a security change rather than a prompt change.

### 101. Context File Truncation

Hermes bounds the size and read time of automatically loaded context files such as project guidance documents. Oversized or slow files are truncated or skipped with a warning so one context source cannot crowd out the rest of the system prompt ([official configuration guide](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**Concrete ClassNote integration analysis**

- **Capability and business value:** Keep the ClassNote orchestration rules available even when a repository contains large guidance files, generated output, or slow mounted storage.
- **ClassNote extension point:** Place the public Hermes integration contract in a small, versioned context file and keep data-specific policy in typed tools rather than a giant prompt.
- **Responsibilities:** Hermes discovers, bounds, and loads context; the integration layer validates the loaded contract version; the ClassNote API remains authoritative for permissions and records; the frontend does not depend on hidden prompt content.
- **End-to-end flow:** `session start → Hermes discovers context files → size/time guard → compact integration rules loaded → natural-language intent → scoped API tool → frontend result`.
- **Interfaces and schema:** Define a compact context manifest with contract version, allowed actions, and escalation rules. No student/comment schema change is needed.
- **Feasibility and dependencies:** High. It requires keeping policy modular and testing behavior when a context file is truncated or unavailable.
- **Privacy and security risks:** A truncated rule file may omit a safety instruction; auto-loaded files can contain private data or hostile instructions; slow mounts can create inconsistent startup behavior.
- **Recommendation:** Keep only non-sensitive, stable orchestration guidance in auto-loaded context. Treat missing or truncated policy as a fail-closed condition for writes, and let the API enforce every rule that affects data access.

### 102. Tool Output Truncation Limits

Hermes applies related caps to raw tool output before it enters the conversation. This limits oversized command, search, and integration responses while allowing the agent to request smaller pages or use spillover when the full result is needed ([official configuration guide](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**Concrete ClassNote integration analysis**

- **Capability and business value:** Prevent a broad student search or report query from silently consuming the context window and degrading later intent extraction.
- **ClassNote extension point:** Return paginated, typed API responses with explicit completeness metadata so Hermes never mistakes a truncated page for the full result.
- **Responsibilities:** Hermes enforces final output caps; the integration layer preserves `complete`, cursor, and snapshot fields; the ClassNote API performs filtering and pagination; the frontend shows partial versus complete results.
- **End-to-end flow:** `teacher query → API filter/page → Hermes output cap → completeness check → next-page request if needed → reconciled preview → confirmed action`.
- **Interfaces and schema:** Tool responses should include `items`, `next_cursor`, `complete`, `snapshot_ref`, and `request_id`. No new business table is required.
- **Feasibility and dependencies:** High if APIs are paginated. It depends on stable snapshot semantics for reports that span multiple pages.
- **Privacy and security risks:** A truncated result can omit a relevant student; raw output may include unnecessary PII; repeated pagination can bypass rate limits if not bounded.
- **Recommendation:** Make completeness a required field in every ClassNote read tool, cap page size server-side, and block writes based on incomplete lookup results. Use aggregate summaries by default and require explicit expansion for student-level details.

### 103. Context Compression

Hermes can summarize older conversation turns when context approaches its limit, using configured compression behavior and an appropriate auxiliary model. The goal is to retain useful state while reducing the amount of raw history sent on later turns ([official configuration guide](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**Concrete ClassNote integration analysis**

- **Capability and business value:** Keep a long classroom conversation usable without repeatedly sending every prior observation, while reducing latency and model cost.
- **ClassNote extension point:** Treat compression as a conversational optimization and provide a compact canonical state summary; never use compressed text as the authority for student identity or saved comments.
- **Responsibilities:** Hermes performs compression; the integration layer supplies stable intent and correlation markers; the ClassNote API resolves current student/comment state; the frontend warns when a resumed conversation needs reconfirmation.
- **End-to-end flow:** `long session → compression threshold → Hermes summary → new teacher message → fresh API lookup → preview with current state → API commit`.
- **Interfaces and schema:** Compression metadata should include session version, summarized turn range, unresolved questions, and pending request ID. No student/comment schema change is required.
- **Feasibility and dependencies:** High, but quality depends on the compressor context window and clear summary invariants.
- **Privacy and security risks:** Compression can omit a consent decision, negation, or student distinction; the summary may contain sensitive observations; an auxiliary provider may see data that was intended to remain local.
- **Recommendation:** Use compression for navigation and drafting only. After compression, force fresh student resolution, scope validation, and teacher confirmation before a write; prefer local or approved processing for sensitive summaries.

### 104. Iteration Budget

Hermes can bound the number of agent/tool iterations in a turn, limiting how long an autonomous loop may continue. The budget is intended to stop runaway tool use while allowing ordinary multi-step workflows to finish ([official configuration guide](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**Concrete ClassNote integration analysis**

- **Capability and business value:** Put a hard ceiling on repeated student searches, clarification loops, or failed write retries so one Telegram message cannot consume unbounded resources.
- **ClassNote extension point:** Set the budget around the integration workflow and return a structured “needs review/retry” state when the ceiling is reached.
- **Responsibilities:** Hermes counts model/tool iterations; the integration layer classifies retryable versus terminal API errors; the ClassNote API enforces idempotency and transaction limits; the frontend shows incomplete processing instead of a fabricated answer.
- **End-to-end flow:** `request → lookup → disambiguation/tool calls → iteration budget reached or success → API commit/read-back → teacher status`.
- **Interfaces and schema:** Tool results need a retryability code, attempt count, request ID, and idempotency key. No student/comment schema change is required.
- **Feasibility and dependencies:** High. The useful budget depends on the number of expected tools and whether the API can combine lookups safely.
- **Privacy and security risks:** A low budget can cause incomplete matching; a high budget increases cost and duplicate-call risk; retry details may expose internal errors.
- **Recommendation:** Use a conservative per-turn budget, make repeated reads batchable, and stop immediately on authorization or validation errors. Require a fresh teacher action after exhaustion and never auto-retry a write without idempotency.

### 105. Wall-Clock Run Budget

Hermes can bound the wall-clock duration of a conversation run independently from the iteration count. At a configured progress point it tells the agent to stop new discovery and produce the deliverable from the state already collected; stale timeouts are scaled to the remaining budget ([official configuration guide](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**Concrete ClassNote integration analysis**

- **Capability and business value:** Keep a teacher-facing request or report job within a predictable response window and avoid spending the entire run on one slow provider call.
- **ClassNote extension point:** Assign different budgets to interactive comment capture and background reporting, while leaving the API transaction path bounded separately.
- **Responsibilities:** Hermes manages the conversation deadline and wrap-up guidance; the integration layer converts timeout into a retry-safe state; the ClassNote API commits atomically; the frontend shows draft, timed-out, or verified states.
- **End-to-end flow:** `request → bounded intent/tool work → budget warning → final preview or timeout → API status/read-back → teacher retry or review`.
- **Interfaces and schema:** Requests need a deadline, request ID, idempotency key, and completion state. No student/comment schema change is required; a job reference can live in task metadata.
- **Feasibility and dependencies:** High for interactive work; medium for long reports because model and API budgets must be coordinated.
- **Privacy and security risks:** A deadline can end after the API committed but before the reply was delivered; retries may duplicate writes; timeout details can expose internal timing or provider behavior.
- **Recommendation:** Use a conservative budget for synchronous comments and a separate asynchronous path for reports. Always query API status after timeout, then retry only with the same idempotency key and a fresh authorization check.

## Showcase scope

This is a reviewable architecture and business-code showcase, not a deployable product or a production Hermes configuration.
