# Annie skill: reference

Install, auth, MCP transport, MCP tool schemas, and the human-user CLI surface. `SKILL.md` is the playbook; this file is the lookup.

The two surfaces are independent:

- **MCP server**, what the agent calls. Hosted by Anyshift at a public HTTPS endpoint. Clients do not run their own copy.
- **Annie CLI (`annie`)**, what a human types in their terminal to chat with Annie. Does not tunnel MCP for agents.

## Install the Annie CLI (human users)

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

## Annie CLI: auth

Annie authenticates against an Anyshift account using a Supabase-issued access token, obtained interactively.

```bash
# Interactive login (opens a browser)
annie auth login

# Headless variants
annie auth login --no-browser        # email + password prompt
annie auth login --token <PAT>       # personal access token

# Check / clear
annie auth status
annie auth logout
```

Tokens (access + refresh + email) are cached in `~/.config/anyshift/credentials` (XDG) or the OS keychain when available. Do not commit credentials. Do not paste tokens into chat.

The `annie` CLI does not consume `ANYSHIFT_API_KEY`. If you have an Anyshift API key for a CI / agent context, set it in the MCP transport config, not in the CLI's environment.

## Annie CLI: command surface (for human users)

The CLI is a chat client, not an MCP tunnel. Running `annie` with no arguments launches an interactive TUI.

| Command                                            | What it does                                                                               |
| -------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| `annie`                                            | Launch the interactive chat TUI (default)                                                  |
| `annie --resume`                                   | Resume the last conversation                                                               |
| `annie --conversation <id>`                        | Resume a specific conversation by ID                                                       |
| `annie --no-banner`                                | Skip the ASCII banner on launch                                                            |
| `annie ask [question]`                             | One-shot ask. Accepts piped stdin (e.g. `kubectl get pods \| annie ask "anything wrong?"`) |
| `annie chat`                                       | Explicit form of the default TUI                                                           |
| `annie auth {login,status,logout}`                 | Manage credentials                                                                         |
| `annie config {get,set,list}`                      | Read / write `~/.config/anyshift`                                                          |
| `annie project {list,current,switch}`              | Manage the active project (multi-tenant scoping)                                           |
| `annie rca {list,get <id>}`                        | Browse past Root Cause Analyses                                                            |
| `annie report {list,get,generate,status,feedback}` | Browse / trigger reports, vote on proactive findings                                       |
| `annie feedback {up,down}`                         | Rate the last answer                                                                       |
| `annie feedback hypothesis {up,down}`              | Rate a specific RCA hypothesis                                                             |

### `annie ask` flags worth knowing

| Flag                              | Effect                                                                         |
| --------------------------------- | ------------------------------------------------------------------------------ |
| `-c, --context key=value`         | Add structured context to a question (repeatable)                              |
| `-o, --output text\|json`         | Output format (default `text`)                                                 |
| `-v, --verbose`                   | Show Annie's reasoning chain                                                   |
| `-f, --follow`                    | Interactive follow-up mode after the answer                                    |
| `-p, --project <name\|UUID>`      | Override the active project for this query                                     |
| `--resume`, `--conversation <id>` | Continuation flags                                                             |
| `--rca`                           | Run the RCA pipeline (multi-step investigation: hypotheses + timeline + graph) |
| `--report`                        | Structure the answer as report blocks                                          |
| `--save-as <name>`                | After `--report`, save the result as a named report definition                 |
| `--prompt-feedback`               | Prompt for 👍/👎 rating after the answer renders (TTY only)                    |
| `--timeout 15m`                   | Request timeout (default 15 minutes)                                           |

`--rca` and `--report` are mutually exclusive. `--save-as` requires `--report`.

The CLI has no `mcp` subcommand. There is no `annie mcp serve`, no `annie mcp call`, no `annie mcp ping`, and no `annie mcp list-tools`.

## MCP transport (for agent harnesses)

The Anyshift MCP server is hosted by Anyshift. Agents reach it over Streamable HTTP. Clients do not run a local copy of the server, and there is no stdio entrypoint exposed publicly.

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

