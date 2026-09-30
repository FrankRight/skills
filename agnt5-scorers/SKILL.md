---
name: agnt5-scorers
description: Score AGNT5 component outputs - pick built-in deterministic checks (exact_match, json_schema, tool_called, step_efficiency, ...) with the config each one requires, built-in LLM-as-judge presets (correctness, faithfulness, goal_success, agent_judge) and their provider/model naming, or write and deploy a custom @scorer (ScorerContext/ScorerRequest) and register it as a project scorer (MCP create_scorer + publish_scorer_version) to get the scorer ID experiments need; trace assertions for glassbox checks; inspecting scores. Use when defining what "correct" means for an experiment or online eval, writing a custom scorer, fixing a config_error from a built-in scorer, or reading scores/evidence for a run.
---

# AGNT5 Scorers

> **TypeScript or Go?** This file shows the Python API. Read [references/typescript.md](references/typescript.md) or [references/go.md](references/go.md) first: same sections, the exact signatures for that SDK, and what it does not support.

A **scorer** returns a score (0.0-1.0), a pass/fail verdict, and an optional explanation for
one component output. `agnt5-experiments` runs scorers against every item of a dataset;
`agnt5-online-evals` runs them on sampled production runs.

## Three scorer classes — pick the cheapest that works

| Class | Needs deployment? | When to use |
|---|---|---|
| **Built-in deterministic** | No — AGNT5-owned logic | Exact/structural checks: format, substring, tool calls, limits |
| **Built-in LLM-as-judge** | No — AGNT5-owned, configurable model/rubric | Subjective quality: correctness, helpfulness, tone |
| **Custom (`@scorer`)** | Yes — registers and deploys with your worker | Anything the built-ins can't express |

Default to deterministic first. Reach for LLM-as-judge only when there's no deterministic
check that captures "correct." Only write custom code when neither built-in covers it.

## Built-in deterministic scorers

Output scorers: `exact_match`, `contains`, `regex_match`, `json_valid`, `json_schema`,
`numeric_range`, `levenshtein`, `structured_assertions`.

Trace scorers (need `events` on the dataset item — captured when importing from a run):
`tool_called` / `tool_not_called`, `tool_sequence` / `tool_sequence_in_order` /
`tool_sequence_exact` / `tool_sequence_any_order`, `tool_trajectory`, `tool_params_match`,
`max_tool_calls` / `max_llm_calls`, `max_tokens`, `duration_under`, `no_errors`,
`state_equals`, `tool_failure_recovered`, `step_efficiency`, `plan_quality`, `plan_adherence`.

```bash
agnt5 experiments create --name support-agent-quality \
  --dataset-id <dataset-id> --dataset-version-id <dataset-version-id> \
  --deployment-id <deployment-id> --component-name support_agent --component-type agent \
  --builtin-scorer json_valid \
  --builtin-scorer '{"name":"tool_called","config":{"tool":"search_orders"}}' \
  --builtin-scorer '{"name":"max_llm_calls","config":{"max":5}}'
```

A bare name works only for built-ins without required config: `exact_match`, `json_valid`,
`levenshtein`, `no_errors`, `tool_failure_recovered`, `step_efficiency`, `plan_quality`,
`plan_adherence`, `correctness`, `goal_success`. Every other built-in needs the
`{"name": ..., "config": {...}}` form:

| Built-in | Required config |
|---|---|
| `contains`, `regex_match` | `pattern` (non-empty string) |
| `json_schema` | `schema` (JSON Schema object) |
| `numeric_range` | `min` and/or `max` (numbers) |
| `structured_assertions` | `assertions` (non-empty array) |
| `tool_called`, `tool_not_called` | `tool` |
| `tool_sequence`, `tool_sequence_in_order`, `tool_sequence_exact`, `tool_sequence_any_order`, `tool_trajectory` | `tools` (array of tool names) |
| `tool_params_match` | `tool` and `params` (object) |
| `max_tool_calls`, `max_llm_calls`, `max_tokens` | `max` |
| `duration_under` | `max_ms` |
| `state_equals` | `name` and `expected` |
| `llm_judge` | `criteria` (or `prompt_template`) and `model` |
| `agent_judge` | `model` |
| `faithfulness` | `context_fields` (selectors starting with `input.`, `output.` or `expected.`) |

`agnt5 experiments create` rejects a bare name for these. `client.eval()` / `batch_eval()` send it
without config, and the check fails or checks nothing: `contains` returns `config_error` with
`Invalid contains config: invalid type: null`. Pass the same `{name, config}` object in code:
`scorers=[{"name": "contains", "config": {"pattern": "in transit"}}]`.

