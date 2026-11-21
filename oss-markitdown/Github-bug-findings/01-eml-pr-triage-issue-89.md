# Triage: 7 competing EML converters for markitdown issue #89

- Issue: #89 "EML support" (opened 2024-12-17 by acseven, labels
  `enhancement` + `open for contribution`, still open). Empty body; only
  requirement in-thread: "EML and MSG formats ideally, but at least EML".
- Links: https://github.com/microsoft/markitdown/issues/89
- Status: TRIAGED and verified by local execution (2026-11-21).
  Recommendation: **move #1890 forward**; do not open an 8th PR.
- Full record: [eml-pr-verdict-issue-89.md](eml-pr-verdict-issue-89.md)

## Ranking (verified)

| Rank | PR | One-line verdict |
|---|---|---|
| 1 | [#1890](https://github.com/microsoft/markitdown/pull/1890) shusingh | Best minimal correct converter; move forward |
| 2 | [#1877](https://github.com/microsoft/markitdown/pull/1877) Lubrsy706 | Near-tie; best single test vector, no Bcc, 64 KiB sniff cap |
| 3 | [#1663](https://github.com/microsoft/markitdown/pull/1663) VANDRANKI | Best recursive-attachment design; regex HTML stripping, first-part-only body |
| 4 | [#2206](https://github.com/microsoft/markitdown/pull/2206) saitejabandaru-in | Freshest base; prefers HTML over plain (bug), silent attachment drops, conflicting |
| 5 | [#1633](https://github.com/microsoft/markitdown/pull/1633) mvanhorn | Clean minimal; attachments silently dropped (can leak into body), regex HTML |
| 6 | [#1503](https://github.com/microsoft/markitdown/pull/1503) lfglopes | Most approvals (4, non-maintainer); raw HTML leaked into Markdown, fixture never registered in test vectors, over-broad `message/` accepts |
| 7 | [#1926](https://github.com/microsoft/markitdown/pull/1926) yousrae2004 | Disqualified: bundles unrelated DocConverter (`unword`) + PptxConverter changes, conflicting; inherits #1503 raw-HTML leak |

Notes: no PR has a maintainer review; every PR shows only `license/cla`
as a check run (all "tests pass" claims are author-reported). #1663/#2206
close #1662, not #89. #1503 descends from #271; #1926 from #1503.

## How the top two were verified

Checked out `pr-1890` (`016e1d3`) and `pr-1877` (`8eed589`) in isolated
worktrees; ran an 18-file corpus + 4 negatives directly against each
`EmlConverter` (36 checks) and again through full `MarkItDown()` routing
with real magika (20 checks, by-path + hint-less + each PR's own vectors):

- Direct: #1890 34/36; #1877 32/36 as scored, effectively 34/36 (its two
  title-type failures are cosmetic — the header object passes
  `isinstance(..., str)`, equality, and `json.dumps`).
- Routing: **#1890 17/20, #1877 16/20**; sole delta is Bcc. Output is
  byte-identical on 12/18 files and on every negative/hint-less case.
- Measured #1890 wins: Bcc header; header-only sniff accepts mail whose
  `To:` sits after a 70 KiB header (#1877 caps its sniff read at 65536
  bytes and rejects); no dangling empty `## Content` heading.
- Measured #1877 win: attachment lines carry MIME types and filename-less
  attachments get a placeholder entry (worth porting).
- Shared defects, each filed separately: unknown-charset crash (file 02),
  header-like-`.txt` stealing (file 03), attached-email flattening (file 04).
- Tied and solid on both: plain-over-HTML preference, HTML-only conversion
  via `HtmlConverter` (links/lists/tables preserved, script dropped),
  ISO-8859-1/RFC 2047/UTF-8/Windows-1252/Shift-JIS decoding, nested
  multipart, CID exclusion, RFC 2231 filenames, own vectors 4/4 routed.

## What #1890 still needs before merge (summary)

1. Rebase onto current `main` (currently `behind`).
2. Fix the unknown-charset crash (file 02) + regression fixture.
3. Port #1877's multipart fixture and #1926's edge fixtures (HTML
   expectations corrected — #1926 asserts raw `<html>` in output), plus
   attached-RFC822 pinning test; run vectors and post results.
4. Gate `accepts()` sniffing to hint-less streams (file 03) + negative test.
5. Confirm README "Email messages (EML, MSG)" wording with maintainers.
Optional: #1503's `getaddresses` normalization; #1877's attachment labels.
Deferred: #1663-style recursive attachment conversion as a follow-up.

## Suggested next step

Comment on #1890 with the must-fix list (charset crash + sniff gating +
ported fixtures) and, once it lands, close #1877/#1633/#1503/#2206/#1926 as
superseded, crediting salvaged fixtures/ideas. Nothing of this kind was
posted in this session.
