# annie-skills

[![License](https://img.shields.io/badge/license-Apache_2.0-blue.svg)](./LICENSE)

Public library of Anyshift-specific skills for AI agents.

Built and maintained by [Anyshift](https://www.anyshift.io). Each skill teaches an AI agent how to use the [Annie CLI](https://www.anyshift.io) and the [Anyshift MCP](https://www.anyshift.io) server: how to install, how to authenticate, how to call each tool, and what to do when things go wrong.

## Relationship to `sre-skills`

This repo is the **vendor-specific** sibling of [`anyshift-io/sre-skills`](https://github.com/anyshift-io/sre-skills).

- `sre-skills` packages **methodology** (investigating an incident, handing over oncall, authoring a postmortem) and runs vendor-neutral by default.
- `annie-skills` packages **tool usage** (Annie CLI, Anyshift MCP) and assumes Annie is available. If you don't run Anyshift, this repo isn't for you. Start at `sre-skills` instead.

The split mirrors the layered architecture described in the `sre-skills` README:

1. Vendor-neutral default (in `sre-skills`).
2. Anyshift MCP as optional context primer (cross-referenced from `sre-skills`).
3. Annie pre-loaded (this repo).

## Skills

| Skill | Status | What it does |
|---|---|---|
| `annie` | *Shipping by end of May 2026* | Teaches the agent how to drive the Annie CLI and call the 5 read-only Anyshift MCP tools (`get_resource_graph`, `get_recent_changes`, `get_dependents`, `get_blast_radius`, `get_temporal_diff`). |

More skills will follow as the surface of Annie and the Anyshift MCP grows.

## Quality bar

Every skill in this repo ships with:

- A self-contained `SKILL.md` runnable end-to-end in Claude Code.
- Two worked examples against realistic scenarios.
- An explicit failure-modes section: what to do when Annie isn't reachable, when the user lacks permissions, when the graph is stale.
- A copy-pasteable install snippet in the skill README.

Recordings are optional for tool-usage skills (where a screen demo adds less than a clean transcript would).

## Adjacent

- [`anyshift-io/sre-skills`](https://github.com/anyshift-io/sre-skills), vendor-neutral SRE methodology skills.
- [`anyshift-io/awesome-sre-skills`](https://github.com/anyshift-io/awesome-sre-skills), curated index of SRE skills, MCP servers, and reading.

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md).

## License

[Apache 2.0](./LICENSE).