## Built-in LLM-as-judge scorers

`llm_judge` (generic — you supply criteria/rubric, optional `choice_scores`), `correctness`
(matches input + expected), `faithfulness` (no hallucination vs. configured context fields),
`goal_success` (did the run achieve the user's goal), `agent_judge` (judges over trace and
tool-call evidence). Needs a provider credential configured as a project secret (e.g.
`OPENAI_API_KEY`).

```bash
--builtin-scorer correctness \
--builtin-scorer '{"name":"llm_judge","config":{"criteria":"Is the response concise and actionable?","provider":"openai","model":"gpt-4o-mini"}}'
```

Judges run in your worker. In raw JSON configs, put the provider in `provider` and a bare name in
`model`: a Python worker sends `"model": "openai/gpt-4o-mini"` to OpenAI as the model name and the
judge scores 0. The SDK preset objects below take `provider/model` instead; a bare name there
means OpenAI.

### SDK evaluator presets (for `client.eval()`/`batch_eval()`, see `agnt5-experiments`)

```python
from agnt5.eval import Correctness, Helpfulness, Faithfulness

scorers = [
    Correctness(),
    Helpfulness(model="openai/gpt-4o"),
    Faithfulness(context_fields=["input.retrieved_chunks"]),   # selectors start with input., output. or expected.
]
```

Full preset list: `Correctness`, `Faithfulness`, `Helpfulness`, `Coherence`, `Conciseness`,
`ResponseRelevance`, `InstructionFollowing`, `GoalSuccess`, `Refusal`, `Harmfulness`,
`Stereotyping`, plus the generic `LLMJudge(criteria=...)`. All accept `model` (default
`openai/gpt-4o-mini`), `temperature` (default `0.0`), `threshold` (default `0.7`), and
`include_input` — default `True` except for `Faithfulness`, `Coherence`, `Conciseness` and
`LLMJudge`, which default to `False`.

Judge failures are scores, not exceptions: a provider error comes back as `score=0.0,
passed=False, explanation="LLM call failed: …"`. The default `temperature=0.0` is one such
error on `openai/gpt-6*` models (they reject any temperature), so a gpt-6 judge silently
scores everything 0 — keep judges on a non-gpt-6 model (the Go SDK's built-in judges and the
Python presets share this default).

## Custom scorers

A deployable scorer takes `(ctx: ScorerContext, request: ScorerRequest)` and returns a
`ScorerResult`. Import from the top-level `agnt5` package — `agnt5.eval.scorer` is the legacy
local-only registry and never deploys.

```python
from agnt5 import ScorerContext, ScorerRequest, ScorerResult, scorer

@scorer(name="cites_order_id", description="Reply must cite the order ID from the input")
async def cites_order_id(ctx: ScorerContext, request: ScorerRequest) -> ScorerResult:
    order_id = (request.input or {}).get("order_id", "")
    cited = bool(order_id) and order_id in str(request.output)
    return ScorerResult(score=1.0 if cited else 0.0, passed=cited,
                        explanation=f"Order ID {order_id} {'found' if cited else 'missing'}")
```

- `@scorer` kwargs: `name`, `description`, `scope` (`item` default, `run`, `trace`, `span`,
  `session`, `fleet_run`), `depends_on=["other_scorer"]`. Bare `@scorer` also works.
- `ScorerRequest` fields: `output`, `expected`, `input`, `trace` (list of trace events),
  `config`, `peer_scores`, `trace_eval_context`. Helpers: `get_config(key, default)`,
  `get_tool_calls()`, `get_tool_call_names()`, `get_total_tokens()`, `get_trace_events(type)`.
- `ScorerResult(score, passed, label=None, explanation=None, metadata=None)`; shortcuts
  `ScorerResult.pass_result("why")` / `ScorerResult.fail_result("why")`.
- Compose scorers: declare `depends_on=[...]`, then read earlier results with
  `ctx.peer_scores("scorer_name")`.
- Custom scorers register and deploy with your worker like any component:
  `Worker(..., scorers=[cites_order_id])`. In explicit mode only the scorers you list are
  registered (the built-in deterministic and judge names are added automatically);
  `Worker(auto_register=True)` registers every `@scorer` it discovers.

Test locally without deploying:

```python
import asyncio
from agnt5 import ScorerRequest, run_scorer

print(asyncio.run(run_scorer("cites_order_id",
      ScorerRequest(output="Refund for order 42 issued", input={"order_id": "42"}))))
```

