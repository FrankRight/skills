---
name: agnt5-observe
description: Look up AGNT5 runtime data with the CLI, MCP (agnt5 mcp), or Studio - list and describe runs, find, follow or cancel a run that hasn't finished, print execution traces (steps, tool calls, LLM spans), read run logs and deployment logs, read throughput/latency/cost metrics, instrument your own code (ctx.logger attributes, agnt5.tracing spans, get_logger / set_log_level, AGNT5_DEBUG), and control automatic OpenAI / OpenAI Agents SDK / Google ADK call capture (AGNT5_CAPTURE*). Use for "show me recent failed runs", "tail the logs", "print the trace for run X", "add a span or log attribute", "why aren't my OpenAI calls in the trace". For a root-cause analysis of one bad run use agnt5-run-investigation; for recurring issues across runs use agnt5-pattern-analysis.
---

# AGNT5 Observe

> **TypeScript or Go?** The commands here apply to every language; the language-specific parts (setup, packaging, runtime behaviour) are in [references/typescript.md](references/typescript.md) and [references/go.md](references/go.md).

A **run** is one execution of a workflow/function/agent. A **trace** is its full execution
timeline — a span tree (workflow → steps → function calls → agent iterations → LLM calls →
tool calls). **Logs** are structured output scoped to the run that produced them.

## Quick look at a failed run

For a full root-cause analysis with evidence, use `agnt5-run-investigation` instead.

```bash
agnt5 inspect runs ls --status failed --since 1h
agnt5 inspect runs describe <runId>
agnt5 inspect trace -r <runId>
agnt5 inspect logs -r <runId>
```

These need CLI `20260930-a31e8d` or later (`agnt5 version update`); on older CLIs
`inspect logs` answers `403 … Workspace context is required for this action`, so read the logs
with the MCP tool `get_run_logs` (below) or on the run page in Studio.

Run the commands inside the linked project directory; they read that project.

## AGNT5 MCP tools

The CLI ships an MCP server over stdio that uses your `agnt5 auth login` session. Register it
with your MCP client, e.g. Claude Code:

```bash
claude mcp add agnt5 -- agnt5 mcp
```

