# Hermes Feature 122: Turn-Scoped Media Attachments

## Source and capability

Source page: Hermes **Sessions** guide. Images, audio, and documents are handled as inputs for the current turn; Hermes may derive text or a description, while raw binary content is not automatically copied into every future prompt.

## Business value

Teachers can send a photo of handwritten observations, a voice note, or a small document without forcing the SchoolRoom database to become a media archive. Keeping media scoped to the turn reduces storage and privacy exposure.

## Proposed SchoolRoom integration

Use Hermes attachment normalization before intent extraction. The integration layer should accept only derived text plus bounded provenance, and should explicitly decide when an attachment is allowed to produce a comment draft.

Responsibilities:

- Hermes: download, classify, transcribe, or describe the attachment for the active turn.
- Integration layer: validate MIME type, size, retention, and derived-text confidence; remove raw paths from business payloads.
- SchoolRoom API: accept a text intent and optional opaque attachment receipt.
- Database: store no binary data; optionally store a short-lived receipt or hash in an audit store.
- Frontend: show whether a draft came from text, audio transcription, or image/document extraction.

## End-to-end data flow

1. The gateway receives a message and attachment.
2. Hermes derives a bounded text representation for the current turn.
3. The integration layer validates the derived content and strips attachment internals.
4. A structured intent is matched to a student and placed in review.
5. Temporary media is deleted according to retention policy after the result is durable.

## Interface and schema changes

Extend the internal intent envelope with `input_kind`, `derivation_confidence`, and `attachment_receipt`. Keep the public student/comment schema unchanged. Add a cleanup job for temporary artifacts if the gateway does not already guarantee retention.

## Feasibility, dependencies, and risks

Feasible, but OCR and transcription can misread names or dates. Attachments may contain unrelated children, faces, or confidential documents. Enforce content limits, malware scanning, redaction, explicit consent for image use, and review for every attachment-derived write.

## Recommendation

Support attachments as a convenience input, never as direct evidence of an approved comment. Persist derived business text only after normal validation and keep raw media short-lived.
