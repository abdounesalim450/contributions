# Bug: `CallbackIOWrapper` undercounts non-byte memoryview writes

- Issue: #1845 (2026-09-26). Link: https://github.com/tqdm/tqdm/issues/1845
- Status: CONFIRMED by local reproduction. Not fixed in this session.

## Observed

```python
from array import array
from io import BytesIO
from tqdm.utils import CallbackIOWrapper
stream = BytesIO(); counts = []
w = CallbackIOWrapper(counts.append, stream, 'write')
w.write(memoryview(array('I', [1, 2, 3])))   # stream write returns 12 (bytes)
len(stream.getvalue())                        # 12
counts                                        # [3]  <- expected [12]
```

## Root cause

`tqdm/utils.py` `CallbackIOWrapper` reports `len(data)` (element count of
the incoming buffer) to the callback instead of the byte count returned by
the wrapped stream's `write()`. For byte-format inputs these coincide;
for e.g. `array('I')` memoryviews they differ by item size. This feeds
`tqdm.wrapattr`-style progress, so bars under-report bytes written.

## Suggested fix (from the issue)

Report the integer returned by the wrapped `write()` when available; keep
the `len(data)` fallback only for streams that return no count.

## Competition

Open PRs #1846 and #1869 (plus closed #1879). Do not duplicate.
