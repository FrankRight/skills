---
name: agnt5-online-evals
description: Set up AGNT5 online evals (live experiments) - sample completed production runs of a deployment or component and score them in the background with pinned scorer versions that read only the run's input and output (your deployed @scorer, json_valid, or structured_assertions); declare scorer input requirements on the first publish, then create, edit, publish, pause and resume the live experiment through the control-plane API with a personal API key. Use for "monitor production quality", "score a sample of live runs", "turn on online evals", or editing or pausing an online eval - not for a one-off experiment over a dataset.
---

# AGNT5 Online Evals

> **TypeScript or Go?** This file shows the scorer in Python. Read [references/typescript.md](references/typescript.md) or [references/go.md](references/go.md) for the scorer in those SDKs. The API calls are the same for every SDK.

An online eval is a **live experiment**. AGNT5 watches `run.completed` events that match a source
(deployment, environment, component, tenant), samples a share of them, and runs pinned scorer
versions on each sampled run's input and output. Scoring runs in the background on a worker
deployment you choose; production runs are never blocked or re-run. To score a fixed dataset,
use `agnt5-experiments`.

## What can score online

A production run has an input and an output, but no expected output and no trace for the scorer.
Online scoring accepts only:

| Scorer | Condition |
|---|---|
| Your deployed `@scorer` | Item scope; reads only `request.input` and `request.output`; its name is not a built-in name |
| `json_valid` built-in | none |
| `structured_assertions` built-in | assertions over `input` and `output` only |

Every pinned scorer version must be item-scoped and declare input requirements
`{"requires": ["input", "output"]}` (only `input` and `output` are allowed). Every other built-in
(`exact_match`, `contains`, `correctness`, `llm_judge`, ...) is rejected. A pin that breaks these
rules fails the publish with `publish explicit scorer input requirements before enabling online
scoring` or `online built-in scorer has no supported owned execution route`.

## Auth and IDs

These are control-plane calls. Service keys (`agnt5_sk_...`) get 401 there. Create a **personal
API key** in Studio (Settings → Profile → API keys) and send it as `X-API-KEY`.

```bash
export AGNT5_PERSONAL_API_KEY=<personal-api-key>
API=https://api.agnt5.com/api/v1/projects/<project-id>   # `agnt5 info` shows the project ID
agnt5 deployment list -o json                             # deployment IDs
```

Responses wrap the object as `{"success": true, "message": "...", "data": {...}}`.

## 1. Write and deploy the scorer

```python
from agnt5 import ScorerContext, ScorerRequest, ScorerResult, scorer

@scorer(name="cites_order_id", description="Reply must cite the order ID from the input")
async def cites_order_id(ctx: ScorerContext, request: ScorerRequest) -> ScorerResult:
    # Online, request.input is the run's input and request.output its output.
    # request.expected and request.trace are empty.
    order_id = (request.input or {}).get("order_id", "")
    cited = bool(order_id) and order_id in str(request.output)
    return ScorerResult(score=1.0 if cited else 0.0, passed=cited,
                        explanation=f"Order ID {order_id} {'found' if cited else 'missing'}")
```

Register it (`Worker(..., scorers=[cites_order_id])`) and deploy (`agnt5-deploy`). The deployment
that hosts the scorer must be a worker deployment (not serverless) in the same project. It can be
the deployment that serves production or a separate one. Scorer authoring details are in
`agnt5-scorers`.

## 2. Create the project scorer and publish version 1 with input requirements

Deploying does not create a project scorer. Create one that points at the deployment hosting it,
then publish its first version over REST with the input requirements:

```bash
SCORER_ID=$(curl -s -X POST "$API/scorers" -H "X-API-KEY: $AGNT5_PERSONAL_API_KEY" -H "Content-Type: application/json" \
  -d '{"name": "cites_order_id", "type": "deployed", "deployment_id": "<scorer-deployment-id>", "component_name": "cites_order_id"}' \
  | jq -r .data.id)

VERSION_ID=$(curl -s -X POST "$API/scorers/$SCORER_ID/versions" -H "X-API-KEY: $AGNT5_PERSONAL_API_KEY" -H "Content-Type: application/json" \
  -d '{"scope": "item", "input_requirements": {"requires": ["input", "output"]}}' | jq -r .data.id)
```

- Use the registered scorer name for both `name` and `component_name`.
- Declare the requirements on the **first** publish. The MCP `publish_scorer_version` tool has no
  `input_requirements` field. Publishing again returns `Scorer draft has no unpublished changes`
  until the scorer's draft changes. Editing only the description does not change the draft.
- MCP `create_scorer` (`type: "deployed"`, `deployment_id`, `component_name`) also works for the
  first call (`agnt5 mcp`, see `agnt5-scorers`).

For `json_valid`, seed the built-in scorers once, then take its published version ID:

```bash
curl -s -X POST "$API/scorers/builtins/seed" -H "X-API-KEY: $AGNT5_PERSONAL_API_KEY" > /dev/null
curl -s "$API/scorers?page_size=100" -H "X-API-KEY: $AGNT5_PERSONAL_API_KEY" \
  | jq -r '.data[] | select(.builtin_name == "json_valid") | .current_published_version_id'
```

