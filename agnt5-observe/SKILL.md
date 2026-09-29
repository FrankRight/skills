---
name: agnt5-observe
description: Look up AGNT5 runtime data with the CLI, MCP, or Studio - list and describe runs, print execution traces (steps, tool calls, LLM spans), stream run or deployment logs, read throughput/latency/cost metrics, instrument your own code (ctx.logger attributes, agnt5.tracing spans, get_logger / set_log_level, AGNT5_DEBUG), and control automatic OpenAI / OpenAI Agents SDK / Google ADK call capture (AGNT5_CAPTURE*). Use for "show me recent failed runs", "tail the logs", "print the trace for run X", "add a span or log attribute", "why aren't my OpenAI calls in the trace". For a root-cause analysis of one bad run use agnt5-run-investigation; for recurring issues across runs use agnt5-pattern-analysis.
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
agnt5 inspect logs -r <runId> --severity ERROR
agnt5 inspect trace -r <runId>
```

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
| `--status` | `completed`, `failed`, `running`, `pending` |
| `--component <name>` / `--component-type <type>` | Filter by component |
| `--since <window>` | e.g. `1h`, `24h`, `7d` |
| `--limit <n>` | Default 20 |
| `-w` | Watch mode |
| `--output json` / `-o json` | Machine-readable |

Each run records: run ID, component name+type, status, duration, queue time, step count,
retries, LLM call count, LLM cost, error (on failure). `describe` also prints next-step
commands (`agnt5 inspect logs -r ...`, `agnt5 inspect trace -r ...`).

## Traces

```bash
agnt5 inspect trace -r <runId>             # tree view, error spans highlighted red
agnt5 inspect trace -r <runId> --flat      # flat list by start time — useful for long traces
agnt5 inspect trace -r <runId> --verbose   # include span attrs: inputs/outputs/model/tokens
agnt5 inspect trace -r <runId> --output json > trace.json
```

Tree example:
```
workflow.travel_booking_workflow      [27.9s]
agent.travel_booking_agent            [27.0s]
chat openai/gpt-5-mini                [8.9s]
tool.search_flights                   [197ms]
```

In Studio: open the run → **Trace** tab — interactive tree, updates live while running.

## Logs

```bash
agnt5 inspect logs -r <runId>
agnt5 inspect logs -r <runId> --severity ERROR
agnt5 inspect logs -r <runId> --follow      # live stream while a run is in progress
agnt5 inspect logs -r <runId> --tail 20
```

Deployment (worker process) logs, not scoped to a run:

```bash
agnt5 logs <deployment-id> --since 2h
agnt5 logs <deployment-id> --follow --timestamps
```

Live output deltas (`output.delta`, `lm.*.delta`, thinking deltas, progress) are transient:
they stream while the run executes but are not stored or replayed on reconnect. Lifecycle,
step, tool and model-call boundary events are durable.

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

## Metrics (Studio or AGNT5 MCP)

From Claude with the AGNT5 MCP connected: `get_analytics_dashboard`, `get_component_breakdown`,
`get_error_breakdown`, `get_llm_usage`, `get_runs_timeseries`, `get_latency_timeseries`.
In Studio:

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
