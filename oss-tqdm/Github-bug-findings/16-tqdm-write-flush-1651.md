# Finding: `tqdm.write` lacks a `flush=` argument (feature gap, not a crash bug)

- Issue: #1651 (2025-02-04). Link: https://github.com/tqdm/tqdm/issues/1651
- Status: CONFIRMED missing locally; classified as feature request.

## Observed

```python
from io import StringIO
from tqdm import tqdm
tqdm.write("hi", file=StringIO(), flush=True)
# TypeError: tqdm.write() got an unexpected keyword argument 'flush'
```

## Context

`tqdm/std.py` classmethod `write(cls, s, file=None, end="\n", nolock=False)`
always calls `fp.write(s); fp.write(end)` inside `external_write_mode` and
never flushes explicitly. `print(..., flush=True)` parity would make
`tqdm.write` a true drop-in `print` replacement for redirected output
(see also `DummyTqdmFile` in `tqdm/contrib/__init__.py` and
`examples/redirect_print.py`). Care needed: only flush when requested,
respect streams whose `flush` may be missing/raising, and decide whether
`DummyTqdmFile` should forward flushed partial lines (related open PR
#1855 adds `DummyTqdmFile.flush()` for buffered partials).

## Competition

Open PR #1724 (`Add flush argument to tqdm.write`) already implements
this. Support/review that PR rather than starting a new one.
