# Bug: `python -m tqdm --log` with missing/invalid level crashes raw

- Issue: #1827 (2026-09-10). Link: https://github.com/tqdm/tqdm/issues/1827
- Status: CONFIRMED by local reproduction. Not fixed in this session.

## Observed

```
$ printf '' | python3 -m tqdm --log
  File "tqdm/cli.py", line 177, in main
    logLevel = argv[log_idx + 1]
IndexError: list index out of range

$ printf '' | python3 -m tqdm --log NOTALEVEL
  File "tqdm/cli.py", line 178, in main
    logging.basicConfig(level=getattr(logging, logLevel), ...)
AttributeError: module 'logging' has no attribute 'NOTALEVEL'
```

## Root cause

`tqdm/cli.py` `main()` parses `--log` before the `try` block that converts
bad CLI options into `TqdmKeyError`/`TqdmTypeError`, with no bounds or level
validation: `argv[log_idx + 1]` and bare `getattr(logging, logLevel)`.

## Suggested fix (from the issue)

Move `--log` parsing into the `try` block; validate presence of the value
and that the level exists (also consider `--log=LEVEL` forms used elsewhere
in the CLI parser).

## Competition

Open PR #1831 (`Fix CLI error handling for invalid log levels`); #1910 was
closed. Do not duplicate without checking #1831 first.
