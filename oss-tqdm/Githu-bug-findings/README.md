# Session summary — tqdm bug findings

Date: 2026-11-20. Checkout: `tqdm/tqdm` at master `9cf5a12`. Python 3.12.16 (macOS).
One file per bug (duplicate issues for the same bug are merged; PRs are
mentioned inside each bug file, not given their own files).

## What happened in this session

1. Codebase tour: `tqdm/std.py` is the core (~1500 lines); `utils.py`,
   `_monitor.py`, `cli.py` support it; notebook/gui/tk/rich/asyncio/keras/dask
   modules and `tqdm/contrib/` are adapters around `std.tqdm`; tests live in
   `tests/` and mirror the modules.
2. Triage of open issues: 463 open in total. Reviewed the newest 100
   (2026-11 to 2023-12), filtered out feature requests, questions, spam and
   environment-only reports, and locally reproduced the bug reports.
3. Deep dive on the reversed-iteration bug (#1484/#1588): found linked PRs,
   proved the root cause, implemented a local fix with strengthened tests.
4. Set up a project-local `.venv` (editable install + pytest 9.2.0,
   pytest-asyncio, pytest-timeout, pytest-cov) because pytest was missing.
5. Committed the reversed fix locally (not pushed, no PR) as `428cef0` on
   branch `fix-reversed-1588`, author
   `abdounesalim450 <359083260+abdounesalim450@users.noreply.github.com>`.
6. Checked for similar PRs before opening one: found 4 open PRs for the
   reversed bug (#1485, #1591, #1711, #1903). #1903 has essentially the same
   fix, so instead of a duplicate PR we posted an approved comment on #1903
   offering stronger test cases:
   https://github.com/tqdm/tqdm/pull/1903#issuecomment-7051352177
7. Re-checked #1563 (DummyTqdmFile bytes) with the same thorough PR search:
   no PR fixes it, but 4 open PRs touch the same class and the semantics are
   undecided. Implemented only the conservative subset (mixed str/bytes
   error + `__del__` robustness), committed locally as `ef84df5` on branch
   `fix-dummytqdmfile-1563`.

Nothing was pushed and no PR was opened. The only public action was the
#1903 comment above. `.venv/` is untracked and was never staged.

## Index of bug files

| File | Bug | Status in this session |
|---|---|---|
| 01-reversed-tqdm-1484-1588.md | `reversed(tqdm(...))` yields forward order | Confirmed; fixed locally |
| 02-dummytqdmfile-bytes-1563.md | `DummyTqdmFile.write` bytes handling | Confirmed (narrowed); conservative fix locally |
| 03-format-interval-nan-inf-1928.md | `format_interval` crashes on NaN/Inf | Confirmed |
| 04-cli-log-invalid-level-1827.md | `python -m tqdm --log` IndexError/AttributeError | Confirmed |
| 05-colour-invalid-hex-1826.md | Invalid hex `colour=` raises ValueError | Confirmed |
| 06-asyncio-as_completed-generator-1811.md | `as_completed` rejects generators | Confirmed |
| 07-callbackiowrapper-memoryview-1845.md | Non-byte memoryview writes undercounted | Confirmed |
| 08-itertools-product-repeat-1858.md | `product(repeat=...)` total/type bugs | Confirmed |
| 09-cli-multibyte-delimiter-1847.md | Multibyte `--delim` split across buffers | Confirmed |
| 10-osc8-hyperlink-width-1856.md | OSC 8 links break width/trim | Confirmed |
| 11-set-description-str-colon-1616.md | `set_description_str()` appends colon | Confirmed |
| 12-notebook-ncols-pixels-1927.md | Notebook `ncols` treated as px | Code-inspected, not run |
| 13-discord-telegram-unboundlocal-1812.md | Connection errors masked by UnboundLocalError | Code-inspected, not run (no `requests`) |
| 14-pandas-empty-total-1810.md | Empty DataFrame / `total=` consumed once | Code-inspected, not run (no `pandas`) |
| 15-process_map-unsized-1801.md | `process_map` ValueError on unsized iterables | Already fixed on master (stale issue) |
| 16-tqdm-write-flush-1651.md | `tqdm.write` lacks `flush=` (feature gap) | Confirmed missing |

## Local scratch files (outside the repo, kept for re-running)

- `/tmp/repro_1588.py` — reversed-bug reproduction (4 FAIL before fix, 0 after)
- `/tmp/probe_reversed_edges.py` — reversed edge cases (restore, close, disabled)
- `/tmp/repro_1563.py` — DummyTqdmFile str/bytes behaviour matrix
- `/tmp/test_reversed_suggest.py` — test snippet suggested on PR #1903

## Key lesson

`closed_by_pull_requests` and narrow search queries under-report competing
PRs. For the reversed bug, an issue-number search found 1 open PR; a plain
`reversed` PR search found 4. Always search PRs by symptom keywords too.
