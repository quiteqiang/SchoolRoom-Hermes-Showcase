# Hermes Safety and Channel Controls

This file contains the detailed ClassNote integration analyses for Hermes features 81–90. It is part of the public showcase documentation and contains no production credentials or deployment configuration.

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
