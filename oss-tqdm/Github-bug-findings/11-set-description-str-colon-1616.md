# Bug: `set_description_str()` no longer drops the colon

- Issue: #1616 (2024-09-26). Link: https://github.com/tqdm/tqdm/issues/1616
- Status: CONFIRMED by local reproduction. Not fixed in this session.

## Observed

```python
from io import StringIO
from tqdm import tqdm
out = StringIO()
bar = tqdm(total=10, file=out, mininterval=0)
bar.set_description_str("text")
bar.refresh()
# output contains "text:   0%|..."  <- colon still present
```
`set_description_str` is documented as the raw variant (no `": "` appended,
unlike `set_description`), yet both now render identically; the only
workaround is a custom `bar_format`.

## Root cause (area)

Description handling in `tqdm/std.py`: `set_description_str` vs the
`desc`/`prefix` logic in `format_meter`, which unconditionally renders
`f"{desc}: "` style prefixes for the default `bar_format` (`{desc}: ...`
 handling was changed so the raw string still passes through the colon
logic). Exact commit not bisected in this session.

## Competition

Open PR #1871 (`Stop set_description_str from appending a colon`);
closed attempts #1747 and #1758. Do not duplicate without reviewing #1871.
