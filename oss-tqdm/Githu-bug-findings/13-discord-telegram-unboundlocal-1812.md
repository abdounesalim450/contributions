# Bug: discord/telegram contrib mask connection errors with UnboundLocalError

- Issue: #1812 (2026-09-03). Link: https://github.com/tqdm/tqdm/issues/1812
- Status: CODE-INSPECTED, not run (`requests` not installed in session env;
  a mocked-network repro was attempted and blocked on the missing package).

## Evidence from source

`tqdm/contrib/discord.py:41-52` (`DiscordIO.message_id`, mirrored in
`tqdm/contrib/telegram.py:38-49` `TelegramIO.message_id`):

```python
try:
    req = self.session.post(...)
    res = req.json()
    req.raise_for_status()
except ...:
    if req.status_code == 429:   # <- req unbound if post() itself raised
```

If `post()` raises (connection refused, DNS, timeout), `req` was never
assigned, so the handler raises `UnboundLocalError` and hides the original
`requests` exception. The issue's repro points `DiscordIO.API` at a closed
local port with proxies disabled.

## Suggested fix

Initialise `req = None` before the `try` and guard the 429 handling
(`if req is not None and req.status_code == 429`), or restructure so the
original exception propagates. Apply identically to both files, and check
`tqdm/contrib/slack.py` for the same pattern before fixing only two.

## Competition

Open PR #1837; closed attempts #1818, #1913. Review #1837 first.
