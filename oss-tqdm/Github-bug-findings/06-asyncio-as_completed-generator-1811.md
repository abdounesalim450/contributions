# Bug: `tqdm.asyncio.as_completed` rejects generators

- Issue: #1811 (2026-09-03). Link: https://github.com/tqdm/tqdm/issues/1811
- Status: CONFIRMED by local reproduction (Python 3.12). Not fixed here.

## Observed

```python
import asyncio
from tqdm.asyncio import tqdm_asyncio

async def main():
    tasks = (asyncio.create_task(asyncio.sleep(0, result=v)) for v in range(3))
    for fut in tqdm_asyncio.as_completed(tasks):   # TypeError: object of type 'generator' has no len()
        await fut
```
The same generator works with stdlib `asyncio.as_completed` (Python >= 3.12
accepts generators of tasks). Passing a list, or explicit `total=3`, works.

## Root cause

`tqdm/asyncio.py` `as_completed()` does `total = len(fs)` unconditionally
when `total` is omitted.

## Suggested fix

Materialise unsized iterables (e.g. `fs = list(fs)` when it has no length)
or use `length_hint` and pass `total=None` for truly unsized inputs; note
materialising changes laziness for huge generators, so document/test it.

## Competition

Open PR #1815; closed attempts #1817, #1842, #1912. Check #1815 before
doing anything here.
