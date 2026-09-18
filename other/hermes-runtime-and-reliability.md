# Hermes Runtime and Reliability

This file contains the detailed ClassNote integration analyses for Hermes features 91–100. It is part of the public showcase documentation and contains no production credentials or deployment configuration.

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
