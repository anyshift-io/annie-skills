# Contributing to annie-skills

This is the Anyshift-specific skills library. Contributions are welcome from anyone who uses Annie and the Anyshift MCP, individuals and other vendors included.

## What belongs in this repo

Skills that teach an AI agent how to use **Annie CLI** or the **Anyshift MCP server**. Tool-usage skills, by design.

Skills that are vendor-neutral by design (methodology, runbooks, postmortem patterns) belong in [`anyshift-io/sre-skills`](https://github.com/anyshift-io/sre-skills), not here.

A useful test: if the skill runs end-to-end without Annie, it belongs in `sre-skills`. If the skill needs Annie or the Anyshift MCP to work, it belongs here.

## The bar

Every skill in this repo ships with:

1. **A self-contained `SKILL.md`.** Runnable end-to-end in Claude Code. YAML frontmatter (`name`, `description`, optional `allowed-tools`). No external state assumed beyond Annie CLI and an Anyshift MCP endpoint.
2. **Two worked examples.** Realistic flows showing the skill in motion. Live in `examples.md` or a sibling file.
3. **An explicit failure-modes section.** What does the agent do when Annie isn't reachable, when auth fails, when the user lacks permissions, when the graph is stale. List the modes honestly.
4. **A copy-pasteable install snippet.** Reader can install Annie, set the MCP endpoint, and run the skill in under 60 seconds.

Skills that don't meet this bar stay in PR until they do.

## Skill layout

The canonical layout is established by `skills/annie/` (the reference template). Mirror that structure for new skills:

```
skills/<skill-name>/
  SKILL.md         # frontmatter + instructions
  examples.md      # 2 worked examples
  reference.md     # tool reference / install / auth (optional)
```

## How to add a new skill

1. Fork the repo.
2. Create `skills/<skill-name>/` mirroring the reference template.
3. Open a PR. Reviewers check: the four quality-bar items above, scope (does this belong here vs in `sre-skills`), and accuracy against the current Annie / Anyshift MCP surface.

## License

By contributing, you agree your contribution is licensed under [Apache 2.0](./LICENSE).