Register it without `--services`: `get_run_events` and `get_run_input_output` are in no
category, so any `--services` list hides them. `get_run_events` needs CLI `20261002-b0b8c8` or
later. The tools take IDs rather than a project directory: `list_runs`, `get_run_summary`,
`get_run_events` (a run's steps, LLM and tool calls, attempts and errors, in order),
`get_run_logs`, `get_run_input_output`, `list_deployments`, and more. There are no MCP tools
for traces or metrics: use `agnt5 inspect trace` and Studio.

## Runs

```bash
agnt5 inspect runs ls
agnt5 inspect runs ls --status failed --since 1h
agnt5 inspect runs ls --component my_workflow -w          # watch mode, refresh every 2s
agnt5 inspect runs ls --output json | jq '.data[].run_id'
agnt5 inspect runs describe <runId>
```

| Flag | Description |
|---|---|
| `--status` | `completed`, `failed`, `cancelled` (the CLI also offers `running` and `pending`, but they match nothing: unfinished runs are not listed) |
| `--component <name>` / `--component-type <type>` | Filter by component |
| `--since <window>` | e.g. `1h`, `24h`, `7d` |
| `--limit <n>` | Default 20 |
| `-w` | Watch mode |
| `--output json` / `-o json` | Machine-readable |

Each run records: run ID, component name+type, status, duration, queue time, step count,
retries, LLM call count, LLM cost, error (on failure). Two gaps today: `step_count` is 0 for
TypeScript and Go runs and for Python workflows, so count steps from the trace or the run's
events; and a failed Go workflow's summary has no error type or message, so read the error
from its trace and logs. `describe` also prints next-step
commands (`agnt5 inspect logs -r ...`, `agnt5 inspect trace -r ...`).

### Runs that haven't finished

Run summaries are written when a run ends. The CLI also reads queued, running and paused runs
from the gateway: `agnt5 inspect runs ls` lists them (`--status paused`, `--status running`),
and `describe` and `trace` work on them; `describe` notes that step, retry and LLM totals
arrive once the run ends. The MCP `list_runs` lists them too, on its first page;
`get_run_summary` reads them without the totals, and `get_run_events` returns their events so
far. CLIs older than `20260930-a31e8d` only show a run after it ends (`describe` answers 404
"No summary").
`agnt5 run` returns at the run's first pause with `status: paused` and the run ID; for a
function or workflow it also prints the run ID when it stops waiting.

To cancel such a run, or follow it without the CLI, use the gateway with a service key
(`agnt5-deploy`). Export it only where you run these calls: the CLI reads `AGNT5_API_KEY` too,
and its control-plane commands answer 401 with a service key.

```bash
curl -s -H "X-API-KEY: $AGNT5_API_KEY" "https://gw.agnt5.com/v1/runs?component_name=my_workflow&limit=10"  # unfinished runs too (queued, assigned, ...)
curl -s -H "X-API-KEY: $AGNT5_API_KEY" https://gw.agnt5.com/v1/runs/<run-id>          # status
curl -s -H "X-API-KEY: $AGNT5_API_KEY" https://gw.agnt5.com/v1/runs/<run-id>/events   # journal so far
curl -s -X POST -H "X-API-KEY: $AGNT5_API_KEY" -H "Content-Type: application/json" \
  -d '{"reason": "stuck"}' https://gw.agnt5.com/v1/runs/<run-id>/cancel
```

Reading needs the default `run` scope; cancelling needs a key with the `workflow` scope
(`--scopes run,workflow`), otherwise it returns 403 `INSUFFICIENT_SCOPES`. A cancelled run
then shows up in `agnt5 inspect runs ls --status cancelled`.

## Traces

```bash
agnt5 inspect trace -r <runId>             # tree view, error spans highlighted red
agnt5 inspect trace -r <runId> --flat      # flat list by start time — useful for long traces
agnt5 inspect trace -r <runId> --verbose   # include span attrs: inputs/outputs/model/tokens
agnt5 inspect trace -r <runId> --output json > trace.json
```

Spans can take about a minute to arrive after a run ends. "No spans found" right after a run
usually means "not yet": retry before concluding the trace is empty.

Tree example:
```
workflow.travel_booking_workflow      [27.9s]
agent.travel_booking_agent            [27.0s]
chat openai/gpt-5-mini                [8.9s]
tool.search_flights                   [197ms]
```

In Studio: open the run → **Trace** tab — interactive tree, updates live while running.

`agnt5 inspect trace -r` looks the run up among the project's 200 most recent run summaries,
then on the gateway, which also has runs that have not ended. If it still reports the run as not
found, read the run's events with the MCP tool `get_run_events`, or open its **Trace** tab in
Studio.

## Logs

A run's logs hold what your code logged through the SDK logger (`ctx.logger`, `get_logger`)
plus the run's lifecycle lines. Read them with `agnt5 inspect logs -r <runId>` (`--severity`,
`--follow`, `--tail`; older CLIs answer 403), the MCP tool `get_run_logs(run_id, project_id)`,
or the run page in Studio.

Plain stdout/stderr (`print`, `console.log`, Go `log.Printf`) never reaches a run's logs.
Locally it prints in the `agnt5 dev` terminal (`agnt5 dev logs` when detached). A deployed
worker's stdout is not shown anywhere, so log what you need through the SDK logger.

Deployment logs are the platform's record of a deployment (bundle, scheduling, readiness,
traffic switch), not your worker's output:

```bash
agnt5 logs <deployment-id> --since 2h
agnt5 logs <deployment-id> --follow --timestamps
agnt5 deploy debug <deployment-id> --logs     # a crashed worker's exit code and last output line
```

Live output deltas (`output.delta`, `lm.message.delta`, `lm.thinking.delta`, `lm.tool_call.*`,
`progress.*`) are transient: they stream while the run executes but are not stored or
replayed on reconnect. An agent run on the platform streams them as `lm.message.*` and
`lm.thinking.*` (what `Client.stream_events` receives); a Python `Agent.stream()` called
in-process yields `lm.content_block.*` instead. Lifecycle, step, tool and model-call boundary
events are durable.