## 3. Create the live experiment

```bash
curl -s -X POST "$API/eval/online/experiments" -H "X-API-KEY: $AGNT5_PERSONAL_API_KEY" -H "Content-Type: application/json" -d '{
  "name": "support-agent-online",
  "description": "Order-ID citation on live runs",
  "draft": {
    "source": {"event_types": ["run.completed"], "deployment_id": "<production-deployment-id>", "component_name": "support_agent"},
    "scorer_deployment_id": "<scorer-deployment-id>",
    "scorer_version_ids": ["<version-id>"],
    "sample_rate": 0.1,
    "evidence_timeout_ms": 60000
  }
}' | jq '.data | {id, revision, status}'
```

| Field | Rule |
|---|---|
| `source.event_types` | Exactly `["run.completed"]` |
| `source.deployment_id`, `environment_id`, `component_name`, `tenant_id` | Optional filters; omitted ones match everything in the project |
| `scorer_deployment_id` | Worker deployment in this project that hosts the scorers; a deployed scorer's `deployment_id` must equal it |
| `scorer_version_ids` | 1 to 32 published scorer version IDs. They stay pinned when newer versions are published |
| `sample_rate` | Share of matching runs to score, 0 to 1 |
| `evidence_timeout_ms` | How long to wait for the run's input and output, 1 to 600000 |

Creating or editing saves a draft. A draft never scores anything.

## 4. Publish to turn it on

```bash
curl -s -X POST "$API/eval/online/experiments/<id>/publish" -H "X-API-KEY: $AGNT5_PERSONAL_API_KEY" \
  -H "Content-Type: application/json" -d '{"expected_revision": <revision>, "confirm_project_cutover": true}'
```

Publishing checks every pin, freezes a new version and enables it: the response shows
`"enabled": true` and `status` `active` (`pending_application` until the runtime confirms). The
first live experiment in a project moves the whole project from legacy online evals to
runtime-owned ones. Without `"confirm_project_cutover": true` that first publish returns 409.

## Operate it

| Task | Call |
|---|---|
| List | `GET $API/eval/online/experiments` → `items`, `next_cursor`, `online_eval_owner` |
| Read one | `GET $API/eval/online/experiments/<id>` |
| Edit | `PUT $API/eval/online/experiments/<id>` with `{"expected_revision", "name", "description", "draft"}`, then publish again |
| Pause or resume | `POST $API/eval/online/experiments/<id>/enabled` with `{"expected_revision": <revision>, "enabled": false}` |

- Edit, publish, and pause or resume need the current `revision` as `expected_revision`. A stale
  value returns 409: read the experiment again and retry.
- Request bodies are strict. An unknown field (for example `id` or `status` copied from a GET)
  returns 400 `Invalid live experiment configuration`.
- Pause and resume use the last published version. Draft edits stay unpublished.
- `status` reports `draft`, `pending_application`, `active`, `paused`, or a configuration problem.
- A redeploy or promotion creates a new deployment ID (`agnt5-deploy`). Point the scorer at it
  (`PUT $API/scorers/$SCORER_ID` with `{"deployment_id": "<new-id>"}`), publish a new scorer
  version with the input requirements, then update `scorer_deployment_id`, the source
  `deployment_id` and `scorer_version_ids`, and publish the live experiment again.

## Results

Online results are not returned by `agnt5 scores list` or the MCP `list_scores` tool. Read them per
run, in two calls (these two responses are not wrapped in `data`):

```bash
# 1. The run's online-eval delivery carries the decision ID
DECISION=$(curl -s "$API/events/runs/<run-id>/deliveries" -H "X-API-KEY: $AGNT5_PERSONAL_API_KEY" \
  | jq -r '.items[] | select(.subscription_id == "online-eval:<experiment-id>") | .online_decision_id')

# 2. The decision: was the run sampled, and what did each scorer return?
curl -s "$API/eval/online/runs/<run-id>/decisions/$DECISION" -H "X-API-KEY: $AGNT5_PERSONAL_API_KEY" \
  | jq '{selected: .decision.selected, reason: .decision.reason, complete: .scoring.complete,
         scores: [.scoring.result.scores[] | {scorer, score, passed, explanation}]}'
```

- `decision.selected` is `false` for runs the sample skipped; `reason` explains it (for example
  `selected_rate_100`).
- `scoring.complete` turns `true` once the scorers ran, usually within seconds of the run finishing.
  `scoring.result.output` is the output that was scored.
- A scorer that needs an expected value fails online (`passed: false`, explanation such as "no
  expected city"): online runs have no expected output.
- Studio shows the same under **Online evals** in the project sidebar and a run's **Online evals** tab.

When online scores show a regression, save the failing runs as test cases
(`agnt5 datasets add-run <dataset-id> <run-id> --expected-output '...'`) and gate the fix with an
experiment (`agnt5-experiments`).

## Source

https://agnt5.com/docs/improve/online-evals
