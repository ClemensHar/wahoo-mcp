# Connecting the Wahoo MCP Server to Claude Desktop

This document summarizes the requirements and changes needed to make the Wahoo
MCP server work with **Claude Desktop** (macOS). Claude Desktop launches MCP
servers differently from Claude Code, which surfaced a couple of gotchas worth
recording.

## Requirements

- **macOS** with Claude Desktop installed.
- **`uv`** installed (this project uses it to run the server). Find its absolute
  path with `command -v uv` — on Apple Silicon Homebrew this is
  `/opt/homebrew/bin/uv`.
- A working local checkout of this repo with dependencies installed (`uv sync`).
- Valid Wahoo OAuth tokens in a `token.json` file (created via `make auth`).
- Wahoo API credentials: `WAHOO_CLIENT_ID` and `WAHOO_CLIENT_SECRET`.

## Key differences from Claude Code

Claude Code launches the MCP server **from the project directory** and with your
shell's environment. Claude Desktop does **neither**:

1. **No shell `PATH`** — Claude Desktop does not inherit your shell environment,
   so a bare `"command": "uv"` fails with "command not found". You must use the
   **absolute path** to `uv`.
2. **Working directory is `/`, not the project** — any code that reads files via
   a **relative** path breaks, because the process cwd is not the repo root.

## Changes we made

### 1. Claude Desktop config

File (macOS): `~/Library/Application Support/Claude/claude_desktop_config.json`

Added a `wahoo` entry under `mcpServers`. Note the **absolute path** to `uv` and
the **absolute paths** in the `env` block:

```json
{
  "mcpServers": {
    "wahoo": {
      "command": "/opt/homebrew/bin/uv",
      "args": [
        "--project",
        "/absolute/path/to/wahoo-mcp",
        "run",
        "python",
        "-m",
        "src.server"
      ],
      "env": {
        "WAHOO_TOKEN_FILE": "/absolute/path/to/wahoo-mcp/token.json",
        "WAHOO_CLIENT_ID": "<your-client-id>",
        "WAHOO_CLIENT_SECRET": "<your-client-secret>"
      }
    }
  }
}
```

- `--project <path>` tells `uv` where to find the project/dependencies. It does
  **not** set the process working directory.
- Paths in `env` must be **absolute** — the server runs with cwd `/`, so a
  relative `token.json` would not be found.

### 2. Code fix: make schema loading cwd-independent

`src/server.py` loaded its JSON tool schemas from a **cwd-relative** path, which
worked in Claude Code (launched from the repo) but failed in Claude Desktop
(launched from `/`).

Before:

```python
def load_json_schema(schema: str) -> dict:
    """Load a JSON schema from a file."""
    return json.loads((Path("src/schemas") / schema).read_text())
```

After — resolve relative to the module file so it works from any cwd:

```python
def load_json_schema(schema: str) -> dict:
    """Load a JSON schema from a file."""
    schemas_dir = Path(__file__).parent / "schemas"
    return json.loads((schemas_dir / schema).read_text())
```

This is a general portability fix, not just a Desktop workaround.

## `.env` interaction (good to know)

The project's `.env` sets `WAHOO_TOKEN_FILE=token.json` (relative), and
`src/server.py` calls `load_dotenv()`. Because python-dotenv defaults to
`override=False`, **environment variables already set by the Claude Desktop
config win** over `.env`. That's why the absolute `WAHOO_TOKEN_FILE` in the
config takes effect.

## Verifying the setup

You can smoke-test the server over stdio the same way Claude Desktop does, from
an unrelated working directory (e.g. `/tmp`) to catch cwd-related bugs.

Two gotchas when testing from a shell (these are **test-harness artifacts**, not
server bugs — Claude Desktop is not affected):

- **Env vars in a pipeline**: in `VAR=x cmd1 | cmd2`, the `VAR=x` prefix applies
  only to `cmd1`. Use `env VAR=x ... cmd2` to pass vars to the server.
- **stdin EOF race**: piping `printf` closes stdin immediately, which can tear
  down the server's streams before an async (network-backed) tool call finishes.
  Hold stdin open with a trailing `sleep`.

Example that exercises `initialize`, then `list_workouts`:

```bash
cd /tmp && {
  printf '%s\n' \
    '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"t","version":"1"}}}' \
    '{"jsonrpc":"2.0","method":"notifications/initialized"}' \
    '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"list_workouts","arguments":{"per_page":2}}}'
  sleep 6
} | env \
    WAHOO_TOKEN_FILE="/absolute/path/to/wahoo-mcp/token.json" \
    WAHOO_CLIENT_ID="<your-client-id>" \
    WAHOO_CLIENT_SECRET="<your-client-secret>" \
  /opt/homebrew/bin/uv --project /absolute/path/to/wahoo-mcp run python -m src.server
```

A successful run returns a JSON-RPC response containing your workouts.

In Claude Desktop itself: **fully quit (Cmd+Q) and reopen** the app after editing
the config, then confirm the `wahoo` tools appear and try
"list my recent Wahoo workouts".

## Security note

Adding the config puts `WAHOO_CLIENT_SECRET` in plaintext in
`claude_desktop_config.json` (in addition to Claude Code's config). Consider
sourcing it from a secrets manager / keychain, and rotate the secret if it may
have been exposed.
