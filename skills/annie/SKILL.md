---
name: annie
description: Use the Anyshift MCP server (and the Annie CLI as the user-facing chat client) to investigate infrastructure: discover resource types, search the graph, track changes, audit a single resource's timeline, and inspect resources in detail. Use when the user asks about a resource, a deploy, an impact analysis, or wants to compare infra state across time. Keywords: annie, anyshift, mcp, infrastructure, resource graph, change tracking, audit timeline, temporal diff.
allowed-tools: Bash, Read, Grep
---

# Annie Skill

Teach an AI agent how to drive the **Anyshift MCP server** to investigate infrastructure state, recent changes, and resource relationships, and how to point a human user at the **Annie CLI** when they want to ask Annie questions themselves.

## Mental model

There are two surfaces in this ecosystem, and they are not the same:

1. **Anyshift MCP server.** A read-mostly MCP server that exposes the Anyshift infrastructure graph (resources, relationships, change history) to any MCP-capable agent harness (Claude Code, Cursor, Cline, etc.). This is the surface the agent calls directly. Tool names are `snake_case`.
2. **Annie CLI (`annie`).** A user-facing terminal chat client. Running `annie` with no arguments launches an interactive TUI; `annie ask "..."` is the one-shot variant. The CLI is for humans talking to Annie, not for agents tunnelling MCP tool calls.

**The CLI does not expose an `annie mcp call` interface.** Agents invoke MCP tools through the agent harness's MCP transport, not through the CLI.

## What this skill does

1. Helps the agent **pick the right MCP tool** for the question.
2. Shows the agent how the MCP server is **configured** in a Claude Code / Cursor / Cline harness.
3. Tells the human user where the **Annie CLI** fits and how to install / auth it.
4. **Falls back gracefully** when the MCP server isn't reachable, when auth fails, or when the graph is stale.

## When to use this skill

Invoke this skill when the user asks anything that maps to _"what does our infra look like right now?"_ or _"what changed?"_ or _"who/what touches X?"_. Concrete triggers:

- "What changed in our AWS account between yesterday at 9am and now?"
- "What does this deploy touch?"
- "Who owns this S3 bucket? What is it connected to?"
- "Walk me through the history of `vpc-foo`."
- "Pull the current details on these three resources at once."

Skip this skill when the question is about **methodology** (how to run an oncall handover, how to write a postmortem). For those, use the `sre-skills` plugin.

## Prerequisites

The MCP server is hosted by Anyshift at a public HTTPS endpoint. Clients do not run their own copy. See [`reference.md`](./reference.md) for the transport config. The skill assumes:

- An MCP server registered (as `anyshift` or equivalent) in the agent harness config, pointing at the hosted endpoint.
- A valid bearer token for the hosted endpoint.

If the user also wants to ask Annie questions themselves (not through the agent), install the Annie CLI (`annie --version`, then `annie auth login`).

## The 5 public MCP tools

The agent has five read tools by default. Pick the smallest tool that answers the question, not the biggest. The MCP server itself instructs callers to start with `catalog_resource_types` whenever the resource-type label is unclear, because it returns the exact labels used elsewhere.

### `catalog_resource_types`

Discovery starting point. Returns every resource type the graph knows about. Use first when you don't know what to filter by, or when the user's noun ("the EC2 things", "the buckets") needs to be mapped to a canonical type label.

### `search_resources_by_term`

Flexible search across resources by name, property, or type, at any timestamp. Returns matched resources with their properties and relationship context (inbound + outbound by default). Supports filtering by `universe` (`TF`, `CLOUD`, `STATE`, `DATADOG`) and `resourceType`. Paginated.

```
search_resources_by_term({ search_term: "prod-vpc" })
search_resources_by_term({ resourceType: "S3_BUCKET" })
search_resources_by_term({ search_term: "database", timestamp: "2026-05-18T10:00:00.000Z" })
search_resources_by_term({ universe: "CLOUD", search_term: "web" })
```

This is also how you reach "what is X connected to?". Relationships ride along with each match unless `excludeInboundRelationships: true` is set.

### `track_infrastructure_changes`

Lists graph nodes modified between two timestamps. Use when the user asks "what changed in this window?". Returns an ordered change set with the affected resources.

```
track_infrastructure_changes({ start: "2026-05-19T08:00:00Z", end: "now" })
```

### `audit_resource_timeline`

Comprehensive change history for one specific resource by its `hashed_id`, over a timespan. Use when the user wants the lifecycle of a single resource (e.g. "what's happened to `vpc_prod_main` this week?"), not "what changed across the estate".

```
audit_resource_timeline({ hashed_id: "<from a prior search>", start: "...", end: "..." })
```

### `inspect_resource_details`

Batch fetch full details (properties + relationships) for one or more resources at a given timestamp. Use after `search_resources_by_term` when you have a short list of `hashed_id`s and want the complete picture in one call.

```
inspect_resource_details({ hashed_ids: ["id1", "id2", "id3"], timestamp: "now" })
```

## What is NOT a tool

Things the previous version of this skill claimed existed and do not:

