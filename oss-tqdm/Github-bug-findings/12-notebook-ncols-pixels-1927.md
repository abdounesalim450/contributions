# Bug: `tqdm.notebook` interprets numeric `ncols` as pixels

- Issue: #1927 (2026-11-19). Link: https://github.com/tqdm/tqdm/issues/1927
- Status: CODE-INSPECTED, not run (no Jupyter/ipywidgets env in session).

## Evidence from source

`tqdm/notebook.py:126-131` (`status_printer`):
```python
if ncols is not None:  # use default style of ipywidgets
    # ncols could be 100, "100px", "100%"
    ncols = str(ncols)  # ipywidgets only accepts string
    ...
        if int(ncols) > 0:  # isnumeric and positive
            ncols += 'px'
    container.layout.width = ncols
```
Everywhere else (e.g. `tqdm/std.py`) numeric `ncols` counts *characters*.
So terminal-style code such as pooch's `tqdm.auto(..., ncols=79)` renders a
79-pixel-wide bar in notebooks/docs instead of 79 characters.

## Suggested fix (from the issue)

Treat numeric `ncols` as characters using the CSS `ch` unit
(`ncols += "ch"`); explicit strings like `"100px"`/`"100%"` keep working.
Risk: changing long-standing notebook rendering may break users who
depended on the px behaviour — maintainer call.

## Competition

Open PR #1930 (`notebook: treat a numeric ncols as characters`), based
on the same master. Review that instead of duplicating.
