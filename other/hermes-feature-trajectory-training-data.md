# Hermes Feature 127: Trajectory and Training-Data Export

## Source and capability

Source page: Hermes **Architecture** guide. Hermes can turn agent sessions into structured trajectories suitable for evaluation or training-data generation. The trajectory captures interaction steps rather than merely the final answer.

## Business value

A sanitized trajectory set could measure whether Hermes correctly resolves student aliases, asks for clarification, and respects review gates. It can improve the integration without exposing production conversations to a model-training workflow.

## Proposed SchoolRoom integration

Export only synthetic or explicitly consented examples from the integration test suite. The exporter should replace student names, identifiers, chat IDs, and free-form observations with placeholders before creating evaluation data.

Responsibilities:

- Hermes: emit the structured turn/tool trajectory.
- Integration layer: redact and label expected intent, match result, and policy outcome.
- SchoolRoom API: provide test fixtures or a read-only export of synthetic cases only.
- Database: no production data export by default; store evaluation metadata separately.
- Frontend: show aggregate accuracy and error categories, not raw transcripts.

## End-to-end data flow

1. A synthetic classroom utterance is sent through Hermes.
2. The trajectory records model output, tool calls, validation, and final result.
3. A redaction pipeline removes identifiers and secrets.
4. An evaluator scores intent extraction, matching, and safety behavior.
5. Only aggregate metrics or approved fixtures return to the showcase repository.

## Interface and schema changes

Define a redacted trajectory schema with `input_template`, `intent`, `expected_policy`, `observed_policy`, and `outcome`. Do not add a production endpoint that dumps complete conversations.

## Feasibility, dependencies, and risks

Feasible for offline evaluation. Student comments are sensitive educational data and may contain identifying details even after simple masking. Prefer synthetic fixtures, deterministic redaction, access review, retention limits, and a deny-by-default export command.

## Recommendation

Adopt trajectory export for tests and regression analysis only. Do not use production student conversations for training unless separate consent, governance, and irreversible de-identification are in place.
