# Bug: `tqdm.contrib.itertools.product` mishandles `repeat` values

- Issue: #1858 (2026-10-02). Link: https://github.com/tqdm/tqdm/issues/1858
- Status: CONFIRMED (two cases locally; overflow case from the issue).

## Observed locally

```python
from tqdm.contrib.itertools import product
class R:
    def __index__(self): return 2
list(product(range(3), repeat=R()))
# TypeError: unsupported operand type(s) for ** or pow(): 'int' and 'R'
# (stdlib itertools.product accepts __index__ and yields 9 items)

list(product([], repeat=-1))
# ZeroDivisionError: 0.0 cannot be raised to a negative power
# (stdlib raises ValueError: repeat argument cannot be negative)
```

Also reported in the issue (not re-run here): `repeat=numpy.int64(16)` with
`range(16)` infers total `0` via int64 overflow instead of `16**16`; and
`product(unsized, repeat=0)` displays an unknown total although the total
is always 1 (sized inputs correctly show `1/1`).

## Root cause

`tqdm/contrib/itertools.py` `product()` uses `repeat` directly as an
exponent when inferring the total, instead of normalising it through the
integer index protocol (`operator.index`) as `itertools.product` does, and
does not validate negativity before the arithmetic.

## Suggested fix (from the issue)

Normalise with `operator.index(repeat)`, validate `repeat >= 0` with the
stdlib error, infer total `1` for `repeat == 0`, respect explicit `total=`.

## Competition

Open PR #1905 (closed attempt #1859). Do not duplicate.
