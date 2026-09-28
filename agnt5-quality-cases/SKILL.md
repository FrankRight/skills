---
name: agnt5-quality-cases
description: Open, update, link, and resolve AGNT5 quality cases - track a behavior regression, eval failure, or production incident from discovery to a verified, shipped fix, linked to the runs, scores, experiments, and datasets that produced it; and automate the topic -> case -> proposal -> eval -> promote cycle with the SDK SelfImprovementLoop. Use when the user says "open a quality case", "track this regression", "mark the fix verified", "link this run to the case", or wants a self-improvement loop.
---

# AGNT5 Quality Cases

A **quality case** tracks a behavior problem from discovery to resolution, linked directly to
the runs/scores/datasets/deployments already in AGNT5 — no separate issue tracker needed.
Open one when an online-eval alert fires (`agnt5-online-evals`), an experiment run regresses
(`agnt5-experiments`), or a production run fails unexpectedly. Studio: **Evaluate → Quality
cases**.

## Anatomy

| Field | Notes |
|---|---|
| `title`, `description` | Summary + detail |
| `category` | `behavior_quality`, `eval_regression`, `production_failure`, `deployment_health`, `runtime_infra`, `support_request`, `release_risk` |
| `severity` | `low`, `medium`, `high`, `critical` |
| `status` | See lifecycle below |
| `source_type` | `eval_run_item`, `eval_alert`, `runtime_run`, `guardrail_decision`, `deployment_failure`, `worker_health`, `runtime_cluster`, `manual`, `support_ticket` |
| `expected_behavior` / `observed_behavior` | What should have happened vs. what did |
| `labels` | Free-form tags |

## Lifecycle

```
open → triaged → investigating → candidate_ready → verified → shipped → closed
```

| Status | Meaning |
|---|---|
| `open` | Newly created, not reviewed |
| `triaged` | Severity/category confirmed |
| `investigating` | Root-cause analysis in progress |
| `candidate_ready` | A fix candidate is ready for eval |
| `verified` | Eval results confirm the fix works |
| `shipped` | Fix deployed to production |
| `closed` | Resolved or won't fix |

## Create

From Claude with the AGNT5 MCP server connected, prefer the
`create_quality_case` / `get_quality_case` / `list_quality_cases` tools over raw curl — same
fields as the REST API, no token handling.

```bash
curl -X POST "https://api.agnt5.com/api/v1/projects/<project-id>/quality/cases" \
  -H "Authorization: Bearer <token>" -H "Content-Type: application/json" \
  -d '{
    "title": "Support agent cites wrong order status",
    "description": "Agent reported order as shipped when still processing for 3 of 50 eval items.",
    "category": "behavior_quality", "severity": "high", "source_type": "eval_run_item",
    "experiment_run_id": "<run-id>",
    "expected_behavior": "Agent returns current status from orders API",
    "observed_behavior": "Agent returned stale cached status",
    "labels": ["orders", "caching"]
  }'
```

Or Studio → **Evaluate → Quality cases → New case** (can link a run/experiment/alert at
creation time). Or from a failing experiment run: Studio run page → select failing items →
**Actions → Create quality case**.

Recurring production behaviors surface first as **behavior topics** (`GET
.../quality/topics`); `POST .../quality/topics/<topic-id>/create-case` turns one into a case
with its representative runs attached.

## Update

```bash
curl -X PATCH "https://api.agnt5.com/api/v1/projects/<project-id>/quality/cases/<case-id>" \
  -H "Authorization: Bearer <token>" -H "Content-Type: application/json" \
  -d '{"status": "investigating", "description": "Root cause: prompt does not refresh order cache. Fixing in PR #42."}'
```

## List and filter

```bash
curl ".../quality/cases?status=open&severity=high" -H "Authorization: Bearer <token>"
curl ".../quality/cases?category=eval_regression" -H "Authorization: Bearer <token>"
curl ".../quality/cases?label=orders" -H "Authorization: Bearer <token>"
```

Combinable filters: `status`, `severity`, `category`, `source_type`, `label`.

## Verify a fix before shipping

Once a fix is `candidate_ready`, build a regression dataset from the case's linked failing
runs with `POST .../quality/cases/<case-id>/regression-dataset`, or from a failing experiment
run with `agnt5 experiments runs regression-dataset` (see `agnt5-experiments`).

Then link it and move the case forward as the gate passes:

```bash
curl -X PATCH ".../quality/cases/<case-id>" -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" -d '{"status": "verified"}'
# ... and "shipped" after the fix deploys
```

Link an experiment, experiment run, or other evidence to an existing case:

```bash
curl -X POST ".../quality/cases/<case-id>/links" -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"link_type": "experiment_run", "target_id": "<run-id>", "metadata": {"experiment_id": "<experiment-id>"}}'
```

## Audit trail

Every status change, note, or link auto-creates a case event. Add one explicitly:

```bash
curl -X POST ".../quality/cases/<case-id>/events" -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"event_type": "note_added", "actor_kind": "user", "body": "Confirmed stale cache reproduces on 10% of cold-start runs."}'
```

Event types: `note_added`, `investigation_added`, `candidate_linked`,
`release_evidence_linked`, `self_improvement_loop_decision`. Payload: `event_type`, `body`,
optional `actor_kind` and `metadata`.

## Automate it: the self-improvement loop

`agnt5.improvement.SelfImprovementLoop` runs the whole cycle against the control plane: pick a
behavior topic → open a case → create a proposal → build a regression dataset → run an eval
experiment against a candidate deployment → decide whether to promote.

```python
# inside an async @workflow or @function handler (ctx = its context)
from agnt5.improvement import (
    AGNT5ImprovementBlocks, ImprovementControlPlaneClient, ImprovementLoopPolicy,
    ImprovementLoopRequest, LoopStatus, SelfImprovementLoop,
)

# Reads AGNT5_CONTROL_PLANE_URL (or AGNT5_API_BASE_URL), AGNT5_PROJECT_ID,
# AGNT5_CONTROL_PLANE_TOKEN (or AGNT5_ACCESS_TOKEN) when args are omitted.
client = ImprovementControlPlaneClient()
loop = SelfImprovementLoop(
    AGNT5ImprovementBlocks(client),
    ImprovementLoopPolicy(min_pass_rate=0.95, max_failed_items=0,
                          require_human_approval=True, allow_auto_promote=False),
)
result = await loop.run(ctx, ImprovementLoopRequest(
    component_name="support_agent",
    metadata={"candidate_deployment_id": "<deployment-id>", "wait_for_eval": True},
))
if result.status == LoopStatus.NEEDS_APPROVAL:
    ...  # a human approves the attempt in Studio
```

`result.status` is one of `NO_TOPIC`, `EVALUATION_PENDING`, `EVALUATION_FAILED`,
`NEEDS_APPROVAL`, `PROMOTION_READY`. Pass `topic_id=` or `case_id=` on the request to skip
topic selection. Keep `require_human_approval=True` unless the user explicitly wants
auto-promotion.

## Source

https://agnt5.com/docs/improve/quality-cases
