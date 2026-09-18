# Hermes Feature 111: Local Model Runtime

## Source and capability

Source page: Hermes **Local Models** user guide. Hermes can manage a local inference runtime, select a hardware-appropriate model, and route normal agent turns through an OpenAI-compatible local endpoint. The documented value is that prompts and model responses can remain on the operator's machine after the model is downloaded.

## Business value

SchoolRoom handles student names and teacher comments, so a local model can reduce data exposure and keep the natural-language command path usable during an internet outage. It also gives a school operator a predictable cost boundary for routine roster and comment operations.

## Proposed SchoolRoom integration

The extension point is the Hermes model/provider layer, not the SchoolRoom API. Keep Hermes responsible for intent extraction and tool selection. Keep the integration layer responsible for validating the extracted command and calling the two-table backend through typed endpoints.

Responsibilities:

- Hermes: run the configured local model and emit a structured intent.
- Integration layer: enforce an allowlisted operation such as `create_comment`, normalize names and aliases, and reject unsupported actions.
- SchoolRoom API: remain the only component allowed to write students or comments.
- Database: no schema change; the existing student/comment relationship remains authoritative.
- Frontend: show the same review queue and provenance regardless of model provider.

## End-to-end data flow

1. A teacher sends a natural-language message through the gateway.
2. Hermes sends the message to the local model runtime.
3. Hermes returns a typed intent with student reference, comment text, date, and confidence.
4. The integration layer resolves the reference against the roster and applies validation rules.
5. A low-confidence or ambiguous match becomes a review item; a valid request calls the comment endpoint.
6. SchoolRoom persists the comment and the frontend refreshes the review queue.

## Interface and schema changes

No database migration is required. Add a provider capability check and a model-health check to the integration layer. The intent contract should include `operation`, `student_ref`, `comment_text`, `observed_on`, and `confidence`, with no raw database expression.

## Feasibility, dependencies, and risks

Feasible as an operational option, but quality depends on a model that supports reliable tool calls and enough context for the roster. Model download size, memory pressure, slow first-token latency, and stale local model files are the main dependencies. A local model can still make incorrect matches, so it must not bypass validation or review. The deployment must also protect the model directory and disable arbitrary terminal tools for the school profile.

## Recommendation

Offer local inference as a privacy-first profile after the structured intent contract and confidence policy are in place. Do not make it the only supported provider until representative Chinese and English classroom messages pass the same acceptance tests as the primary provider.