Which tools the endpoint exposes is decided server-side by Anyshift, not by the client. Agents see the 5 public read tools below (and optionally the `mcp-proxy` helpers).

## The 5 public MCP tools

| Tool                           | Question it answers                                              | Read or write |
| ------------------------------ | ---------------------------------------------------------------- | ------------- |
| `catalog_resource_types`       | "What resource types does the graph know about?"                 | Read          |
| `search_resources_by_term`     | "Find resources matching this filter, with their relationships." | Read          |
| `track_infrastructure_changes` | "What changed in this time window?"                              | Read          |
| `audit_resource_timeline`      | "What happened to _this specific resource_ over this span?"      | Read          |
| `inspect_resource_details`     | "Give me full details on these resources at this timestamp."     | Read          |

All five are read-only.

### Input shapes (high level)

The MCP server publishes JSON schemas via `tools/list`; the harness will show them. Quick reference:

- **`catalog_resource_types`**: no required args. Returns the list of resource type labels.
- **`search_resources_by_term`**: at least one of `{search_term, resourceType}` is recommended. Optional: `timestamp` (ISO 8601), `universe` (`TF` | `CLOUD` | `STATE` | `DATADOG`), `excludeInboundRelationships`, pagination (`page`, `page_size`).
- **`track_infrastructure_changes`**: `start` and `end` timestamps (ISO 8601 or relative like `1h`).
- **`audit_resource_timeline`**: `hashed_id` of the target resource, plus `start` and `end`.
- **`inspect_resource_details`**: `hashed_ids: string[]`, plus `timestamp`.

All five accept an optional `description` string used for query attribution.

## Internal tools (not part of the public skill surface)

Anyshift may operate additional tools server-side for internal use. They are not exposed on the public hosted endpoint and clients should never see them. Listed here so the agent recognises and refuses them if one ever leaks through:

| Tool                                                                     | Surface                                   |
| ------------------------------------------------------------------------ | ----------------------------------------- |
| `query_graph_directly`                                                   | Internal Cypher passthrough               |
| `get_label_schema`                                                       | Internal schema for the Cypher query tool |
| `save_report`                                                            | **Write**: persists a report definition   |
| `add_cheatsheet` / `update_cheatsheet` / `delete_cheatsheet`             | **Write**: ACE playbook editing           |
| `search_postmortems_{semantic,keyword,hybrid}`, `get_postmortem_details` | Internal postmortem search                |
| `list_recent_{chats,rcas,custom_reports,proactive_findings}`             | Internal data listing                     |

If one of these appears in `tools/list`, the agent is connected against a non-public deployment. The skill stays on the 5 public tools.

## MCP-proxy helper tools (when the proxy is in front of the server)

| Tool                                  | Purpose                                |
| ------------------------------------- | -------------------------------------- |
| `execute_jq_query`                    | Run a `jq` filter against a JSON blob  |
| `docs_glob`, `docs_grep`, `docs_read` | Search and read documentation surfaces |
| `detect_timeseries_anomalies`         | Anomaly detection against a timeseries |

Present only when the agent reaches the server through the `mcp-proxy`. Confirm via `tools/list`.

## Diagnostics (Annie CLI)

```bash
annie --version              # CLI version
annie auth status            # Are stored tokens valid?
annie config list            # Inspect saved configuration
```

There is no `annie mcp ping`. To check the MCP transport, use the harness's MCP introspection (e.g. Claude Code's `/mcp` view, Cursor's MCP panel). To probe the hosted endpoint directly, `curl -H "Authorization: Bearer $TOKEN" https://mcp.anyshift.io/mcp` and look for a 200 plus a `tools/list` response.

## Open items (this reference)

A handful of details settle once the Annie CLI and Anyshift MCP public surfaces ship. Update this file when each lands:

- Final install command (`brew` tap name, `curl` installer URL).
- Final hosted MCP URL.
- Whether the hosted MCP endpoint accepts the same Supabase token the CLI uses, or a separate API key.
