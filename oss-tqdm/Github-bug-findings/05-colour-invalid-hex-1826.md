# Bug: invalid hex `colour='#00ff0g'` raises ValueError instead of warning

- Issue: #1826 (2026-09-10). Link: https://github.com/tqdm/tqdm/issues/1826
- Status: CONFIRMED by local reproduction. Not fixed in this session.

## Observed

```python
from io import StringIO
from tqdm import tqdm
list(tqdm(range(1), file=StringIO(), colour="not-a-colour", mininterval=0))  # warns, OK
list(tqdm(range(1), file=StringIO(), colour="#00ff0g", mininterval=0))
# ValueError: invalid literal for int() with base 16: '0g'
```

## Root cause

`tqdm/std.py` `Bar.colour` setter: unknown colour *names* are caught
(`KeyError`/`AttributeError` -> `TqdmWarning`, bar continues uncoloured),
but a 7-char `#......` string goes through
`int(i, 16) for i in (value[1:3], value[3:5], value[5:7])` and its
`ValueError` is not caught.

## Suggested fix

Catch `ValueError` in the hex branch and fall through to the existing
unknown-colour warning path.

## Competition

Two open PRs already: #1829 and #1897, both doing exactly this. Do not
duplicate; reviewing one of them is the useful contribution.
