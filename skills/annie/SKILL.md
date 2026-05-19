---
name: annie
description: Use the Annie CLI and the Anyshift MCP server to investigate infrastructure: resource graph, recent changes, dependents, blast radius, and temporal diffs. Use when the user asks about a resource, a deploy, an impact analysis, or wants to compare infra state across time. Keywords: annie, anyshift, mcp, infrastructure, blast radius, dependents, deploy impact, resource graph, terraform drift, temporal diff.
allowed-tools: Bash, Read, Grep
---

# Annie Skill

Teach an AI agent how to drive **Annie CLI** and the **Anyshift MCP server** to investigate infrastructure state, recent changes, and blast radius. This skill is the foundation that every other `annie-skills` skill builds on.

## What this skill does

1. **Installs and authenticates** the Annie CLI on the local host.
2. **Connects** to the Anyshift MCP server over the configured transport (stdio or HTTP).
3. **Calls the 5 read-only MCP tools** in the right order for a given question:
   - `get_resource_graph` (snapshot of resources and their relationships)
   - `get_recent_changes` (deploys, Terraform applies, IAM updates in a window)
   - `get_dependents` (what depends on a given resource)
   - `get_blast_radius` (downstream impact of a change or outage)
   - `get_temporal_diff` (state of infra at point A vs point B)
4. **Falls back gracefully** when Annie isn't reachable, when auth fails, or when the graph is stale.

## When to use this skill

Invoke this skill when the user asks anything that maps to *"what does our infra look like right now?"* or *"what changed?"* or *"what would break if I touch X?"*. Concrete triggers:

- "What does deploy `abc123` touch?"
- "Who owns this S3 bucket?"
- "What depends on the `payments-api` service?"
- "What's the blast radius of taking down the `prod-db` RDS instance?"
- "What changed in our AWS account between yesterday at 9am and now?"
- "Is the Terraform state for `vpc-foo` drifted?"

Skip this skill when the question is about **methodology** (how to run an oncall handover, how to write a postmortem). For those, use `anyshift-io/sre-skills`.

## Prerequisites

Two things must be set up before the skill runs:

1. **Annie CLI installed.** See [`reference.md`](./reference.md) for the install snippet.
2. **Anyshift MCP endpoint configured.** Either as a stdio MCP server (Annie CLI subprocess) or as a remote HTTP MCP endpoint with an API key. See [`reference.md`](./reference.md).

Run the preflight check at the start of every session:

```bash
annie auth status
annie mcp ping
```

If either fails, jump to the failure-modes section below before doing anything else.

## How to call the tools

The 5 read-only tools cover one question shape each. Pick the smallest tool that answers the question, not the biggest.

### `get_resource_graph`

Snapshot of the resource graph. Use to find a resource, see its type, see what it's connected to.

```
get_resource_graph(filter: {name: "payments-api"})
get_resource_graph(filter: {type: "aws_rds_instance", env: "prod"})
```

Returns: nodes (resources) and edges (relationships: depends-on, owned-by, deployed-by).

### `get_recent_changes`

Changes (deploys, Terraform applies, IAM mutations, manual console actions) in a time window. Use when the user asks "what changed?" or correlates an incident to a deploy.

```
get_recent_changes(since: "2026-05-19T08:00:00Z", until: "now")
get_recent_changes(resource: "payments-api", since: "1h")
```

Returns: ordered list of changes with actor, source (Terraform / console / pipeline), and affected resources.

### `get_dependents`

Reverse-edge lookup: what depends on this resource?

```
get_dependents(resource: "prod-db", transitive: true)
```

Returns: direct dependents by default; pass `transitive: true` for the full downstream tree.

### `get_blast_radius`

Impact analysis: if this resource changes or fails, what breaks? Conceptually `get_dependents(transitive: true)` filtered to "would experience user-visible impact". Use before a risky deploy or to scope an incident.

```
get_blast_radius(resource: "prod-db", change_type: "delete")
get_blast_radius(resource: "iam-role-deploy", change_type: "modify")
```

Returns: scored list of impacted resources, with a brief reason per resource.

### `get_temporal_diff`

State of the graph at point A vs point B. Use when the user wants to know what *specifically* changed between two times.

```
get_temporal_diff(at_a: "2026-05-19T08:00:00Z", at_b: "now", scope: {env: "prod"})
```

