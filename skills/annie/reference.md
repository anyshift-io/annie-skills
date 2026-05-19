# Annie skill: reference

Install, auth, MCP transport, and exact CLI flags. The `SKILL.md` is the playbook; this file is the lookup.

## Install

> The exact install command is finalized as part of the Annie CLI public release. The placeholders below show the supported install shapes; replace with the canonical command when the public CLI ships.

### macOS / Linux

```bash
# Homebrew (recommended once tap is public)
brew install anyshift-io/tap/annie

# curl installer
curl -fsSL https://install.anyshift.io/annie | sh
```

### Verify

```bash
annie --version
annie auth status
```

If `annie auth status` says "not authenticated", run the auth step below.

## Auth

Annie authenticates against an Anyshift account using an API key.

```bash
# Interactive login (opens browser)
annie auth login

# Or set the key directly (CI / non-interactive)
export ANYSHIFT_API_KEY="ash_..."
annie auth status
```

Auth state is cached in `~/.config/anyshift/credentials` (XDG) or the OS keychain when available. Do not commit credentials. Do not paste keys into chat.

## MCP transport

The Anyshift MCP server can be reached two ways. Pick one per environment.

### Stdio (Annie CLI as MCP subprocess)

Use this when the agent runs on the same host as the Annie CLI. The CLI exposes the MCP server over stdin / stdout.

Claude Code (`~/.claude.json` or `.mcp.json` in the repo):

```json
{
  "mcpServers": {
    "anyshift": {
      "command": "annie",
      "args": ["mcp", "serve"],
      "env": {
        "ANYSHIFT_API_KEY": "${ANYSHIFT_API_KEY}"
      }
    }
  }
}
```

Cursor / Cline / Continue.dev: same shape, server name `anyshift`, command `annie mcp serve`.

### Remote HTTP

Use this when the agent runs in a sandbox without the Annie CLI on PATH. The MCP server is hosted by Anyshift; the agent talks to it over Streamable HTTP.

```json
{
  "mcpServers": {
    "anyshift": {
      "url": "https://mcp.anyshift.io/mcp",
      "headers": {
        "Authorization": "Bearer ${ANYSHIFT_API_KEY}"
      }
    }
  }
}
```

> The hosted MCP URL (`mcp.anyshift.io/mcp`) is finalized as part of the public MCP release. Replace with the canonical URL when the public endpoint ships.

## CLI cheatsheet

The skill invokes MCP tools through one of two shapes.

### Through the CLI directly (any harness)

```bash
annie mcp call <tool> --<arg> <value> ...
```

Examples:

```bash
annie mcp call get_resource_graph --filter '{"name": "payments-api"}'
annie mcp call get_recent_changes --since "1h" --resource "payments-api"
annie mcp call get_dependents --resource "prod-db" --transitive true
annie mcp call get_blast_radius --resource "prod-db" --change_type "delete"
annie mcp call get_temporal_diff \
  --at_a "2026-05-18T08:00:00Z" --at_b "now" \
  --scope '{"env": "prod"}'
```

### Through the agent harness (MCP-native)

Tool names match the CLI: `get_resource_graph`, `get_recent_changes`, `get_dependents`, `get_blast_radius`, `get_temporal_diff`. Arguments match the CLI flags.

## The 5 tools

| Tool | Question it answers | Read or write |
|---|---|---|
| `get_resource_graph` | "What resources match this filter, and what are they connected to?" | Read |
| `get_recent_changes` | "What changed in this time window?" | Read |
| `get_dependents` | "What depends on this resource?" | Read |
| `get_blast_radius` | "What breaks if this resource changes or fails?" | Read |
| `get_temporal_diff` | "What's different between point A and point B?" | Read |

All five are read-only. The skill does not call any write-capable tool, even if the MCP surface exposes one in a later version.

## Diagnostics

```bash
annie auth status            # is the API key valid?
annie mcp ping               # is the MCP endpoint reachable?
annie mcp list-tools         # what tools does this endpoint expose?
annie mcp call <tool> --help # what arguments does this tool take?
```

When opening a bug, attach the output of `annie diagnose` (collects version, auth state, MCP endpoint, last 5 tool calls).

## Open items (this reference)

A handful of details settle once the Annie CLI and Anyshift MCP public surfaces ship. Update this file when each lands:

- Final install command (`brew` tap name, `curl` installer URL).
- Final hosted MCP URL.
- Whether `ANYSHIFT_API_KEY` is the canonical env var or a different name.
- Whether `annie mcp call` is the final CLI shape (vs `annie call <tool>`).
