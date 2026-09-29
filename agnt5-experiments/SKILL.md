---
name: agnt5-experiments
description: Evaluate AGNT5 components end to end - curate eval datasets (add production runs or manual examples, bulk-upload JSONL/CSV, deduplicate, publish immutable versions), run a component, deployment, or prompt against a dataset version, score every item, compare runs, gate CI on pass-rate thresholds, and turn failed items into a regression dataset; or run lighter-weight inline evals with client.eval()/batch_eval(). Use for "set up an eval", "save this run as a test case", "import these examples from CSV", "publish a dataset version", "compare this prompt/model change against the baseline", "add an eval gate to CI", or "did my change make it better or worse".
---

# AGNT5 Experiments and Datasets

> **TypeScript or Go?** This file shows the Python API. Read [references/typescript.md](references/typescript.md) or [references/go.md](references/go.md) first: same sections, the exact signatures for that SDK, and what it does not support. Known SDK bugs with workarounds are in the `agnt5-sdk-pitfalls` skill.

A **dataset** is a curated set of test cases. An **experiment** binds a target (deployed
component, deployment, or Prompt) to a dataset **version** and a set of scorers (see
`agnt5-scorers`). Each **experiment run** executes the target against every item, scores the
outputs, and produces a comparable pass/fail summary. Datasets and experiment definitions are
versioned and immutable, so two runs over the same dataset version are directly comparable.

| Use `agnt5 experiments` (platform-tracked) | Use `client.eval()`/`batch_eval()` (SDK, code-only) |
|---|---|
| Need to gate CI on a threshold | Quick local regression check during development |
| Comparing across deployments/dataset versions over time | One-off quality check, no need to persist results |
| Need Studio visibility, annotations, rescoring | Don't need the full platform overhead |

All dataset, experiment, report, and score commands take `--format table|json|jsonl` (jsonl
for piping to `jq`) and `--project-id <uuid>` (target a project other than the current one).

## 1. Build a dataset

Every dataset has one mutable **draft** (edits land here) and zero or more immutable
**versions** — experiments always run against a version, never the draft.

Item fields: `input` (JSON, required), `expected_output` (JSON, optional — scorers compare
against it), `metadata` (your labels), `events` (trace events, captured when importing from a
run — needed by trace-level scorers), `split` (e.g. `train`/`test`).

```bash
agnt5 datasets create --name support-agent-golden-set \
  --description "Curated support conversations with verified answers"
agnt5 datasets list --search support-agent     # find the dataset ID later

# From a production run — captures input/output/trace events
agnt5 inspect runs ls                          # find the run ID first
agnt5 datasets add-run <dataset-id> <run-id> \
  --expected-output '{"answer": "Refund issued within 5 business days"}'
  # add --metadata '{"source":"prod"}' on create/add-run/add-example/publish to label items

# Manual example
agnt5 datasets add-example <dataset-id> \
  --input '{"message": "Where is my order #4512?"}' \
  --expected-output '{"intent": "order_status"}'

# Bulk JSONL — one item per line: {"input":…, "expected_output":…, "metadata":…, "events":…, "split":…}
agnt5 datasets upload <dataset-id> --file examples.jsonl
cat examples.jsonl | agnt5 datasets upload <dataset-id>     # or pipe from stdin

# Bulk CSV — map columns to fields
agnt5 datasets upload-csv <dataset-id> --file examples.csv \
  --input-column question --expected-output-column answer --split-column split
  # --partial defaults true (valid rows import even if some fail); --partial=false = all-or-nothing
```

Other CSV flags: `--metadata-column`, `--events-column`, `--tags-column`,
`--source-run-id-column`, `--source-ref(-column)`, `--delimiter`, `--no-header` (columns by
zero-based index). Or Studio → project → **Evaluate → Datasets**. From Claude with the AGNT5
MCP connected: `create_eval_dataset`, `add_run_to_dataset_draft`,
`add_manual_dataset_example`, `import_dataset_csv`, `preview_dataset_dedup` /
`apply_dataset_dedup`, `publish_dataset_version`.

### Deduplicate and publish

```bash
agnt5 datasets dedup preview <dataset-id> --include-preview        # see duplicate groups first
agnt5 datasets dedup apply <dataset-id>                            # keep earliest, remove rest
agnt5 datasets dedup apply <dataset-id> --remove-item-id <item-id> # or target specific items

agnt5 datasets publish <dataset-id> --description "Adds 40 cancellation cases from last week"
agnt5 datasets versions list <dataset-id>
agnt5 datasets versions export <dataset-id> <version-id>                             # as JSONL
agnt5 datasets versions compare <dataset-id> <base-version-id> <compare-version-id>  # added/removed/changed
agnt5 datasets versions restore-draft <dataset-id> <version-id>                      # reset draft
agnt5 datasets examples list <dataset-id> --version 2 --include-payload
```

The draft stays editable after publishing — keep curating, publish again when ready.

## 2. Create and run an experiment

```bash
agnt5 experiments create --name support-agent-quality \
  --dataset-id <dataset-id> --dataset-version-id <dataset-version-id> \
  --target-type component --deployment-id <deployment-id> \
  --component-name support_agent --component-type agent \
  --builtin-scorer json_valid --builtin-scorer correctness \
  --config '{"passed_threshold":1}'

# Target a Prompt instead of a component:
agnt5 experiments create --name support-prompt-comparison \
  --dataset-id <dataset-id> --dataset-version-id <dataset-version-id> \
  --target-type prompt --deployment-id <deployment-id> \
  --prompt-id <prompt-id> --prompt-version-id <prompt-version-id> \
  --builtin-scorer correctness

agnt5 experiments run <experiment-id>                          # fire and forget
agnt5 experiments run <experiment-id> --wait --timeout 15m      # block, non-zero exit if gate fails
agnt5 experiments run <experiment-id> --deployment-id <candidate-id> --name "pr-1234"  # compare a candidate
```

