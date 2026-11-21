# Behavior gap: attached `message/rfc822` email silently flattened in #1890/#1877

- Affected PRs (both open, neither merged):
  https://github.com/microsoft/markitdown/pull/1890 (`016e1d3`),
  https://github.com/microsoft/markitdown/pull/1877 (`8eed589`)
- Parent issue: https://github.com/microsoft/markitdown/issues/89
- Status: CONFIRMED by local execution, direct and routed (2026-11-21).
  Severity: low for a minimal converter; filed to pin the behavior.
- Corpus file: `11_attached_rfc822.eml` — outer mail ("Forwarding
  MARKER11_BODY") with a complete inner `EmailMessage` attached
  (inner From `inner@example.com`, inner Subject, inner body
  `MARKER11_INNER_BODY`).

## Observed (both PRs)

`msg.walk()` descends into the attached message, so body extraction treats
the inner `text/plain` part as another outer body part:

```markdown
# Email Message

**From:** A <a@example.com>
**To:** B <b@example.com>
**Subject:** Fwd MARKER11

## Content

Forwarding MARKER11_BODY

Inner body MARKER11_INNER_BODY
```

- Inner **headers are lost** (no inner From/To/Subject/Date anywhere) and
  there is no heading or separator marking where the attached message
  begins — a forwarded email reads as one continuous body.
- #1890 additionally omits the `message/rfc822` part from `## Attachments`
  (it lists only parts with filenames; attached messages have none), so
  the attachment's existence is invisible except via the inlined text.
- #1877 lists `- unnamed attachment (message/rfc822)` *and* inlines the
  inner body — the attached message is double-represented.
- Neither converter crashes; nested `multipart/*` (mixed > alternative)
  and inline CID images are handled correctly by both, so this is specific
  to walking through `message/rfc822` boundaries.

## Root cause

Body/attachment iteration uses `Message.walk()` / `iter`-style traversal,
which recurses into attached messages. Parts are classified only by
content type and `Content-Disposition`; the inner message's text part has
disposition `inline`/none relative to its parent message, so it passes
the "not an attachment" filter of the outer walk.

## Suggested handling

Minimal fix options for the winning PR, in increasing ambition:
1. Stop the body walk at `message/rfc822` boundaries and list the attached
   message under `## Attachments` (e.g. by inner Subject/filename), or
2. Recurse deliberately: render the attached message as its own headed
   section with inner headers preserved (aligns with the deferred
   recursive-attachment follow-up from #1663).
Either way, commit a fixture pinning chosen behavior so it changes
deliberately, not by `walk()` accident. Not a merge blocker for a minimal
EML converter; see [01-eml-pr-triage-issue-89.md](01-eml-pr-triage-issue-89.md).
