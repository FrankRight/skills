---
name: agnt5-online-evals
description: Set up AGNT5 online evals - an evaluation policy that samples completed production runs (uniform rate plus boost rules for slow runs), scores them asynchronously with a published scorer, and fires an alert when the pass rate drops below a floor. Use for "monitor production quality", "score a sample of live runs", "alert me when quality drops", or editing/disabling an existing policy or alert - not for a one-off experiment.
---

# AGNT5 Online Evals

Online evals measure production behavior without blocking user traffic: attach a published
**scorer** (see `agnt5-scorers`) to a **deployment**, AGNT5 samples completed runs and scores
them in the background, then Studio shows live aggregates, sample decisions, and alerts.

**Prerequisites**: a deployment receiving production runs, at least one enabled scorer with a
published version (built-in, or a custom one from `agnt5-scorers`; from Claude the AGNT5 MCP
tools `create_scorer` / `publish_scorer_version` do this), developer access to the project.

## Set up in Studio (fastest path)

1. Open the deployment → **Quality** tab.
2. Choose a **Scorer**, set **Sample %** for ordinary runs.
3. Set **Slow-run %** and **Slow-run threshold ms** to over-sample slow runs.
4. Set **Pass-rate floor** + **Min count** for an optional alert.
5. **Preview** — estimates observed/selected runs, scorer jobs, alert status.
6. **Enable** — creates the policy and alert.

Edit the policy table to disable/enable a policy or alert, or **edit** to create a new policy
version with updated sampling/scorers.

## Preview via API (no side effects)

```bash
curl -X POST "https://api.agnt5.com/api/v1/projects/<project-id>/eval/online/preview" \
  -H "Authorization: Bearer <token>" -H "Content-Type: application/json" \
  -d '{
    "lookback_seconds": 86400, "max_runs": 1000,
    "policy": {
      "mode": "async_online", "binding_scope": "deployment", "deployment_id": "<deployment-id>",
      "sampling_config": {
        "type": "uniform", "rate": 0.02,
        "boost": [{"field": "duration_ms", "op": "gte", "value": 30000, "rate": 0.10}]
      },
      "scorers": [{"scorer_id": "<scorer-id>", "scorer_version_id": "<scorer-version-id>",
                   "scope": "run", "ordinal": 1, "required": true, "threshold": 0.9, "weight": 1}]
    },
    "alert": {
      "name": "Production quality drop", "severity": "warning", "deployment_id": "<deployment-id>",
      "scorer_id": "<scorer-id>", "scorer_version_id": "<scorer-version-id>",
      "window_seconds": 1800, "metric": "pass_rate", "operator": "lt", "threshold": 0.9,
      "min_count": 50, "action_type": "notify"
    }
  }'
```

Response reports job volume (not cost — scorer cost depends on model/provider config and
custom scorer runtime).

## Enable via API

```bash
# 1. Create the policy — body is the "policy" object from the preview above
curl -X POST "https://api.agnt5.com/api/v1/projects/<project-id>/eval/policies" \
  -H "Authorization: Bearer <token>" -H "Content-Type: application/json" -d @policy.json

# 2. Create the alert — body is the "alert" object from the preview, plus
#    "evaluation_policy_id": "<policy-id returned by step 1>"
curl -X POST "https://api.agnt5.com/api/v1/projects/<project-id>/eval/online/alerts" \
  -H "Authorization: Bearer <token>" -H "Content-Type: application/json" -d @alert.json
```

## Operate a running setup

| Task | Endpoint |
|---|---|
| List policies | `GET /eval/policies?mode=async_online&deployment_id=<id>` |
| Disable a policy | `PATCH /eval/policies/<policy-id>` body `{"enabled": false}` |
| New policy version | `PATCH /eval/policies/<policy-id>` with new `sampling_config`/`scorers` |
| List sample decisions | `GET /eval/online/sample-decisions?deployment_id=<id>` |
| List live scores | `GET /eval/scores?source=live&deployment_id=<id>` |
| Live score aggregate | `GET /eval/online/scores/aggregate?source=live&deployment_id=<id>` |
| Disable an alert | `PATCH /eval/online/alerts/<alert-id>` body `{"enabled": false}` |

(All under `https://api.agnt5.com/api/v1/projects/<project-id>`.) Cross-reference live scores
against `agnt5 scores list --root-run-id <id>` / `--session-id <id>` from `agnt5-scorers`.

When an alert fires, open a quality case to track the investigation — see
`agnt5-quality-cases`.

## Source

https://agnt5.com/docs/improve/online-evals
