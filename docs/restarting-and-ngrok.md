# Restarting the Wahoo MCP Server (and the ngrok question)

Quick reference for getting everything running again after quitting Claude
Desktop and all other services.

## Getting it running again

There is **no long-running service to restart**. Claude Desktop spawns the MCP
server on demand over stdio.

To use it again, just **reopen Claude Desktop**. It reads
`claude_desktop_config.json` on launch and starts the `wahoo` server itself when
a tool is called — as long as these are still true:

- `uv` is still at `/opt/homebrew/bin/uv`
- the repo still lives at the same absolute path (deps installed via `uv sync`)
- `token.json` still contains valid tokens

You do **not** need to manually start the server, activate a venv, or run
anything in a terminal.

## Do you need ngrok? Not for normal use

**ngrok is only for the interactive OAuth login (`make auth`) — never for
running or using the server.**

The server authenticates with the stored refresh token in `token.json` and
**auto-refreshes** the access token (which expires every 2h) using
`WAHOO_CLIENT_SECRET`. No browser, redirect, or tunnel is involved.

| Scenario | ngrok needed? |
|----------|--------------|
| Restart Claude Desktop and keep using it | No |
| Access token expires (every 2h) → auto-refresh | No |
| Machine rebooted, tokens still in `token.json` | No |
| Re-authenticate from scratch (`make auth`) | Yes* |

\* Only because the Wahoo app is currently registered with an ngrok HTTPS
redirect URI. `auth.py` also supports a plain `localhost` redirect, so
registering `http://localhost:8080/callback` in the Wahoo developer portal would
remove the ngrok dependency for re-auth too.

## If you do need to re-authenticate

Only when the refresh token stops working (revoked, or expired after long
disuse):

1. Start ngrok at the local auth port (`ngrok http 8080`) and ensure the Wahoo
   app's redirect URI matches the ngrok URL in `.env`.
2. Run `make auth` and complete the browser login → writes fresh tokens to
   `token.json`.
3. Shut ngrok down again.

**Bottom line:** for day-to-day use, reopen Claude Desktop and go. ngrok stays
off unless you're doing a fresh OAuth login.
