# Bug: EML converters in #1890/#1877 crash on unknown body charset (routed: silent raw-source dump)

- Affected PRs (both open, neither merged):
  https://github.com/microsoft/markitdown/pull/1890 (`016e1d3`),
  https://github.com/microsoft/markitdown/pull/1877 (`8eed589`)
- Parent issue: https://github.com/microsoft/markitdown/issues/89
- Status: CONFIRMED by local execution, direct and through full routing
  (2026-11-21, Python 3.12.16). Must-fix for whichever PR merges.
- Corpus file: `08_badcharset.eml` (generator: `/tmp/eml_test/make_corpus.py`)

## Reproduction

```eml
From: A <a@example.com>
To: B <b@example.com>
Subject: BadCharset MARKER08
MIME-Version: 1.0
Content-Type: text/plain; charset="x-no-such-charset-99"
Content-Transfer-Encoding: base64

<base64 of "Fallback body MARKER08_BODY ...">
```

## Expected vs actual (observed)

| Entry point | Expected | Actual (both PRs, identical) |
|---|---|---|
| `EmlConverter.convert()` directly | Markdown with decoded body (lossy fallback OK) | Raises `LookupError: unknown encoding: x-no-such-charset-99`; no result |
| `MarkItDown().convert("08_badcharset.eml")` | Same Markdown | **No error**, but output is the raw `.eml` source verbatim (headers + still-base64 body `RmFsbGJhY2sg…`), `title=None` |

The routed behavior is the worse failure mode: converter fallthrough
swallows the exception, `PlainTextConverter` wins, and the caller gets
plausible-looking garbage with no signal. All *registered* charsets tested
decode correctly on both branches (ISO-8859-1 QP + RFC 2047, UTF-8 base64,
Windows-1252 8bit, Shift-JIS) — only unknown labels break.

## Root cause

Both converters parse with `policy=policy.default` and read text parts via
`EmailMessage.get_content()`, which raises `LookupError` for unregistered
charset labels instead of falling back. #1890 already contains a
`get_payload(decode=True)` + charset/`errors="replace"` fallback, but only
on its legacy non-`EmailMessage` branch — `policy.default` always produces
`EmailMessage` parts, so that branch is dead and the exception escapes from
the `isinstance(part, EmailMessage)` path. #1877 calls `part.get_content()`
unconditionally with no fallback at all.

## Verified fix path (stdlib only, tested separately)

Wrap the `get_content()` call: on `LookupError`/`UnicodeDecodeError`, use
`part.get_payload(decode=True)` decoded as UTF-8 with `errors="replace"`
(recovers `"Fallback body MARKER08_BODY �� raw"` for the fixture above).
For #1890 this means moving its existing fallback so it also covers the
`EmailMessage` branch in `_get_part_content`.

## Suggested next step

Block merge of the winning EML PR (#1890 per
[01-eml-pr-triage-issue-89.md](01-eml-pr-triage-issue-89.md)) on this fix
plus a committed regression fixture with a bogus charset label asserting
the body marker survives conversion (direct and routed).