Returns: added / removed / modified resources, with the change source.

## Instructions

When invoked, the agent should:

### 1. Run preflight

Verify `annie auth status` and `annie mcp ping`. If either fails, surface the failure mode (see below) and stop.

### 2. Pick the right tool for the question

Map the user's question to the smallest tool above. Don't call `get_blast_radius` when `get_dependents` answers the question. Don't call `get_temporal_diff` when the user asked "what changed in the last hour" (use `get_recent_changes`).

### 3. Make the call

Either via the CLI (`annie mcp call <tool> --arg ...`) or via the MCP transport configured in the agent harness. See [`reference.md`](./reference.md) for both forms.

### 4. Read the result, then narrow

The first call is almost always a starting point, not the answer. Expect to follow up:

- `get_resource_graph` → `get_dependents` on a specific node.
- `get_recent_changes` → `get_blast_radius` on a suspicious change.
- `get_temporal_diff` → `get_recent_changes` to find *who* made each change.

### 5. Cite resources by their stable ID

When reporting findings to the user, quote the resource's stable ID (the one Annie returns), not just its friendly name. Two resources can share a friendly name across environments.

### 6. Stop at read-only

This skill never mutates infra. The 5 tools listed are read-only by design. If the user asks the agent to *apply* a change, hand back to a Terraform / change-management workflow. Do not invent write tools.

## Worked examples

See [`examples.md`](./examples.md) for two end-to-end flows:

1. **Investigate the impact of a recent deploy** (`get_recent_changes` → `get_blast_radius`).
2. **Diff infra state between two points in time** (`get_temporal_diff` → `get_recent_changes`).

## Failure modes

This skill is wrong, or should escalate to a human, in the following cases.

### Annie CLI not installed

`annie` not on PATH. Run the install snippet from [`reference.md`](./reference.md) and re-check. If install fails, the user is offline or behind a proxy: ask them to install manually and resume.

### Auth not configured

`annie auth status` returns "not authenticated". The user has not set up an API key (or the key has expired). Do NOT prompt the user for their API key in chat. Tell them to run `annie auth login` (or set `ANYSHIFT_API_KEY` per their org's secret-management policy) and resume.

### MCP endpoint unreachable

`annie mcp ping` fails. Either the MCP server is down, the network is blocked, or the endpoint URL is wrong. Surface the raw error and stop. Do not fall back to scraping the cloud console.

### User lacks permission for a resource

A tool call returns "forbidden" or "not visible to your account". The user's API key is scoped tighter than the resource. Report which resource was forbidden, name the permission level needed (read on `<resource-type>` in `<env>`), and stop. Don't retry against other resources hoping for a hit.

### Graph is stale

`get_resource_graph` returns a `last_synced_at` older than the user-relevant window (e.g. user is asking about a deploy from 10 minutes ago, graph is 6 hours stale). Surface the staleness, then try `get_recent_changes` for the missing window. If the change isn't there either, the answer is "Annie hasn't seen this yet, check the source of truth directly" (Terraform plan, cloud console).

### Conflicting answers across tools

`get_dependents` says X depends on Y, but `get_blast_radius` doesn't list Y when X is changed. This usually means the dependency is structural (CloudFormation parent / child) but not impact-bearing (no user traffic flows through it). Note both, do not silently pick one.

### Tool the agent expected isn't there

Anthropic / Claude tooling sometimes lists tools the MCP server doesn't expose (e.g. a write-capable tool from an older version). If the listed tool is not in the 5 above, do not call it. Tell the user the surface is limited to read-only and continue.

## Anti-patterns

- **Calling `get_resource_graph` with no filter on a large estate.** Returns thousands of nodes. Always filter by name, type, or env.
- **Using `get_temporal_diff` for "what changed in the last 5 minutes".** Use `get_recent_changes`; it's cheaper and ordered by event time.
- **Treating "no dependents" as "safe to delete".** Annie sees what it sees. A resource might be referenced from outside the graph (e.g. a hand-rolled script, a customer-facing URL). Escalate before deletion.
- **Reciting tool output verbatim.** Synthesize. The user wants the answer, not the JSON.

## Reference

For install, auth, MCP transport configuration, and exact CLI flags, see [`reference.md`](./reference.md).
