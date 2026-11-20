# Bug: OSC 8 hyperlinks break width calculation and trimming

- Issue: #1856 (2026-10-02). Link: https://github.com/tqdm/tqdm/issues/1856
- Status: CONFIRMED by local reproduction. Not fixed in this session.
  This was candidate #3 in the initial shortlist.

## Observed

```python
from tqdm import tqdm
from tqdm.utils import disp_len, disp_trim
link = "\x1b]8;;https://example.com\x1b\\docs\x1b]8;;\x1b\\"
disp_len(link)          # 37, expected 4 (visible label "docs")
disp_trim(link, 4)      # '\x1b]8;'  <- cut inside the escape sequence
tqdm.format_meter(1, 2, 1, ncols=35, prefix=link,
                  bar_format="{desc}: {bar}| {n}/{total}")
# '\x1b]8;;https://example.com\x1b\\docs\x1b]8;;'  <- bar/counter gone, link unterminated
```

## Root cause

`tqdm/utils.py:18` `RE_ANSI = re.compile(r"\x1b\[[;\d]*[A-Za-z]")` matches
only CSI sequences (`ESC [ ...`), not OSC (`ESC ] ... ST/BEL`).
`disp_len` (utils.py:307) therefore counts OSC payload as visible text, and
`disp_trim` (utils.py:315) falls back to raw slicing that can cut inside a
sequence. Layout thus depends on URL length, which breaks bars that put
hyperlinked labels in `desc`.

## Suggested fix

Extend the ANSI regex (or add an OSC pattern) so `disp_len` ignores OSC
sequences, and make `disp_trim` never slice inside an escape sequence or
drop the OSC terminator. Tests belong in `tests/tests_utils.py`.

## Competition

Only linked PR is #1857, which is CLOSED (issue remains open), so at
re-check time there was no open PR. Verify again before starting work.
