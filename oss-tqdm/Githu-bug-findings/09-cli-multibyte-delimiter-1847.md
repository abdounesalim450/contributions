# Bug: multibyte CLI delimiters split across read buffers are missed

- Issue: #1847 (2026-09-26). Link: https://github.com/tqdm/tqdm/issues/1847
- Status: CONFIRMED by local reproduction. Not fixed in this session.

## Observed

```sh
printf '1é2é3' | python -m tqdm --delim 'é' --buf-size 2 --bar-format '{n}it'
# reports 2it, expected 3it   (é is 2 bytes in UTF-8; buffer cuts it in half)

printf '1é2é3' | python -m tqdm --delim 'é' --buf-size 2 --update --bar-format '{n}it'
# SyntaxError while parsing '1é2' (two records glued), expected 6it
```

## Root cause

`tqdm/cli.py` `posix_pipe` searches each fresh read chunk for the delimiter
*before* combining it with the saved prefix from the previous chunk, so a
delimiter straddling two chunks never matches. Stdout payload itself stays
byte-identical; only counting/`--update` parsing breaks.

## Suggested fix (from the issue)

Search the combined (prefix + chunk) bytes so split delimiters can match,
keeping the passthrough payload unchanged.

## Competition

Open PRs #1848 and #1907 (closed: #1854, #1909). Do not duplicate.
