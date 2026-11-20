# Finding: `process_map` unsized-iterable crash is ALREADY FIXED (#1801)

- Issue: #1801 (2026-08-20, still open).
  Link: https://github.com/tqdm/tqdm/issues/1801
- Status: NOT REPRODUCIBLE on master `9cf5a12` — stale issue. This file
  exists so nobody wastes time "fixing" it again.

## Reported bug

`process_map(print, (i for i in range(5)))` raised
`ValueError: min() iterable argument is empty` from
`tqdm/contrib/concurrent.py` `_min_map_len`, because
`length_hint(generator, -1)` returns `-1` and all iterables were filtered
out before `min()` (regression from the #1473 shortest-iterable change in
4.70.0, per the issue's own archaeology).

## Why it no longer reproduces

Current master already contains the fix at
`tqdm/contrib/concurrent.py:102-104`:

```python
def _min_map_len(iterables):
    """min(map(length_hint, iterables), default=0)"""
    return min((n for it in iterables if (n := length_hint(it, -1)) >= 0), default=0)
```

Local check in this session: `_min_map_len([(i for i in range(5))])`
returned `0` (no exception). The remaining work is tracker hygiene
(closing #1801 and the now-redundant open PRs #1802/#1922; #1805 closed),
which only maintainers can do. Note `process_map` progress with
`chunksize > 1` (#1791, open PRs #1792/#1793) is a *separate* still-real
bug in the same file — do not conflate them.
