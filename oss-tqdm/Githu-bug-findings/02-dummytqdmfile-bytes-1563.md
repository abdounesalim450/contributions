# Bug: `DummyTqdmFile.write` cannot handle `bytes` (narrowed)

- Issue: #1563 (2024-04-07, tqdm 4.66, no comments ever)
- Link: https://github.com/tqdm/tqdm/issues/1563
- Status: CONFIRMED in narrowed form; conservative subset fixed locally.

## Observed behaviour matrix (`/tmp/repro_1563.py`)

```
Dummy(StringIO).write(b'hello\n')  -> TypeError: string argument expected, got 'bytes'  [#1563]
Dummy(BytesIO).write(b'hello\n')   -> OK, b'hello\n'      (bytes branch works for binary)
Dummy(StringIO).write('hello\n')   -> OK
Dummy(BytesIO).write('hello\n')    -> TypeError (same as real BytesIO baseline)
Buffered 'part' then b'ial\n'      -> confusing join TypeError (before fix)
Pending b'pending' on StringIO, del -> "Exception ignored" TypeError, data lost (before fix)
Pending b'pending' on BytesIO, del  -> OK, flushed by __del__
```
Real-stream baselines raise the identical TypeErrors, so bytes-to-text
failing with a newline is arguably correct stream behaviour. The bytes
branch in `tqdm/contrib/__init__.py:24`
(`nl = b"\n" if isinstance(x, bytes) else "\n"`) exists to support wrapping
binary streams (e.g. `sys.stdout.buffer`), which adjacent PRs rely on.

## Indefensible parts (fixed locally)

1. Mixed str/bytes writes: a mismatched *partial* write was silently appended
   to the buffer (corruption); a mismatched newline write failed inside
   `tqdm/std.py:725` `fp.write(s)` with a join error.
2. `__del__` (`tqdm/contrib/__init__.py:34`) caught `(OSError, ValueError)`
   but not `TypeError`, so destructors printed "Exception ignored" and lost
   pending data; it also never cleared the buffer, so a manual+GC `__del__`
   pair could write pending data twice (`pendingpending`, observed).

## Related open PRs (same class, do not fix #1563)

#1919 (`_buf` via `wrapper_setattr`), #1908 (`write()` returns length),
#1855 (adds `flush()`), #1740 (docstrings). Any future PR here will likely
conflict with #1908/#1919/#1855.

## What we did

- Local branch `fix-dummytqdmfile-1563`, commit `ef84df5` (not pushed):
  `write()` rejects str/bytes mixing immediately with
  `TypeError: DummyTqdmFile.write() cannot mix str and bytes inputs; ...`,
  buffer left intact; `__del__` also catches `TypeError` and clears `_buf`
  after the flush attempt (write-exactly-once, destructor never raises).
- Tests in `tests/tests_tqdm.py` (`test_dummy_file_mixed_str_bytes`,
  `test_dummy_file_del_flushes_pending`, `test_dummy_file_del_type_mismatch`):
  5 cases, all failed before the fix, pass after. Full suite (not perf):
  166 passed/11 skipped. flake8 clean on `tqdm/contrib/__init__.py`.
- Deliberately NOT decided: decode bytes via `wrapped.encoding`, pass
  through `wrapped.buffer`, or reject bytes-to-text early. With a newline,
  bytes-to-text still raises the stream's `TypeError`; pending bytes in
  `__del__` on a text stream are still dropped silently (like closed-stream
  data), because writing them requires that maintainer decision.
