# Bug: EML `accepts()` sniffing in #1890/#1877 steals header-like `.txt` files

- Affected PRs (both open, neither merged):
  https://github.com/microsoft/markitdown/pull/1890 (`016e1d3`),
  https://github.com/microsoft/markitdown/pull/1877 (`8eed589`)
- Parent issue: https://github.com/microsoft/markitdown/issues/89
- Status: CONFIRMED end-to-end under full `MarkItDown()` routing
  (2026-11-21, Python 3.12.16, magika 1.0.3). Identical on both branches.
- Corpus file: `neg_fromlike.txt` (generator: `/tmp/eml_test/make_corpus.py`)

## Reproduction

```text
From: someone
Subject: tricky plain text that resembles headers

But this is a .txt stream with no email mime hints MARKERNEG2
```

```python
MarkItDown().convert("neg_fromlike.txt")          # by path, .txt hint present
MarkItDown().convert_stream(BytesIO(data), stream_info=StreamInfo())  # hint-less
```

## Expected vs actual (observed, both entry points, both PRs)

- Expected: converted as plain text (verbatim).
- Actual: converted as email —
  `# Email Message\n\n**From:** someone\n**Subject:** tricky plain text
  that resembles headers\n\n## Content\n\nBut this is a .txt stream…`

Controls that pass on both branches (by path and hint-less): ordinary
`.txt` stays text, `.csv` renders as a Markdown table, `.html` converts as
HTML. `accepts()` always restores stream position. The stealing requires
the file to *begin* with `From:` plus `To:`/`Subject:` header-like lines —
narrow, and arguably email-shaped, but the explicit `.txt`/`text/plain`
hint is ignored, which is the defect.

## Root cause

Both `EmlConverter.accepts()` implementations fall through extension/MIME
misses to content sniffing (parse headers; accept if `From` and
`To`/`Subject` present) regardless of contradicting hints already in
`StreamInfo`. In `_markitdown.py`, converters run in priority order per
stream-info guess: `EmlConverter` registers at specific-format priority
(0.0) while `PlainTextConverter` is generic (10.0), so within the very
first (`.txt`) guess the EML sniff wins before plain text is tried.
#1890 sniffs header-only (no size cap); #1877 sniffs the first 65536 bytes
(separately measured to *miss* real mail whose `To:` sits after a 70 KiB
header — #1890 accepts that case, #1877 rejects it).

## Suggested fix

Gate the sniff fallback to genuinely hint-less streams
(`extension` and `mimetype` both empty), or get an explicit maintainer
decision to keep current behavior. Add negative tests: header-like `.txt`
by path must stay plain text; hint-less header-like text behavior pinned
either way; keep the passing plain/CSV/HTML negatives.

## Suggested next step

Required item for the winning PR (#1890) before merge; see
[01-eml-pr-triage-issue-89.md](01-eml-pr-triage-issue-89.md).
