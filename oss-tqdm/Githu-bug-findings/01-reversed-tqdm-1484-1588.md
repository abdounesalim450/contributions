# Bug: `reversed(tqdm(...))` yields items in forward order

- Issues: #1484 (original, 2023-07-23, `tqdm([1,2,3,4])`),
  #1588 (duplicate, 2024-06-05, `trange`). Both still open, no labels.
- Links: https://github.com/tqdm/tqdm/issues/1484 ,
  https://github.com/tqdm/tqdm/issues/1588
- Status: CONFIRMED by local reproduction; fixed locally in this session.

## Expected vs actual (observed, master `9cf5a12`, Python 3.12.16)

```
next(reversed(trange(10)))        expected 9          actual 0
list(reversed(trange(10)))        expected [9..0]     actual [0..9]
list(reversed(tqdm([1,2,3,4])))   expected [4,3,2,1]  actual [1,2,3,4]
list(reversed(tqdm(range(9))))    expected [8..0]     actual [0..8]
```
Repro script: `/tmp/repro_1588.py` (exit 1 before fix, 0 after).
Controls passed: forward iteration; workaround `tqdm(reversed(range(5)))`.
`reversed()` on a generator-backed bar correctly raises
`TypeError: 'generator' object is not reversible`.

## Root cause

`tqdm/std.py:1125` `tqdm.__reversed__` swapped in `reversed(self.iterable)`,
returned `self.__iter__()`, then a `finally` restored the original iterable.
`tqdm/std.py:1166` `__iter__` is a generator, so `iterable = self.iterable`
(line 1170) runs only on first `next()` — after the restore. The existing
`tests/tests_tqdm.py:1561` `test_reversed` only asserted bar output
(`0%`/`100%`), never order, so CI stayed green.

## Open PRs for this same bug (do not duplicate)

- #1485 (`__iter__` refactor, 2023, conflicts)
- #1591 (shallow-copy bar, 2024, conflicts; returns a `tqdm` object, and we
  verified `next()` on it raises `TypeError: 'tqdm' object is not an
  iterator`, failing #1588's exact example). #1589 was its closed predecessor.
- #1711 (makes `__reversed__` a generator, 2026, conflicts; TypeError for
  non-reversible iterables becomes lazy instead of eager)
- #1903 (nested-generator swap, 2026-10-26, same base `9cf5a12`, awaiting
  review) — essentially the same fix as ours.

## What we did

- Local branch `fix-reversed-1588`, commit `428cef0` (not pushed):
  `__reversed__` evaluates `reversed(orig)` eagerly (immediate `TypeError`,
  like the builtin) and returns a nested generator that swaps the iterable
  inside iteration and restores it in its `finally`; also removed a latent
  `UnboundLocalError` in the old `finally` when `self.iterable` was missing.
- Strengthened `test_reversed`: order for range+list, `next(reversed(trange(10))) == 9`,
  eager `TypeError` for generators. Failed before fix
  (`assert [0, 1, ...] == [8, 7, ...]`), passes after. Edge probes in
  `/tmp/probe_reversed_edges.py`: iterable restored after exhaustion and
  after `close()`, never-started iterator harmless, disabled bars work.
- Suite: `tests/tests_tqdm.py` 85 passed/1 skipped; full `tests/`
  (not perf) 161 passed/11 skipped. flake8 clean on `std.py`.
- Posted approved test-suggestion comment on #1903 instead of opening a 5th PR:
  https://github.com/tqdm/tqdm/pull/1903#issuecomment-7051352177

## Suggested next step

Wait for #1903. Our commit is a superseding fallback only if #1903 stalls,
and must credit #1485/#1591/#1711/#1903 if ever used.
