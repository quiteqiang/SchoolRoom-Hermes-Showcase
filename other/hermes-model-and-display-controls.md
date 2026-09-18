# Hermes Model and Display Controls

This file contains the detailed ClassNote integration analyses for Hermes features 101–110. It is part of the public showcase documentation and contains no production credentials or deployment configuration.

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

### 106. API Timeouts

Hermes separates socket-read, stale-stream, stale-non-stream, and overall API-call timeouts. Local providers receive different implicit handling because large-context prefill can take longer, while explicit provider or model settings override the defaults ([official configuration guide](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**Concrete ClassNote integration analysis**

- **Capability and business value:** Distinguish a slow but healthy local model from a genuinely stuck request, reducing false failures during voice transcription or natural-language matching.
- **ClassNote extension point:** Coordinate Hermes model timeouts with integration HTTP timeouts and database transaction timeouts so one layer does not abandon work while another continues.
- **Responsibilities:** Hermes bounds provider calls; the integration layer maps timeout categories to retry policy; the ClassNote API owns transaction deadlines and idempotent status checks; the frontend shows processing versus retryable failure.
- **End-to-end flow:** `message/voice → STT or model call → timeout classification → retry/fallback or status lookup → API validation/write → final response`.
- **Interfaces and schema:** Return a stable error envelope with timeout class, retryable flag, request ID, and idempotency key. No new student/comment columns are required.
- **Feasibility and dependencies:** High, but requires measured baselines for local STT/model latency and API response time. Provider-specific timeout support must be tested.
- **Privacy and security risks:** Automatic retries can duplicate a successful but unacknowledged write; long timeouts hold sensitive context in memory; fallback providers may cross data-processing boundaries.
- **Recommendation:** Use longer read timeouts only for explicitly local STT/model paths, keep write transactions short, and make every retry status-aware. Do not enable broad fallback for student content without an approved privacy boundary.

### 107. Reasoning Effort

Hermes exposes a reasoning-effort control from none through higher levels, with session-scoped and per-model overrides. Higher effort can improve complex decisions at the cost of tokens and latency, and Hermes clamps unsupported levels rather than silently escalating them ([official configuration guide](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**Concrete ClassNote integration analysis**

- **Capability and business value:** Use low effort for simple queue lookups and higher effort for ambiguous multi-student references or report planning.
- **ClassNote extension point:** Let the integration layer classify request complexity, but keep the final effort policy in the Hermes profile so user text cannot arbitrarily increase cost or bypass safeguards.
- **Responsibilities:** Hermes selects the supported reasoning level; the integration layer supplies a request class and validates tool arguments; the ClassNote API remains deterministic and authoritative; the frontend indicates review required for ambiguous results.
- **End-to-end flow:** `teacher request → complexity classification → bounded reasoning effort → tool call → API lookup/validation → preview → confirmation`.
- **Interfaces and schema:** Add internal `reasoning_class`, latency budget, and request ID metadata. No student/comment schema change is needed; never persist hidden reasoning content.
- **Feasibility and dependencies:** High, subject to provider support and a cost/latency policy. It improves interpretation quality but cannot repair incomplete student data.
- **Privacy and security risks:** Higher effort may send more context or keep it longer; hidden reasoning must not be exposed as a data source; model effort does not grant authorization.
- **Recommendation:** Default to low or medium for routine observations and escalate only for ambiguity, never for permission. Keep the output contract structured, hide internal reasoning, and require teacher confirmation whenever confidence remains low.

### 108. Runtime Metadata Footer

Hermes can append a small runtime-context footer to the final gateway message, such as model, context occupancy, latency, or a home-relative working context. The footer is opt-in and appears only on the final message, leaving interim updates clean ([official configuration guide](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**Concrete ClassNote integration analysis**

- **Capability and business value:** Give teachers and operators lightweight provenance about how a response was produced without exposing internal prompts or tool arguments.
- **ClassNote extension point:** Add a sanitized final-message footer in the channel adapter, separate from the business result card and API response.
- **Responsibilities:** Hermes formats selected runtime fields; the integration layer removes operational details not suitable for the audience; the ClassNote API returns the authoritative record state; the frontend labels metadata as diagnostic, not business content.
- **End-to-end flow:** `request → Hermes processing → ClassNote API result → verified final message → optional safe runtime footer`.
- **Interfaces and schema:** Define an allowlist of fields such as response latency bucket and request correlation status. No database schema change is needed; do not persist the raw footer with the comment.
- **Feasibility and dependencies:** High. It depends on channel-specific formatting and a clear separation between teacher-facing and operator-facing views.
- **Privacy and security risks:** Model names, working context, or timing can reveal deployment details; latency can be misread as a guarantee; footers can make a draft look more authoritative.
- **Recommendation:** Keep the footer off for ordinary teacher messages and enable only a minimal, non-operational status in operator views. Never include paths, hostnames, provider credentials, raw tool counts, or student identifiers.

### 109. File-Mutation Verifier

Hermes can append an advisory when a file write or patch failed and the target was not later changed successfully. It helps catch cases where an agent summarizes a batch of edits as successful even though one or more mutations did not land ([official configuration guide](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**Concrete ClassNote integration analysis**

- **Capability and business value:** Reduce false completion claims when Hermes edits ClassNote code, documentation, or migration files.
- **ClassNote extension point:** Use the verifier in engineering and maintenance sessions, while using API read-back rather than file mutation checks for student/comment writes.
- **Responsibilities:** Hermes reports failed file mutations; the integration layer maps them to an incomplete change set; CI runs API, database, and frontend checks; the frontend or review workflow blocks publication until evidence exists.
- **End-to-end flow:** `change request → Hermes patch/write → mutation receipt → verifier warning or success → tests/build/read-back → human review → merge`.
- **Interfaces and schema:** Verification output should contain file scope, operation type, failure reason, request ID, and evidence status. No business database schema change is required.
- **Feasibility and dependencies:** High for repository changes. It depends on deterministic patch receipts and an explicit verification command set for each affected layer.
- **Privacy and security risks:** Verification output can expose filenames or fixture data; a file change can still be semantically wrong; a successful patch does not prove a safe migration or authorized data operation.
- **Recommendation:** Enable the verifier for code and configuration maintenance, pair it with tests and clean-diff review, and keep it separate from business-data correctness. For student comments, trust API transaction status and read-back, never filesystem mutation evidence.

### 110. Per-Turn Summary and Spinner Token Flow

Hermes can show a per-turn accounting summary and live output-token flow in interactive CLI surfaces. The summary is derived from observed tool progress, excludes failed calls, and is suppressed for gateway/messaging surfaces that use other display controls ([official configuration guide](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)).

**Concrete ClassNote integration analysis**

- **Capability and business value:** Help operators understand whether a slow ClassNote workflow spent time reading, searching, or running tools without exposing the teacher’s data in the conversation.
- **ClassNote extension point:** Convert the same internal accounting concepts into a redacted operator metric stream; keep teacher-facing Telegram responses concise and business-focused.
- **Responsibilities:** Hermes counts tool activity and tokens in supported surfaces; the integration layer emits safe stage metrics; the ClassNote API exposes request latency and transaction status; the frontend provides an operator diagnostics view separate from teacher comments.
- **End-to-end flow:** `request → tool/model activity → ephemeral accounting events → API result → operator summary and teacher-safe final response`.
- **Interfaces and schema:** Use a metric event with request ID, stage, duration bucket, tool category, and terminal state. Do not store raw prompts, token text, or student details in the student/comment tables.
- **Feasibility and dependencies:** Medium to high. CLI support is straightforward; a useful web/dashboard view requires a metrics transport and retention policy.
- **Privacy and security risks:** Timing and tool categories can reveal sensitive workflow details; token counts may expose usage patterns; detailed summaries in group chats can identify another teacher’s activity.
- **Recommendation:** Keep raw token flow CLI-only and use coarse, redacted metrics for operators. Send teachers only “processing/reviewed/saved” states, and ensure the final business result comes from the API rather than display telemetry.
