# Annie skill: worked examples

Two end-to-end flows showing the Annie skill in motion. Both assume the agent has the `anyshift` MCP server registered and reachable (see [`reference.md`](./reference.md)).

Tool calls are written as JSON arg objects since they are invoked through the MCP transport, not the CLI. Responses below are abridged.

---

## Example 1: investigate the impact of a recent deploy

**Scenario.** The user pings the agent: _"We just shipped `payments-api` 30 minutes ago. Latency on `checkout-service` jumped at the same time. Is the deploy responsible?"_

**Expected flow:** `track_infrastructure_changes` (find the deploy) → `search_resources_by_term` (locate the changed resource + its relationships) → `inspect_resource_details` (full picture of the suspected dependents) → answer with `hashed_id`s.

### Step 1: scope the deploy

```
track_infrastructure_changes({
  start: "30m",
  end: "now"
})
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
      "hashed_id": "h_svc_payments-api_prod_4a7c",
      "kind": "deploy",
      "summary": "v3.41.2 → v3.41.3"
    }
  ]
}
```

One deploy in the window. Take the `hashed_id` forward.

### Step 2: pull the resource and its relationships

```
search_resources_by_term({
  search_term: "payments-api",
  universe: "CLOUD"
})
```

Result (abridged): the matching service resource (`hashed_id: h_svc_payments-api_prod_4a7c`) with its outbound relationships pointing to dependents, including `h_svc_checkout-service_prod_8d12`.

### Step 3: inspect the suspected dependent in full

```
inspect_resource_details({
  hashed_ids: ["h_svc_checkout-service_prod_8d12"],
  timestamp: "now"
})
```

Returns `checkout-service` with its full properties (image tag, replica count, traffic source) and the explicit relationship pointing back to `payments-api` on the request hot path.

### Step 4: report to the user

> The `payments-api` deploy at 11:02 UTC (`chg_01H...`, v3.41.2 → v3.41.3, `h_svc_payments-api_prod_4a7c`) is the only change in the window. `checkout-service` (`h_svc_checkout-service_prod_8d12`) declares a hot-path call to `payments-api`. The deploy is on the dependency path of the affected service. Roll back as the cheapest test: Annie sees the graph, not the runtime metric, so it cannot confirm causation, only adjacency.

### What the agent did right

- Started window-scoped, then drilled into the one suspected resource. No "snapshot the whole estate" call.
- Carried `hashed_id` through every step instead of re-resolving by friendly name.
- Did not invent a `get_blast_radius` call. Assembled the impact picture from `search_resources_by_term` + `inspect_resource_details` payloads, which is what the surface actually provides.
- Reported adjacency, not causation.

---

## Example 2: walk the timeline of a specific resource

**Scenario.** A teammate asks: _"Our `prod` VPC config looked different yesterday morning than it does now. What specifically changed, and who changed it?"_

**Expected flow:** `search_resources_by_term` (find the VPC's `hashed_id`) → `audit_resource_timeline` (full change history for that resource) → attribute each delta to an actor.

### Step 1: find the VPC's hashed_id

```
search_resources_by_term({
  search_term: "prod",
  resourceType: "AWS_VPC"
})
```

Result (abridged): one match, `vpc_prod_main`, `hashed_id: h_vpc_prod_main_1f3e`.

### Step 2: pull its timeline over the window

```
audit_resource_timeline({
  hashed_id: "h_vpc_prod_main_1f3e",
  start: "2026-05-18T08:00:00Z",
  end: "now"
})
```

Result (abridged):

```json
{
  "events": [
    {
      "id": "chg_01H...A",
      "at": "2026-05-18T14:27:11Z",
      "actor": "louis.fradin@anyshift.io",
      "source": "terraform-cloud:run/run-xyz",
      "kind": "tf-apply",
      "field": "enable_dns_hostnames",
      "from": false,
      "to": true
    },
    {
      "id": "chg_01H...B",
      "at": "2026-05-18T15:02:45Z",
      "actor": "stephane.jourdan@anyshift.io",
      "source": "aws-console",
      "kind": "manual",
      "field": "tags.Owner",
      "from": "platform",
      "to": "sre"
    }
  ]
}
```

Two events, two fields, two different actors, two different paths (Terraform vs console).

### Step 3: report to the user

> Between 2026-05-18 08:00 UTC and now, `vpc_prod_main` (`h_vpc_prod_main_1f3e`) was modified twice. Field `enable_dns_hostnames` flipped false → true via Terraform Cloud at 14:27 UTC by louis.fradin (`chg_01H...A`). Tag `Owner` was changed `platform` → `sre` via the AWS console at 15:02 UTC by stephane.jourdan (`chg_01H...B`). The console edit drifts from Terraform: flag for the next plan-and-apply cycle.

### What the agent did right

- Used `audit_resource_timeline` scoped to one `hashed_id`, not `track_infrastructure_changes` over the whole estate.
- Surfaced the Terraform-vs-console split as the actually-interesting finding, not just the field values.
- Cited the `hashed_id` alongside the friendly name so the reader can re-pull the resource later.
