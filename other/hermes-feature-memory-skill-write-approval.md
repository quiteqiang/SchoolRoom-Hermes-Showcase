# Hermes Feature 118: Memory and Skill Write-Approval Gate

## Source and capability

Source pages: Hermes **Persistent Memory** guide and **Slash Commands** reference. Hermes can stage memory or skill writes for review, show a diff, and allow an operator to approve or reject the change before it affects later sessions.

## Business value

The approval gate is a natural fit for a school setting: teachers can benefit from reusable aliases and phrasing rules without allowing an accidental or sensitive observation to become permanent agent knowledge.

## Proposed SchoolRoom integration

Install the gate at the Hermes-to-integration boundary. The integration layer should submit only sanitized candidate preferences and should reject any candidate containing student performance details, unbounded transcript text, or credentials.

Responsibilities:

- Hermes: stage, diff, approve, reject, and notify about memory/skill changes.
- Integration layer: run a privacy classifier and map approved entries to a narrow policy schema.
- SchoolRoom API: remain unchanged for ordinary student/comment writes.
- Database: no change to the two tables; store policy versions separately if persistence is needed.
- Frontend: optionally show an admin-only “pending agent policy” view.

## End-to-end data flow

1. Hermes observes a repeated correction and proposes a candidate rule.
2. The candidate is staged rather than immediately active.
3. The integration layer scans and summarizes the diff.
4. An authorized operator approves or rejects it.
5. Only an approved, versioned policy is used in future intent extraction.

## Interface and schema changes

Define a policy export/import interface with explicit fields and versioning. Keep the student and comment contracts unchanged. Record reviewer identity in the policy store, not in comment text.

## Feasibility, dependencies, and risks

Feasible and strongly aligned with least privilege. The main risks are approval fatigue, reviewers approving hidden sensitive content, and stale policies surviving roster changes. Require concise diffs, expiration or revalidation, and a one-click rollback.

## Recommendation

Enable this before enabling any persistent SchoolRoom-specific learning. Default to reject or manual review, and allow only alias/matching/style rules with clear provenance.
