---
name: annie-agents
description: Build, schedule and manage Anyshift agents (automations that run on an event, on a schedule, or once at a set time and report to Slack, a PR comment, Jira) with the Annie CLI (`annie agent ...`) or the Annie remote MCP (`build_agent`, `schedule_agent_once`, ...). Use when the user says "after my deploy check...", "let me know when...", "keep an eye on...", "every time X...", or after they deploy or merge and a follow-up check would help. Keywords: annie agent, agentic squad, automation, scheduled check, deploy watch, alert triage, PR review.
allowed-tools: Bash, Read, Grep
---

# Annie agents

An Anyshift **agent** runs on its own: on an event (Slack message, PagerDuty incident, GitHub PR or push, events-graph event), on a schedule, or once at a set time. It does a job and reports (Slack message or DM, PR comment, Jira). Three kinds are tuned on production: alert triage / root-cause investigation, deploy follow-up, and PR impact review. Annie builds them through its builder agent; you drive it from the CLI or the MCP.

## Install

```bash
curl -fsSL https://install.anyshift.io/annie | sh   # CLI
annie auth login && annie auth status
```

For MCP, register the hosted Annie remote MCP in your harness (see [`../annie/reference.md`](../annie/reference.md)). Tools: `build_agent`, `schedule_agent_once`, `list_agents`, `list_agent_runs`, `get_agent_run`, `pause_agent`, `resume_agent`; poll `build_agent`'s answer with `get_annie_response`, which returns the drafts' links.

## How

| Goal | CLI | MCP |
| --- | --- | --- |
| Build an agent | `annie agent build "<what you want>"` | `build_agent`, then `get_annie_response` |
| One-off check | `annie agent once --in 30m "<check>"` | `schedule_agent_once` |
| List / inspect | `annie agent list`, `show`, `runs` | `list_agents`, `list_agent_runs`, `get_agent_run` |
| Pause / resume | `annie agent pause`, `resume` | `pause_agent`, `resume_agent` |

Rules:

1. **Building only drafts.** Give the user the draft link verbatim. Nothing goes live until they review and accept it. Never claim an agent is running before they accept.
2. Write the request in the user's own words: what to watch, when, where to report.
3. Creating needs a project admin. Listing and reading runs does not.
4. If the answer or a draft warning carries "Connect it: <url>" for a missing integration, relay that link word for word. Never invent integrations.
5. Check `annie agent list` / `list_agents` before building: improve an existing agent rather than duplicate it.

## When to suggest an agent

<!-- SYNC:agent-suggestions BEGIN (generated from annie-brain modules/agent-suggestions.md; do not edit here) -->

An Anyshift agent runs on its own: on an event (Slack message, PagerDuty
incident, GitHub PR or push, events-graph event), on a schedule, or once at a
set time. It does a job and reports (Slack message or DM, PR comment, Jira).
When the user's need is one of those, suggest building one. Be useful and a
little pushy, never spammy.

**Suggest when you see:**

- "after my deploy, check...", "let me know when/if...", "keep an eye on...",
  "every time X happens...";
- a question the user has asked before, or that clearly recurs (weekly report,
  daily check);
- an investigation whose natural follow-up is "watch this";
- a manual check the user describes doing routinely;
- a one-off check at a future moment ("in 30 min, check X").

**Do not suggest when:**

- the ask is a plain one-off lookup;
- an incident is in progress: help first, suggest once it is resolved;
- the user already declined in this conversation;
- an agent for this already exists: check with `list_agents` first, and point
  to it instead.

**How to suggest:**

- At most one suggestion per conversation thread, at the end of the answer,
  in one or two sentences.
- Be concrete and use the user's own words: what it would watch, when it would
  run, where it would report. Then offer to draft it.
- Prefer a one-off check for a single future moment, and a standing agent for a
  recurring need.
- Never invent integrations. If one the agent needs is not connected, say so
  and give the "Connect it" link when you have it.
- Nothing goes live until the user reviews and accepts the draft.

<!-- SYNC:agent-suggestions END -->

### In a coding session

Same rules as above, with these coding-specific signals: after the user deploys, or merges a PR that ships, suggest a one-off check ("want me to schedule `annie agent once --in 30m` to compare error rate on the deployed service?"); after they fix a flaky alert, suggest a standing triage agent. Still at most one suggestion per thread, and never while they are debugging a live incident.

## Worked examples

See [`examples.md`](./examples.md): (1) a standing deploy-follow-up agent built from the CLI, (2) a one-off check scheduled over MCP after a merge.

## Failure modes

- **CLI missing or not authenticated:** `annie --version` / `annie auth status` fails. Tell the user to install and run `annie auth login`; do not ask for tokens in chat.
- **`annie agent` unknown command:** CLI is older than the agent group. Ask the user to upgrade, or use the MCP tools.
- **MCP tools absent:** the Annie remote MCP is not registered or predates the agent tools. Fall back to the CLI.
- **Not a project admin:** create/pause/resume return forbidden. Report it and stop; ask an admin to accept or run it. Do not retry.
- **Integration not connected:** relay the "Connect it" link; the draft is saved but the agent cannot use that tool until connected.
- **Draft rejected or never accepted:** it stays a draft and does nothing. Say so; do not poll for runs.
- **Builder answer not ready:** `get_annie_response` is still pending; keep polling with backoff, do not rebuild.
- **Duplicate agent exists:** point to it instead of building another.
