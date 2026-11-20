# Bug: `format_interval()` crashes on non-finite floats (NaN, Inf)

- Issue: #1928 (2026-11-19). Link: https://github.com/tqdm/tqdm/issues/1928
- Status: CONFIRMED by local reproduction. Not fixed in this session.

## Observed

```
tqdm.tqdm.format_interval(float('nan')) -> ValueError: cannot convert float NaN to integer
tqdm.tqdm.format_interval(float('inf')) -> OverflowError: cannot convert float infinity to integer
```
Reachable in real bars via `.format_meter`: `remaining = (total - n) / rate`,
so e.g. `tqdm(total=float('nan'))` crashes in `close()`/`__exit__` through
`display()` -> `__str__` -> `format_meter` -> `format_interval`.

## Root cause

`tqdm/std.py` `format_interval()` does `divmod(abs(int(t)), 60)` with no
finiteness check. `int(nan)`/`int(inf)` raise.

## Suggested fix (from the issue)

Return the unknown placeholder (`'?'`) for `math.isnan`/`math.isinf` inputs
instead of raising.

## Competition

Open PR #1929 (`format_interval: render non-finite durations as the unknown
placeholder`) already implements this. Do not duplicate; review it instead.
