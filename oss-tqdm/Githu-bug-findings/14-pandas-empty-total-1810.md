# Bug: `tqdm.pandas` empty DataFrame crash + `total=` applies only once

- Issue: #1810 (2026-09-03). Link: https://github.com/tqdm/tqdm/issues/1810
- Status: CODE-INSPECTED, not run (`pandas` not installed in session env).
  Two distinct defects in one issue report, kept in one file.

## Defect 1 — empty DataFrame raises ZeroDivisionError

In the `inner` closure of `tqdm.pandas` (`tqdm/std.py`, the `pandas()`
integration), non-grouped totals are computed as
`total = df.size // df.shape[axis]`, which divides by zero for zero rows
(`axis=0`) or zero columns (`axis=1`) even though pandas accepts those
frames:

```python
import pandas as pd
from tqdm import tqdm
tqdm.pandas()
pd.DataFrame(columns=["a", "b"]).progress_apply(lambda c: c)
# ZeroDivisionError: integer division or modulo by zero
```

## Defect 2 — configured `total=` is consumed after first use

`tqdm.pandas(total=123)` stores kwargs in a shared dict; the first
`progress_apply` pops `total` from it, so later calls fall back to the
inferred length (issue shows `3/123` first, then `4/4` instead of `4/123`).

## Suggested fix

Guard the division (`shape[axis] or` -> total 0/unknown), and read `total`
without mutating the shared kwargs (copy per call). Related pandas-area
issues seen in triage: #1803 (`GroupBy.progress_map` AttributeError),
#1798 (`DataFrame.progress_map` wrong total) — separate bugs, same file
region.

## Competition

Open PR #1870 (`Fix pandas progress totals for empty frames and repeated
calls`); closed partial attempt #1825. Do not duplicate.
