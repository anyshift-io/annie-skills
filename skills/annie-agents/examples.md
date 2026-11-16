# Annie agents: worked examples

## Example 1: standing deploy follow-up (CLI)

User: "After each prod deploy of `payments-api`, check error rate and latency for 30 minutes and tell me in #payments if it regressed."

```bash
annie agent list                 # no deploy follow-up for payments-api yet
annie agent build "After each prod deploy of payments-api, watch error rate and latency for 30 minutes and post in #payments if either regressed vs the previous hour"
```

Output (abridged): a summary of the draft plus `Review and accept: https://app.anyshift.io/agents/drafts/...`. Give the user that link as printed and say: nothing is live until you accept it. If the output has a warning `Connect it: <url>` (for example Slack not connected), relay it word for word.

## Example 2: one-off check after a merge (MCP)

User merged a fix and says "in 30 min check the checkout 5xx rate is back to normal".

1. `list_agents` shows nothing relevant.
2. Call `schedule_agent_once` with the check text "Check that checkout 5xx rate is back to its pre-incident level; report in the thread" and a 30 minute delay.
3. Reply: "Scheduled for 30 minutes from now; I'll report there. Review it here: <link from the tool result>." Later, `list_agent_runs` then `get_agent_run` shows the result.
