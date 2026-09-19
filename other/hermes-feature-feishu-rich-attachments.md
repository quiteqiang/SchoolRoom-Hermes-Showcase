# Hermes Feature 123: Rich Attachments in Feishu

## Source and capability

Source page: Hermes **Feishu / Lark** messaging guide. The adapter can collect images, files, and rich-text attachments from an inbound message and can send text, native images, documents, audio, video, and other files on the way out.

## Business value

Feishu can be an alternative staff channel for schools that do not want every workflow in Telegram. A teacher could forward a document or voice note and receive a review-ready draft in the same collaboration tool.

## Proposed SchoolRoom integration

Treat Feishu as another Hermes channel adapter feeding the same integration contract. Channel-specific message formatting belongs in Hermes; student matching, review status, and API writes must remain channel-neutral.

Responsibilities:

- Hermes: normalize Feishu posts and attachments, then render acknowledgements and review links.
- Integration layer: convert normalized text/derived media into the same typed intent used by Telegram.
- SchoolRoom API: remain the single write path for comments.
- Database: no channel-specific columns; optional provenance belongs in an audit store.
- Frontend: show channel-neutral comments and a safe source label.

## End-to-end data flow

1. A Feishu message or post arrives with text and attachments.
2. Hermes collects supported files and derives bounded text where appropriate.
3. The integration layer validates class scope, student reference, and content policy.
4. SchoolRoom creates a pending comment.
5. Hermes sends a concise acknowledgement or review result back to the originating chat.

## Interface and schema changes

Add a normalized inbound envelope with `channel`, `message_kind`, `text`, `attachments`, and `reply_target`. Do not add Feishu-specific fields to student/comment tables. The adapter should expose opaque attachment receipts rather than provider URLs.

## Feasibility, dependencies, and risks

Feasible if the school already operates a Feishu bot. Rich posts can contain nested files, unsupported formats, or content from multiple students. Enforce file type/size limits, scope authorization, provider-specific retention, and no automatic approval based on channel trust.

## Recommendation

Implement Feishu as a second adapter only after the channel-neutral envelope is stable. Reuse all matching, review, and privacy controls from the Telegram path.