- There is no `get_blast_radius` primitive. Impact analysis is assembled from the relationship graph: start with `search_resources_by_term` on the candidate resource, follow its outbound relationships, then `inspect_resource_details` on the dependents. Report the chain, not a "blast radius score": the server doesn't compute one.
- There is no `get_dependents` primitive. Dependents are derivable from the relationship payload that `search_resources_by_term` and `inspect_resource_details` already return.
- There is no `get_temporal_diff` primitive. The diff shape "added / removed / modified between A and B" is best assembled from `track_infrastructure_changes` over the window, optionally narrowed with `audit_resource_timeline` per resource.
- There is no public write tool. Internal write tools exist (cheatsheet editing, report saving, ACE flows) but they run only on Anyshift-internal deployments and are out of scope for this skill. If a tool call surfaces an unexpected write tool, do not call it (see "Failure modes" below).

## Optional: MCP-proxy helper tools

When the agent connects through the hosted MCP endpoint, an `mcp-proxy` may sit in front of the server and expose a handful of investigation helpers:

- `execute_jq_query`: run a `jq` filter against a JSON blob (handy for slicing tool output).
- `docs_glob` / `docs_grep` / `docs_read`: search and read documentation surfaces.
- `detect_timeseries_anomalies`: anomaly detection against a timeseries.

These do not always appear (the proxy is optional). Check `tools/list` in the harness to see what's actually exposed in the current session.

## How to call the tools

Through the MCP transport in the agent harness. Tool names are `snake_case` and match this document exactly. Arguments are JSON-shaped, matching each tool's input schema (the harness will surface the schema; if you need it explicitly, ask the MCP server's `tools/list`).

The Annie CLI is not the call path for agents.

## Instructions

When invoked, the agent should:

### 1. Verify the MCP server is reachable

If the harness lists the `anyshift` server in `tools/list`, you're good. If not, surface the failure mode (see below) and stop.

### 2. Pick the right tool for the question

Map the user's question to the smallest tool. Don't call `inspect_resource_details` for thousands of IDs when `search_resources_by_term` with a filter is cheaper. Don't call `audit_resource_timeline` for a window-wide question (use `track_infrastructure_changes`).

### 3. Chain tools

The first call is almost always a starting point, not the answer. Expect to follow up:

- `catalog_resource_types` → `search_resources_by_term` filtered to the right type.
- `search_resources_by_term` → `inspect_resource_details` on the matched `hashed_id`s for full properties + relationships.
- `track_infrastructure_changes` → `audit_resource_timeline` to drill into a specific changed resource.

### 4. Cite resources by their `hashed_id`

When reporting findings to the user, quote the resource's `hashed_id`, not just its friendly name. Two resources can share a friendly name across environments; the `hashed_id` disambiguates.

### 5. Stop at read

The public surface is read-only by design. If the user asks the agent to _apply_ a change, hand back to Terraform / change-management. Do not invent write tools.

## Worked examples

See [`examples.md`](./examples.md) for two end-to-end flows:

1. **Investigate the impact of a recent deploy** (`track_infrastructure_changes` → `search_resources_by_term` → `inspect_resource_details`).
2. **Walk the timeline of a specific resource** (`search_resources_by_term` → `audit_resource_timeline`).

## Failure modes

### MCP server not configured

The harness has no `anyshift` server in `tools/list`. Tell the user to register the hosted MCP endpoint in their harness config; see [`reference.md`](./reference.md). Do not fall back to scraping the cloud console.

### Auth not configured or expired

A tool call returns 401 / "unauthorized". The access token is missing or expired. Do not prompt for an API key in chat. Tell the user to refresh the token (`annie auth login` if using the CLI's stored Supabase tokens, or rotate the bearer token in the harness config).

### User lacks permission for a resource

A tool call returns "forbidden" or "not visible to your project". The user's access is scoped tighter than the resource. Report which resource was forbidden and stop. Do not retry against other resources hoping for a hit.

### Graph is stale

A search returns nothing for something you know exists. Possible causes: the indexer hasn't caught up (the change is recent), or the resource is in a universe the user's account doesn't see. Surface the staleness, then try `track_infrastructure_changes` over the missing window. If it isn't there either, the answer is "the graph hasn't seen this yet, check Terraform plan / cloud console directly".

### Tool listed but not actually callable

The harness sometimes lists tools the hosted server doesn't currently expose. If a call errors with "unknown tool", fall back to the 5 documented above. Do not invent variations on the tool name.

### A write-capable tool appears

Some MCP execution modes expose internal write tools (cheatsheet editing, report saving, ACE flows). They require an Anyshift-internal API key and are not part of this skill's public surface. If you see one, do not call it. Tell the user the public surface is read-only and continue.

## Anti-patterns

- **Calling `search_resources_by_term` with no filter on a large estate.** Returns thousands of nodes. Always pass a `search_term`, `resourceType`, or `universe`.
- **Calling `audit_resource_timeline` for "what changed in the last 5 minutes" across everything.** Use `track_infrastructure_changes`; it's window-scoped.
- **Treating "no relationships" as "safe to delete".** The graph sees what it sees. A resource might be referenced from outside the graph (a hand-rolled script, a customer-facing URL). Escalate before deletion.
- **Reciting tool output verbatim.** Synthesize. The user wants the answer, not the JSON payload.
- **Calling tools through the `annie` CLI.** The CLI doesn't tunnel MCP. Use the harness's MCP transport.

## Reference

For install, auth, MCP transport configuration, the actual tool input schemas, and the Annie CLI surface for human users, see [`reference.md`](./reference.md).
