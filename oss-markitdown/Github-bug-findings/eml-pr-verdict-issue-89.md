# EML Support — PR Verdict for Issue #89

Date: 2026-11-21
Repository: [microsoft/markitdown](https://github.com/microsoft/markitdown)
Issue: [#89 "EML support"](https://github.com/microsoft/markitdown/issues/89)

## Verdict

**Move [#1890 "Add EML email converter"](https://github.com/microsoft/markitdown/pull/1890) by shusingh forward.**

It is the best minimal, correct implementation of EML support among the seven open
competing PRs. It should not merge as-is: it needs a rebase onto current `main`,
stronger tests ported from the runner-up PRs, and a fix for the unknown-charset
crash found by local testing (see
[Local test results](#local-test-results-1890-vs-1877) and
[What #1890 still needs](#what-1890-still-needs)).

**Update after local testing (2026-11-21): verdict unchanged, margin narrower.**
#1890 and runner-up #1877 were checked out locally and run against an 18-file
real-world corpus. Their core conversion output is byte-identical on 12 of 18
cases; both share one crash bug. #1890 still wins on measured differentiators
(Bcc header, large-header sniffing, no dangling empty `## Content`), so the
ranking below stands — with the shared crash promoted to a must-fix.

If recursive conversion of email attachments is later declared a requirement,
[#1663](https://github.com/microsoft/markitdown/pull/1663) is the design to build
on as a follow-up — not a reason to block #1890 now.

## Final ranking

| Rank | PR | Author | One-line verdict |
|---|---|---|---|
| 1 | [#1890](https://github.com/microsoft/markitdown/pull/1890) | shusingh | Best minimal correct converter; move forward |
| 2 | [#1877](https://github.com/microsoft/markitdown/pull/1877) | Lubrsy706 | Near-tie; best single test vector, weaker sniffing |
| 3 | [#1663](https://github.com/microsoft/markitdown/pull/1663) | VANDRANKI | Best attachment design; HTML regex flaw |
| 4 | [#2206](https://github.com/microsoft/markitdown/pull/2206) | saitejabandaru-in | Freshest base; prefers HTML over plain (bug) |
| 5 | [#1633](https://github.com/microsoft/markitdown/pull/1633) | mvanhorn | Clean but drops attachments entirely |
| 6 | [#1503](https://github.com/microsoft/markitdown/pull/1503) | lfglopes | Most approvals; leaks raw HTML, fixture untested |
| 7 | [#1926](https://github.com/microsoft/markitdown/pull/1926) | yousrae2004 | Disqualified: bundles unrelated Doc/Pptx changes |

## What the issue actually requires

Issue [#89](https://github.com/microsoft/markitdown/issues/89) (opened 2024-12-17
by acseven, labeled `enhancement` + `open for contribution`, still open) has an
empty body: "No description provided." The only substantive requirement in the
8-comment thread is the author's follow-up:

> "it would be great if the library could process email files, EML and MSG
> formats ideally, but at least EML."

There are no acceptance criteria on headers, HTML vs. plain-text bodies, MIME
parts, attachments, encodings, nested messages, or output format, and no
maintainer has added requirements in the thread. The remaining comments are
status requests and bumps.

Consequences for this evaluation:

- EML-only satisfies the issue ("at least EML"). MSG is already covered
  separately by `OutlookMsgConverter` in the codebase, so no PR is penalized
  for omitting it.
- Attachment *listing* satisfies "process email files" under the minimal
  reading. Recursive attachment *conversion* is an enhancement, not a
  requirement, and is scored as a bonus rather than a blocker.
- All other criteria below are inferred from codebase conventions
  (`OutlookMsgConverter`, `HtmlConverter`, `ZipConverter`, and the
  `_test_vectors.py` test pattern), not from the issue text.

## Evaluation method

For each PR the following were read in full via the GitHub API: PR description,
complete diff (every changed file including test fixtures), reviews, issue
comments, mergeable state / base SHA, and check runs.

Findings that apply to every PR:

- **No maintainer review on any PR.** Only #1503 has reviews at all (four
  approvals, all `author_association: NONE`). All other PRs have zero reviews;
  comments are CLA agreements or author follow-ups.
- **No CI test evidence on any PR.** Every PR shows exactly one check run,
  `license/cla: success`. All "tests pass" claims in PR descriptions are
  author-reported local runs and were treated as unverified.
- **None targets current `main` cleanly.** #2206 and #1926 are in `dirty`
  (conflicting) state; #1890, #1877, #1663, #1633, and #1503 are `behind`
  their bases and need rebasing.
- All seven use only Python's standard `email` module (zero new runtime
  dependencies) — except #1926, whose bundled non-EML changes add an
  `unword` dependency.
- None handles nested `message/rfc822` attached emails or inline CID images;
  this gap is shared and does not affect the ranking.

Ranking criteria, in order: (1) body correctness — `text/plain` preferred,
HTML converted via `HtmlConverter` rather than emitted raw or regex-stripped,
charset-safe decoding; (2) attachment behavior — converted > listed > dropped;
(3) `accepts()` safety and fit with existing converter patterns; (4) test
coverage in the project's `_test_vectors.py` style; (5) scope cleanliness;
(6) mergeability and author responsiveness.

## Evidence per PR

### 1. [#1890](https://github.com/microsoft/markitdown/pull/1890) — shusingh — MOVE FORWARD

Six files, +184 lines, one commit, base `a51f725`, state `behind`.

- Parses with `BytesParser(policy=policy.default)`, so encoded-word headers
  and charsets are handled by the stdlib EmailMessage API.
- Prefers `text/plain`; HTML fallback goes through
  `HtmlConverter().convert_string()`, preserving links, lists, and tables as
  Markdown — the same path the rest of the codebase uses for HTML.
- Extracts the fullest header set among the minimal PRs:
  From, To, Cc, Bcc, Date, Subject; sets result `title` to the stringified
  Subject. Output format (`# Email Message`, `## Content`) matches
  `OutlookMsgConverter`.
- `accepts()` checks extension and `message/rfc822`, then falls back to a
  header-only sniff (`parse(..., headersonly=True)`, requires From plus
  To/Cc or Subject, always seeks back). This mirrors
  `OutlookMsgConverter`'s brute-force OLE sniffing precedent and lets
  hint-less streams still convert.
- Body-part iteration skips `Content-Disposition: attachment` parts, so
  attachments cannot bleed into the body; attachments are listed by name
  under `## Attachments`.
- Includes a charset fallback (`get_payload(decode=True)` + declared charset
  + `errors="replace"`) for legacy `Message` parts that are not
  `EmailMessage` instances.
- Updates `README.md` supported-formats list. Author has followed up twice
  (2026-06-21, 2026-07-01) offering to rebase and address feedback — the
  only author demonstrably still engaged.

Weaknesses (all fixable, none architectural): single plain-text test vector;
attachment behavior is listing-only. Both are covered in
[What #1890 still needs](#what-1890-still-needs).

### 2. [#1877](https://github.com/microsoft/markitdown/pull/1877) — Lubrsy706

Five files, +188 lines, one commit, base `a51f725` (same as #1890), state
`behind`.

- Also correct on fundamentals: `policy.default`, plain preferred, HTML via
  a cached `HtmlConverter` instance, attachments listed with content type.
- Best single test vector of the minimal PRs: a `multipart/mixed` fixture
  asserting the plain body is preferred over the HTML alternative and that
  attachment content is *not* inlined (`must_not_include`). This is the
  fixture to port into the winning PR.
- Ranked below #1890 because: `accepts()` sniffs by reading and fully
  parsing up to 64 KiB (heavier and more false-positive-prone than a
  header-only parse, and will raise nothing on truncated MIME); it converts
  HTML parts even when a plain body exists and discards the result (wasted
  work); it omits Bcc; no README update; no author follow-up since
  2026-05-15. (Local testing downgraded one earlier concern:
  `title=message.get("subject")` returns a
  `_UniqueUnstructuredHeader`, but that type passes `isinstance(..., str)`,
  equality, f-strings, and `json.dumps` — a typing wart, not a measured
  functional break. See [Local test results](#local-test-results-1890-vs-1877).)

Disposition: close as superseded by #1890 after its multipart fixture is
ported, with thanks.

### 3. [#1663](https://github.com/microsoft/markitdown/pull/1663) — VANDRANKI

Four files, +458 lines, one commit, base `604bba1`, state `behind`.
Closes #1662 rather than #89.

- Best attachment design by a wide margin: optional injected `MarkItDown`
  instance following the `ZipConverter` pattern; each attachment is run back
  through `convert_stream()` and rendered under `### <filename>`. With no
  instance, attachments are still listed with an explicit explanatory note —
  nothing is silently dropped. Unsupported/failed attachments produce
  per-file notes instead of aborting conversion.
- Largest dedicated test suite (`tests/test_eml_converter.py`): headers,
  title/empty-subject, missing-Cc suppression, `accepts()` accept/reject,
  no-attachment section suppression, mock-based attachment conversion,
  unsupported-format note, multiple attachments, body/attachment isolation.
- Ranked third despite this because of two correctness flaws in the core
  path: (a) the HTML fallback is a hand-rolled regex tag-stripper
  (`_strip_html`) that loses links, tables, and list structure instead of
  using `HtmlConverter`; (b) `_get_body()` keeps only the *first*
  `text/plain` or `text/html` part, truncating multi-part bodies. It also
  detects attachments by substring-matching the raw `Content-Disposition`
  header rather than `get_content_disposition()`, registers no
  `_test_vectors.py` entry, and does not update the README.

Disposition: keep as the reference design for a follow-up
"recursive attachment conversion" PR built on top of merged #1890
(replacing the regex stripper with `HtmlConverter` and the first-part-only
body logic). Do not merge standalone ahead of #1890.

### 4. [#2206](https://github.com/microsoft/markitdown/pull/2206) — saitejabandaru-in

Four files, +173 lines, one commit, base `e144e0a` (freshest), state
`dirty` (conflicting). Closes #1662.

- Recursive attachment conversion and HTML routed through the `MarkItDown`
  pipeline are the right instincts, and it has one real end-to-end test
  (appended to `test_module_misc.py`) covering headers, body, and a `.txt`
  attachment.
- Core correctness bug: it prefers HTML over plain text (`if body_html:`
  is checked first), inverting the convention every other PR and email
  practice follows. For typical `multipart/alternative` mail the
  HTML-derived rendering replaces the authoritative plain body.
- All body-part and attachment exceptions are swallowed with bare
  `except Exception: pass`, and unconvertible attachments vanish without
  even a filename listing. `markitdown` is a mandatory constructor
  argument (no listing-only fallback like #1663). Only four headers
  (From/To/Subject/Date; no Cc/Bcc). Despite the freshest base it is
  currently conflicting.

Disposition: close. Its attachment test shape (build an `EmailMessage` in
code, convert via `MarkItDown().convert_stream`) is worth reusing in the
follow-up attachment work.

### 5. [#1633](https://github.com/microsoft/markitdown/pull/1633) — mvanhorn

Six files, +205 lines, one commit, base `a6c8ac4`, state `behind`.
Discloses AI assistance in the PR body.

- Clean, readable, honestly scoped as headers-plus-body; two registered
  test vectors (plain, and `multipart/alternative`) with `must_not_include`
  assertions against raw HTML tags.
- Attachments are ignored completely — not converted, not listed — so an
  email whose payload is its attachment converts to headers and a cover
  note. Worse, `_get_body()` walks multipart parts without skipping
  attachments, so a `text/plain` attachment appearing before the body can
  be selected *as* the body, and only the first matching part is kept.
- HTML fallback is the same regex tag-stripper as #1663 (the two share
  near-identical `_get_body` / `_decode_part` / `_strip_html` lineage).
  No README update.

Disposition: close as superseded. Nothing unique to salvage beyond what
#1877's fixture already covers.

### 6. [#1503](https://github.com/microsoft/markitdown/pull/1503) — lfglopes

Four files, +189 lines, five commits, oldest base `dde250a`, state
`behind`. Lineage: updated version of
[#271](https://github.com/microsoft/markitdown/pull/271) by 0xRaduan.

- Genuine strengths: best address-header formatting in the field
  (`email.utils.getaddresses` over `get_all()`, correctly handling
  multi-address From/To/Cc/Bcc), attachments listed with MIME type and
  byte size, plain-text preference logic is correct, and it carries the
  only social proof in the set — four approvals including original author
  0xRaduan ("lgtm, but… would be great to get eyes from someone at MSFT")
  and two independent "tested it, works well" comments from lemig.
- Disqualifying correctness flaw for the win: when no `text/plain` part
  exists, the `text/html` part's content is appended **verbatim** — raw
  `<html>` markup lands in the Markdown output. That is worse than regex
  stripping and much worse than `HtmlConverter`.
- `ACCEPTED_MIME_TYPE_PREFIXES = ["message/"]` claims every `message/*`
  type (delivery-status, disposition-notification, etc.), not just
  `message/rfc822`.
- Its fixture `tests/test_files/test.eml` is added but never registered in
  `_test_vectors.py`, so no committed test exercises this converter at all.
  Output headings (`## Email Headers` / `## Email Content`) also deviate
  from the `OutlookMsgConverter` convention (`# Email Message` /
  `## Content`) shared by the top-ranked PRs.

Disposition: close with thanks once #1890 lands. The `getaddresses` header
formatting is the one idea worth cherry-picking into #1890 if maintainers
want normalized multi-address headers (optional; see below).

### 7. [#1926](https://github.com/microsoft/markitdown/pull/1926) — yousrae2004

Thirteen files, +396/−3, five commits, base `e144e0a`, state `dirty`.
Builds on the #271 → #1503 lineage.

- Disqualified on scope before code review: the branch also adds an
  entire `DocConverter` for legacy `.doc` files (new `unword`
  dependency) and rewrites `PptxConverter` image handling to write image
  files to disk. Neither belongs in an EML PR; the branch cannot merge
  as-is and currently conflicts.
- The EML converter itself is #1503's code, inheriting the raw-HTML leak
  and the over-broad `message/` prefix — and its HTML-only test vector
  asserts `"<html>"` in `must_include`, codifying the leak as expected
  behavior.
- What *is* valuable: the widest edge-case fixture set in the field —
  `test_email_plain.eml` (no attachments → no `## Attachments`),
  `test_email_no_body.eml`, `test_email_html.eml` (HTML-only), and
  `test_email_nosubject.eml` (no `**Subject:**` line) — plus a README
  entry. These fixtures should be ported to #1890 with corrected
  expectations.

Disposition: close. Ask the author to split the Doc/Pptx work into its own
PRs if still wanted; salvage only the EML edge-case fixtures.

## Local test results: #1890 vs #1877

Both branches were fetched (`pr-1890` at `016e1d3`, `pr-1877` at `8eed589`)
into isolated git worktrees and run in a project `.venv` (Python 3.12)
against a purpose-built corpus of 18 `.eml` files plus 4 non-email
negatives, generated with the stdlib `email` package and hand-written raw
messages:

- plain, `multipart/alternative` (plain+HTML), HTML-only (link/bold/list/
  table/script), `multipart/mixed` with text/CSV/PDF attachments,
  ISO-8859-1 quoted-printable with RFC 2047 encoded-word headers, UTF-8
  base64, Windows-1252 8bit, Shift-JIS base64, unknown charset
  (`x-no-such-charset-99`), nested mixed>alternative+attachment,
  `multipart/related` inline CID image, attached `message/rfc822`,
  no-subject, no-body, folded multi-address headers, empty file,
  garbage text with `.eml` name, non-ASCII (RFC 2231) attachment
  filename; negatives: plain `.txt`, `.csv`, `.html`, and a header-like
  `.txt` (`From:`/`Subject:` first lines).

**Scope of execution — caveat closed.** Testing ran in two phases.
Phase 1 (direct): each branch's real `EmlConverter.convert()` and
`accepts()`, with the real `HtmlConverter`. Phase 2 (routing, after the
22.5 MB `onnxruntime` wheel needed by `magika` 1.0.3 was fetched by
parallel range download and installed): full `MarkItDown()` routing —
`convert(path)` on all 18 corpus files, `convert_stream()` hint-less and
hinted, all 4 negatives by path *and* hint-less, and each PR's own
committed test vectors replayed through the four local vector tests
(guess, local, stream-with-hints, stream-without-hints). No stubs were
used in phase 2. Harness and corpus generators were kept outside the
repo (`/tmp/eml_test/`); the repo gained only this file, a `.venv/`,
and two local branches/worktrees.

### Scores

36 checks (marker-based content assertions, crash checks, `accepts()`
behavior, stream-position restoration):

- **#1890: 34/36.** Failures: unknown-charset crash (1 check; a second
  "all titles" failure is the same crash counted again via its missing
  title).
- **#1877: 32/36 as scored, effectively 34/36.** Failures: Bcc header
  missing, same unknown-charset crash, and 2 title-*type* checks. The
  title checks used exact type names; follow-up probing showed the
  returned `_UniqueUnstructuredHeader` passes `isinstance(..., str)`,
  `==` against the subject string, f-strings, and `json.dumps`, so
  those two failures are cosmetic. Adjusted, the PRs tie on the
  checklist — the differentiators are the qualitative items below.

Output diff: markdown is **byte-identical on 12 of 18** corpus files
(plain, alternative, HTML-only, all successful encoding cases, related,
no-subject, folded, garbage). The 6 differing files differ only in
attachment line format, empty-section handling, Bcc, and attached-
message listing (below). Each PR also converts the other's fixture
correctly except for #1877's literal `- notes.txt (text/plain)`
attachment-format expectation.

**Full-routing scores (phase 2, 20 checks): #1890 17/20, #1877 16/20.**
The only scoring difference is Bcc. Shared failures: unknown-charset
recovery (see finding 1 — through routing this degrades to a raw-source
dump instead of an exception) and the header-like-`.txt` stealing test
by path and hint-less (finding 9). Everything else routed correctly on
both branches, including: hint-less streams converting as email with
correct titles, magika stream-info guessing (`message/rfc822`/`.eml`/
`utf-8` exact on both committed vectors), plain/CSV/HTML negatives
never routed as email with content preserved (CSV renders as a table),
and each PR passing its own committed vectors 4/4 (guess, local,
stream-with-hints, stream-without-hints). Routed output between the
branches is identical on every negative and every hint-less case; by
path it differs on the same 6 files as phase 1. One routing-only
observation: the garbage-text file *named* `.eml` converts as an email
(`# Email Message` with the text as body) on both branches — extension
wins, which is acceptable, but it means `accepts()` content checks
never guard the `.eml`-extension path.

### Measured findings that change this verdict

1. **NEW must-fix, shared: unknown charset crashes conversion.** A body
   declared `charset="x-no-such-charset-99"` makes both converters raise
   `LookupError: unknown encoding: x-no-such-charset-99` out of
   `EmailMessage.get_content()`. Called directly, the whole conversion
   fails; through full `MarkItDown` routing (phase 2, measured) the
   failure is arguably worse: the exception is swallowed by converter
   fallthrough, `PlainTextConverter` wins, and the caller silently gets
   the **raw `.eml` source** — headers and still-base64-encoded body,
   `title=None` — with no error signal. Real mail with broken/vendor
   charset labels will hit this. Verified fix
   path (stdlib only, tested separately): on `LookupError`/
   `UnicodeDecodeError`, fall back to `get_payload(decode=True)` decoded
   as UTF-8 with `errors="replace"`, which recovers the body text.
   #1890 already contains such a fallback but only on its legacy-
   `Message` branch, which `policy.default` parts never take — it must
   wrap the `EmailMessage.get_content()` call as well. Neither PR
   description nor fixture covers this case. This is now required item 2
   in the checklist below.
2. **Confirmed: large-header sniffing.** With `To:` placed after a
   70 KiB header value and no hints, #1890 `accepts()` returns `True`
   (header-only parse) while #1877 returns `False` (reads only the
   first 65536 bytes, so `To:` is never seen). The code-reading claim in
   the first draft is now measured.
3. **Confirmed: Bcc.** #1890 renders `**Bcc:** dave@example.com`;
   #1877 drops the header entirely, as predicted.
4. **New small win for #1890: empty sections.** On no-body and empty
   inputs #1890 omits `## Content` (`'# Email Message'` alone for the
   empty file); #1877 emits a dangling `## Content` heading with
   nothing under it on every input.
5. **New small win for #1877: attachment labels.** #1877 lists
   `- doc04.pdf (application/pdf)` (name + MIME type) and lists an
   unnamed attached message as `- unnamed attachment (message/rfc822)`;
   #1890 lists bare filenames and omits filename-less attachments from
   the list. Worth porting the `(content_type)` suffix into #1890
   (checklist item 7); not ranking-changing.
6. **Shared flaw: attached `message/rfc822` is silently flattened.**
   Both converters walk *into* the attached message, appending the
   inner text body to the outer `## Content` with no heading and losing
   all inner headers (From/Subject). #1877 additionally lists it as an
   unnamed attachment, double-representing it. Acceptable for a minimal
   converter, but it should be pinned by a test, and the deferred
   nested-message item in the checklist now describes observed behavior
   rather than a guess.
7. **Downgraded: #1877 title type.** Measured `type(title)` is
   `email.headerregistry._UniqueUnstructuredHeader` on every success,
   but `isinstance(title, str)` is `True` and JSON/f-string/equality
   all behave. The first draft listed this as a defect; it is a
   static-typing wart only. It no longer counts against #1877 beyond a
   one-line `str(...)` cleanup.
8. **Confirmed with nuance: encodings are otherwise solid.** Both
   converters correctly decoded ISO-8859-1 quoted-printable
   (`Café`, RFC 2047 `Jürgen` From and encoded Subject), UTF-8 base64,
   Windows-1252 smart quotes/€, and Shift-JIS — i.e. everything
   `policy.default` handles. The *only* encoding failure is the
   unknown-charset crash in item 1.
9. **Sniffing steals header-like text files (both PRs, CONFIRMED
   end-to-end).** Converter level: `accepts()` returns `True` for the
   header-like `.txt` even when `StreamInfo` says `.txt`/`text/plain`,
   while correctly rejecting plain text, CSV, and HTML negatives and
   always restoring stream position. Full routing (phase 2): the same
   file converts as `# Email Message` with `**From:** someone` /
   `**Subject:** tricky plain text…` headers extracted — **both by
   path (`convert("neg_fromlike.txt")`) and hint-less** — on both
   branches identically, because `EmlConverter` (specific-format
   priority 0.0) is tried before `PlainTextConverter` (generic 10.0)
   and its sniff ignores the contradicting `.txt` hint. All other
   negatives routed correctly (plain text stays text, CSV renders as
   a table, HTML converts as HTML) by path and hint-less. Impact is
   narrow — a text file beginning with `From:` plus `To:`/`Subject:`
   lines is arguably an email — but it is now observed behavior, not
   inference: gating the sniff to hint-less streams remains required
   item 4.
10. **Everything else tied.** Plain-preferred-over-HTML, HTML-only
    conversion quality (headings, `**bold**`, `[text](url)` links,
    lists, tables, `<script>` content dropped), attachment/body
    isolation (attachment bytes never appear in `## Content`), nested
    multipart, inline CID image exclusion, folded headers, RFC 2231
    filenames (`résumé 14.txt`), and no-crash handling of empty/garbage
    inputs are identical in behavior on both branches.

### Effect on the verdict

**Ranking unchanged: #1890 first, #1877 second.** Testing removed one
argument against #1877 (title type), added one small argument for it
(attachment MIME labels), and confirmed three arguments for #1890
(Bcc, large-header sniffing, empty-section cleanliness) with
measurements instead of code reading. Most importantly, both finalists
share a crash on unknown-charset mail that code review alone had
missed in #1890 (its fallback is on a dead branch; routed, it silently
degrades to a raw-source dump) — the winner must fix it before merge,
which slightly narrows, but does not erase, #1890's lead. Full routing
added no new differentiator between the finalists (17/20 vs 16/20,
sole delta Bcc) and confirmed the sniff-stealing risk on both. No result here rehabilitates #1663–#1926: their
disqualifying flaws (raw/regex HTML, HTML-over-plain, dropped
attachments, polluted branch) are code-evident and unaffected by these
runs, though they were not re-tested locally.

## What #1890 still needs

Required before merge:

1. **Rebase onto current `main`** (branch is `behind` base `a51f725`) and
   resolve any registration-order conflicts in `_markitdown.py` /
   `converters/__init__.py`.
2. **Fix the unknown-charset crash (found by local testing).** Wrap the
   `EmailMessage.get_content()` call in `_get_part_content` so
   `LookupError`/`UnicodeDecodeError` fall through to the existing
   payload-decode path (declared charset, then UTF-8 with
   `errors="replace"`). Today a body labeled with an unregistered
   charset raises `LookupError` and fails the entire conversion,
   because the fallback only runs for non-`EmailMessage` parts, which
   `policy.default` never produces. Add a regression fixture with a
   bogus charset label asserting the body text survives.
3. **Port the stronger tests** into `packages/markitdown/tests/_test_vectors.py`
   with fixtures under `tests/test_files/`:
   - from #1877: `multipart/mixed` vector — plain preferred over HTML
     alternative, attachment listed, attachment content not inlined;
   - from #1926 (expectations corrected): HTML-only body (assert converted
     Markdown text present and raw tags such as `<html>`/`<strong>`
     absent — the opposite of #1926's current assertion), no-body email,
     no-subject email (no `**Subject:**` line, `title is None`), and
     no-attachment email (no `## Attachments` section);
   - keep #1890's existing plain vector and its `must_not_include` checks
     for leaked `Content-Type:` / `MIME-Version:` headers;
   - add the unknown-charset regression fixture from item 2 and one
     attached-`message/rfc822` fixture pinning today's observed
     flattening behavior (inner body inlined, inner headers dropped),
     so the deferred follow-up changes it deliberately;
   - run the vectors and report results on the PR, e.g.
     `python -m pytest packages/markitdown/tests/test_module_vectors.py -q`
     — current PRs show no CI test runs, so this replaces author
     assertion with recorded output. Include the stream-without-hints
     path, since the header-sniffing `accepts()` fallback is otherwise
     unexercised. (For reference, both finalists' existing vectors pass
     all four local vector tests under full routing as measured for
     this verdict; the owed run is the rebased PR's own CI/local
     confirmation, including the new fixtures above.)
4. **Gate sniffing so it cannot steal non-email files**: local testing
   showed `accepts()` claims a header-like `.txt`/`text/plain` stream,
   and at specific-format priority it would beat `PlainTextConverter`
   for such files. Restrict the header-sniff fallback to streams with
   no extension and no MIME type (or an explicit maintainer decision to
   keep current behavior), keep the verified rejections of plain text /
   CSV / HTML negatives, and add one negative unit assertion. Stream
   position restoration already passes and must be preserved.
5. **Fix EML wording in the README line**: it currently reads
   "Email messages (EML, MSG)" — accurate for the library as a whole since
   MSG already ships, but confirm maintainers want the combined entry
   rather than an EML-only addition.

Recommended but optional:

6. Normalize multi-valued address headers with `getaddresses` (from
   #1503) so folded/multi-address To/Cc headers render consistently.
   (Measured today: folded headers already render completely via
   `policy.default`; this is normalization polish only.)
7. Port #1877's attachment line format (`- name (content/type)`,
   including a placeholder for filename-less attachments) — measured
   as strictly more informative than bare names — and pin the
   top-level non-text single-part behavior (headers only) with a
   one-line test.

Explicitly deferred (not blockers):

8. Recursive attachment conversion per the #1663 design (optional
   `MarkItDown` injection, per-attachment failure notes), proper
   nested `message/rfc822` rendering (today measured as silent
   flattening: inner body appended to outer content, inner headers
   lost), and inline CID images (today correctly excluded from the
   body). File as a follow-up issue referencing #1663 and #2206 once
   #1890 merges.

## Suggested disposition of the losing PRs

Once #1890 is updated per the checklist above and merged:

- Close #1877, #1633, #1503, #2206, and #1926 as superseded by #1890,
  linking this verdict and crediting the salvaged pieces (#1877 fixture,
  #1926 edge fixtures, #1503 header formatting idea, #271 lineage).
- Keep #1663 open only if its author wants to rebase it into the
  follow-up attachment-conversion PR on top of merged #1890; otherwise
  close it in favor of a fresh follow-up issue.
- Close issue #89 (and note #1662) from the #1890 merge commit.

## Bottom line

Issue #89 asks only that markitdown "process email files… at least EML."
[#1890](https://github.com/microsoft/markitdown/pull/1890) does exactly
that with the fewest correctness compromises: stdlib parsing, plain-text
preference, real HTML-to-Markdown conversion, complete headers, safe
attachment listing, and an `accepts()` fallback consistent with
`OutlookMsgConverter` precedent — all now confirmed by local execution
against multipart, attachment, encoding, and HTML-only mail. Its gaps
are the unknown-charset crash (shared with #1877, one wrapped call to
fix), test breadth, sniff gating, and branch freshness. Every other candidate either breaks body
correctness (raw HTML in #1503/#1926, HTML-over-plain in #2206,
regex-stripped HTML in #1663/#1633), loses attachments silently (#1633,
#2206), or cannot merge as one change (#1926). Move #1890 forward.