Required for create: `--name`, `--dataset-id`, `--dataset-version-id`, a target
(`--target-type component|deployment|prompt` with the matching IDs), and at least one
`--builtin-scorer <name|json>` or `--scorer-id <uuid>` (both repeatable). `run` also takes
`--experiment-version-id` and `--config`. Full flag list: `agnt5 experiments create --help`.

## 3. Inspect and compare

```bash
agnt5 experiments runs list <experiment-id>
agnt5 experiments runs show <run-id>            # status, pass rate, per-scorer aggregates
agnt5 reports summary <run-id>                  # same, standalone
agnt5 reports failures <run-id>                 # failed items only
agnt5 scores list --run-id <run-id>
agnt5 experiments runs cancel <experiment-id> <run-id>
agnt5 experiments runs compare <base-run-id> <compare-run-id>   # score movement + items that flipped
```

## Gate CI on results

```bash
agnt5 experiments run <experiment-id> --deployment-id "$CANDIDATE_DEPLOYMENT_ID" \
  --name "ci-$GIT_SHA" --wait --fail-on-gate
agnt5 reports wait <run-id> --timeout 15m       # or wait on an already-started run
agnt5 reports ci <run-id>                       # print the gate verdict
agnt5 reports export <run-id> --artifact-format csv --out-file eval-results.csv
```

| Exit code | Meaning |
|---|---|
| `0` | Gate passed |
| `2` | CI gate failed (pass rate below threshold) |
| `3` | Run failed or cancelled |
| `4` | Wait timed out |

`--fail-on-gate` defaults to `true`; pass `--fail-on-gate=false` to inspect the result
yourself instead of failing the pipeline. The threshold lives on the experiment's
`--config '{"passed_threshold": <0..1>}'`, not in the pipeline script.

## Turn failures into a regression test

```bash
agnt5 experiments runs regression-dataset <run-id> --name support-agent-regressions --start-run --wait
# rerun against a specific fix candidate:
agnt5 experiments runs regression-dataset <run-id> --name support-agent-regressions \
  --start-run --deployment-id <candidate-deployment-id> --wait
# only specific failed items:
agnt5 experiments runs regression-dataset <run-id> --name order-bugs --run-item-id <item-id> --run-item-id <item-id>
```

Builds a dataset from the failed items, creates a regression experiment over it, and (with
`--start-run`) kicks off the first rerun immediately. Feeds `agnt5-quality-cases`.

## Rescore without re-executing

After fixing a scorer's logic/threshold, rescore a **completed** run's existing outputs:

```bash
curl -X POST "https://api.agnt5.com/api/v1/projects/<project-id>/experiments/<experiment-id>/runs/<run-id>/rescore" \
  -H "Authorization: Bearer <token>" -H "Content-Type: application/json" \
  -d '{"reason": "updated_rubric", "replace_latest": true}'
agnt5 reports ci <run-id>     # check the updated verdict
```

Optional: `scorer_version_ids`, `run_item_ids` to narrow scope, `idempotency_key`.

## Annotate run items (human label vs. scorer verdict)

```bash
curl -X POST ".../experiments/<experiment-id>/runs/<run-id>/items/<item-id>/annotations" \
  -H "Authorization: Bearer <token>" -H "Content-Type: application/json" \
  -d '{"name": "human_label", "label": "pass", "metadata": {"reviewer": "alice"}}'
```

Stored separately from scores, used for meta-evaluation (scorer accuracy vs. human judgment).

## Inline evals from code: `client.eval()` / `client.batch_eval()`

Needs a running worker (`AGNT5_GATEWAY_URL`, falls back to `https://gw.agnt5.com`):

```python
from agnt5 import Client
from agnt5.eval import Correctness

client = Client()
result = client.eval(component="support_agent", component_type="agent",
                      input_data={"message": "Where is my order #1234?"},
                      expected="Your order #1234 is in transit", scorers=[Correctness()])
print(result.passed, result.output)
```

```python
from agnt5 import Client, BatchEvalItem

client = Client()
result = client.batch_eval(
    component="support_agent", component_type="agent",
    items=[
        BatchEvalItem(input={"message": "Where is my order #1234?"}, expected="...", item_id="order-status"),
        BatchEvalItem(input={"message": "Cancel order #5678"}, expected="...", item_id="order-cancel"),
    ],
    scorers=["exact_match"],          # strings, preset instances, or LLMJudge(...) — mix freely
    max_concurrency=10,               # start at 3-5 during development
    timeout=60.0,
)
print(f"Pass rate: {result.pass_rate:.0%}")
for item in result.results:
    print(item.item_id, "PASS" if item.passed else "FAIL", item.duration_ms)
```

Item input forms (mix freely): plain dicts + separate `expected` list; dicts with
`input`/`expected` keys; `BatchEvalItem(input, expected?, item_id?)` for full control. Both
methods accept `deployment_id=` to evaluate a specific deployment.

`BatchEvalResult`: `batch_id`, `status` (`completed`/`partial_failure`/`failed`), `results`,
`stats`, `pass_rate`, `passing_items()`, `failing_items()` (scoring failures),
`failed_items()` (evaluation errors — distinct from scoring failures, check both).

## Source

https://agnt5.com/docs/improve/datasets · https://agnt5.com/docs/improve/experiments · https://agnt5.com/docs/improve/batch-eval