## Instrument your own code

```python
from agnt5 import get_logger, set_log_level
from agnt5.tracing import span, span_context

ctx.logger.info("Parsed invoice", invoice_id=inv.id, pages=len(pages))  # kwargs become log attributes

@span("score_candidates", component_type="function", stage="rank")     # one span per call
async def score_candidates(...): ...

with span_context("db_query", runtime_context=ctx._runtime_context, table="users") as s:
    rows = query()
    s.set_attribute("row_count", str(len(rows)))                         # attribute values are strings
```

`ctx.logger` accepts any keyword arguments and attaches them to the run log line (dicts and
lists are JSON-encoded, other values stringified). `create_span(name, component_type,
runtime_context, attributes={...})` is the underlying context manager; passing
`runtime_context=ctx._runtime_context` (the SDK's own example) links the span to the run's
trace. `get_logger(name)` returns a logger wired to the AGNT5 handlers; `set_log_level("DEBUG")`
or `AGNT5_DEBUG=1` (set before import) turns on SDK debug output.

## Metrics (Studio)

There is no CLI command or MCP tool for metrics. Per run, `get_run_summary` has the duration
and LLM cost, and `get_run_events` the tokens and cost of each LLM call. In Studio:

- **Analytics** — summary for a time window: total executions, success rate, P95 latency,
  total LLM cost; charts for executions/latency over time, LLM usage by model, top errors.
- **Metrics** — per-component breakdown; filters: deployment, component, status; granularity
  `auto|1m|5m|1h|1d`; auto-refresh; UTC toggle.

Time ranges: `1h`, `3h`, `24h`, `7d`, `30d`, or custom.

## Automatic capture of OpenAI / OpenAI Agents SDK / Google ADK calls

Since SDK 0.11, calls made through these libraries *inside* an AGNT5 component show up in the
trace without extra instrumentation, as `agent.*`, `lm.*`, and `tool_call.*` events tagged
`capture_mode=observed` and `source=<library>`. Capture turns on at worker startup when a
supported library version is installed (`pip install "agnt5[openai]"`, `"agnt5[openai-agents]"`,
`"agnt5[google-adk]"`).

| Env var | Effect |
|---|---|
| `AGNT5_CAPTURE=off` | Disable all capture |
| `AGNT5_CAPTURE_OPENAI=0` / `_OPENAI_AGENTS=0` / `_GOOGLE_ADK=0` | Disable one library |
| `AGNT5_CAPTURE_CONTENT_MODE` | `full` (default), `redacted`, or `metadata-only` (no prompt/response text) |
| `AGNT5_CAPTURE_MAX_CONTENT_CHARS` | Per-string cap, default `32768` |

Observed events are best-effort: capture never blocks or alters the provider call, so a missing
span is not proof the call didn't happen. Missing spans usually mean an unsupported library
version or the call ran outside a component. Capture is installed once, when `Worker` or
`ServerlessApp` boots, so these variables must be set before the process starts. An invalid
`AGNT5_CAPTURE_CONTENT_MODE` value logs a warning and falls back to `metadata-only`, not
`full`.

## Machine-readable output (any CLI command)

`--output json` / `-o json` forces JSON; `--output text` forces text. Precedence: explicit
flag > deprecated `--json` (still works, warns on stderr) > `AGNT5_OUTPUT`/`OUTPUT_FORMAT` env
vars > auto-detect (text on a TTY, JSON when piped/redirected — so
`agnt5 inspect runs ls | jq` gets structured output automatically).

Envelope shape: `{"data": {...}}` on success, `{"data": [...], "meta": {...}}` with
pagination, `{"error": {"message", "code", "category", "suggestions"}}` on failure
(stderr, non-zero exit). If the upstream API already nests under `"data"`, the CLI hoists it
so double-nesting never appears.

## Source

https://agnt5.com/docs/run/deploying
