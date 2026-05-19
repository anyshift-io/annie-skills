# Annie skill: worked examples

Two end-to-end flows showing the Annie skill in motion. Both assume Annie CLI is installed and authenticated (see [`reference.md`](./reference.md)).

---

## Example 1: investigate the impact of a recent deploy

**Scenario.** The user pings the agent: *"We just shipped `payments-api` 30 minutes ago. Latency on `checkout-service` jumped at the same time. Is the deploy responsible?"*

**Expected flow:** `get_recent_changes` (find the deploy) → `get_blast_radius` (does it touch `checkout-service`?) → answer with stable IDs.

### Step 1: scope the deploy

```bash
annie mcp call get_recent_changes \
  --since "30m" \
  --resource "payments-api"
```

Result (abridged):

```json
{
  "changes": [
    {
      "id": "chg_01H...",
      "at": "2026-05-19T11:02:18Z",
      "actor": "ci-bot",
      "source": "github-actions:payments-api/.github/workflows/deploy.yml",
      "resource_id": "svc_payments-api_prod",
      "kind": "deploy",
      "summary": "v3.41.2 → v3.41.3"
    }
  ]
}
```

One deploy in the window. Take the `resource_id` forward.

### Step 2: get the blast radius

```bash
annie mcp call get_blast_radius \
  --resource "svc_payments-api_prod" \
  --change_type "modify"
```

Result (abridged):

```json
{
  "impacted": [
    {"resource_id": "svc_checkout-service_prod", "reason": "calls payments-api on hot path", "score": 0.92},
    {"resource_id": "svc_order-confirmation_prod", "reason": "async via SQS, replayable", "score": 0.41},
    {"resource_id": "dashboard_revenue_grafana", "reason": "reads from payments-api metrics", "score": 0.18}
  ]
}
```

`checkout-service` is the top-scored impacted resource. The deploy *can* be responsible.

### Step 3: report to the user

> The `payments-api` deploy at 11:02 UTC (`chg_01H...`, v3.41.2 → v3.41.3) is the only change in the window. `checkout-service` is the top impacted dependent (score 0.92, hot-path call). The correlation is consistent with the deploy being responsible. Roll back as the cheapest test: `annie` cannot confirm causation, only adjacency.

### What the agent did right

- Filtered `get_recent_changes` by `resource` and a short window, not the whole estate.
- Took the stable `resource_id` from step 1 into step 2, instead of re-querying by friendly name.
- Reported adjacency, not causation. Annie sees the graph, not the runtime metric.

---

## Example 2: diff infra state between two points in time

**Scenario.** A teammate asks: *"Our `prod` VPC config looked different yesterday morning than it does now. What specifically changed, and who changed it?"*

**Expected flow:** `get_temporal_diff` (find the deltas) → `get_recent_changes` (attribute each delta to an actor).

### Step 1: get the diff

```bash
annie mcp call get_temporal_diff \
  --at_a "2026-05-18T08:00:00Z" \
  --at_b "now" \
  --scope '{"env": "prod", "type": "aws_vpc"}'
```

Result (abridged):

```json
{
  "added": [],
  "removed": [],
  "modified": [
    {
      "resource_id": "vpc_prod_main",
      "fields": ["enable_dns_hostnames", "tags.Owner"]
    }
  ]
}
```

One resource changed, two fields.

### Step 2: attribute the changes

```bash
annie mcp call get_recent_changes \
  --resource "vpc_prod_main" \
  --since "2026-05-18T08:00:00Z" \
  --until "now"
```

Result (abridged):

```json
{
  "changes": [
    {
      "id": "chg_01H...A",
      "at": "2026-05-18T14:27:11Z",
      "actor": "louis.fradin@anyshift.io",
      "source": "terraform-cloud:run/run-xyz",
      "kind": "tf-apply",
      "summary": "enable_dns_hostnames: false → true"
    },
    {
      "id": "chg_01H...B",
      "at": "2026-05-18T15:02:45Z",
      "actor": "stephane.jourdan@anyshift.io",
      "source": "aws-console",
      "kind": "manual",
      "summary": "tags.Owner: 'platform' → 'sre'"
    }
  ]
}
```

Two changes, both yesterday afternoon, by two different actors via two different paths (Terraform vs console).

### Step 3: report to the user

> Between 2026-05-18 08:00 UTC and now, `vpc_prod_main` was modified twice on two fields. Field `enable_dns_hostnames` was flipped false → true by louis.fradin via Terraform Cloud at 14:27 UTC (`chg_01H...A`). Tag `Owner` was changed `platform` → `sre` by stephane.jourdan via the AWS console at 15:02 UTC (`chg_01H...B`). The console edit drifts from Terraform: flag for the next plan-and-apply cycle.

### What the agent did right

- Scoped `get_temporal_diff` to a single resource type and env, not the whole graph.
- Followed up with `get_recent_changes` filtered to the modified resource, instead of paging the whole window.
- Surfaced the Terraform-vs-console split as the actually-interesting finding, not just the field values.