`run_scorer` resolves your `@scorer`s, `structured_assertions` and the judge built-ins. Other
deterministic built-ins run in the worker's native core, so `run_scorer("exact_match", ...)` raises
`ValueError: Scorer not found`; call the `agnt5.eval` functions instead.

`agnt5.eval` also ships the deterministic scorers as plain functions for tests and for use
inside custom scorers: `exact_match(input, case_sensitive=None)`, `contains(input, pattern)`,
`regex_match(input, pattern)`, `json_valid(input)`, `json_schema(input, schema)`,
`numeric_range(input, min=, max=)`, `levenshtein(input, threshold=)`,
`structured_assertions(input, config)`. All take `ScorerInput(output=, expected=, input=,
trace=)` and return the Rust `agnt5.eval.ScorerResult` (`.score`, `.passed`, `.explanation`).
Ad-hoc judge: `await llm_judge(output, LLMJudgeConfig(criteria=..., model=...), expected=...,
input_data=...)`. Trace helpers: `extract_tool_calls(events)`, `tool_call_names(calls)`,
`tool_trajectory_exact` / `_in_order` / `_any_order(actual, expected)`.

Two `ScorerResult` types exist: `agnt5.ScorerResult` (Python dataclass, what a deployable
`@scorer` returns; also exported as `agnt5.eval.ScorerResultPy`) and `agnt5.eval.ScorerResult`
(the Rust type the local functions return). Bridge them with
`ScorerResult(score=r.score, passed=r.passed, explanation=r.explanation)`.

### Get a scorer ID for experiments

Deploying does not create a project scorer, so a custom scorer has no scorer ID yet. Create one
with the AGNT5 MCP tools, then pass its `id` to `--scorer-id`:

1. `create_scorer` with `type: "deployed"`, `deployment_id` (the deployment that registered the
   scorer), `component_name: "cites_order_id"`, and `name`.
2. `publish_scorer_version` with the returned scorer ID.
3. `agnt5 experiments create ... --scorer-id <scorer-id>` (repeatable).

The component ID from `agnt5 components` is not a scorer ID: `experiments create` accepts it and
`experiments run` then fails with 404. An online eval also needs input requirements on the first
published version, which only the REST API can set (`agnt5-online-evals`).

The MCP tools come from `agnt5 mcp`, an MCP server over stdio that uses your CLI login. Register it
with your MCP client, for example `claude mcp add agnt5 -- agnt5 mcp` in Claude Code.

## Trace assertions (glassbox testing)

Assert on *execution behavior* rather than output content, inside a custom scorer (no CLI
scorer name maps to these):

```python
from agnt5 import ScorerContext, ScorerRequest, ScorerResult, scorer
from agnt5.eval import ScorerInput, TraceAssertion, trace_scorer

@scorer(name="efficiency_check", scope="trace")
async def efficiency_check(ctx: ScorerContext, request: ScorerRequest) -> ScorerResult:
    result = trace_scorer(ScorerInput(output=request.output, trace=request.trace or []), [
        TraceAssertion.max_tokens(2000),
        TraceAssertion.max_lm_calls(4),
        TraceAssertion.no_errors(),
        TraceAssertion.duration_under(15000),
    ])
    return ScorerResult(score=result.score, passed=result.passed, explanation=result.explanation)
```

Assertions: `max_tokens(n)`, `max_lm_calls(n)`, `no_errors()`, `duration_under(ms)`,
`event_sequence([...])`, `step_memoized(name)`, `event_count(type, min)`. `trace_scorer()`
returns score = proportion of assertions passed.

## Inspect scores

```bash
agnt5 scores list --run-id <run-id>
agnt5 scores list --run-id <run-id> --scorer-id <scorer-id>
agnt5 scores list --component-name support_agent --since 2h
agnt5 scores list --root-run-id <root-run-id>
agnt5 scores evidence <score-id> --include scorer_input,scorer_output,evidence
```

Filters on `scores list`: `--run-id`, `--run-item-id`, `--scorer-id`, `--scorer-version-id`,
`--subject-type`, `--subject-id`, `--session-id`, `--root-run-id`, `--component-name`,
`--component-type`, `--journal-id`, `--span-id`, `--since`, `--until`. The MCP equivalents are
`list_scores` and `get_score_evidence`. For online-eval results see `agnt5-online-evals`.

## Source

https://agnt5.com/docs/improve/scorers
