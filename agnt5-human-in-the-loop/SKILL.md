---
name: agnt5-human-in-the-loop
description: Add durable human-in-the-loop pauses to an AGNT5 workflow with ctx.wait_for_user() (text, approval, select, multiselect) or agent-level AskUserTool/RequestApprovalTool, including the answer formats each input type returns and how to answer a paused run from your own backend (status `paused`, telling a question from a durable sleep, POST /v1/workflows/resume/{run_id} with a `workflow`-scoped key, cancelling). Use when a workflow or agent needs to pause for human approval, ask the user a question, or let them pick an option before continuing ("ask_user", "request_approval", "wait for sign-off"), including replay-safety guidance for code that runs before a pause.
---

# AGNT5 Human-in-the-loop

> **TypeScript or Go?** This file shows the Python API. Read [references/typescript.md](references/typescript.md) or [references/go.md](references/go.md) first: same sections, the exact signatures for that SDK, and what it does not support.

`ctx.wait_for_user()` pauses a workflow durably mid-execution, shows a question to the user,
and resumes from that exact point once they respond — the pause survives worker restarts.

## `ctx.wait_for_user()` parameters

| Parameter | Default | Description |
|---|---|---|
| `question` | required | Text shown to the user |
| `input_type` | `"text"` | `"text"`, `"approval"`, `"select"`, or `"multiselect"` |
| `options` | `None` | List of `{"id": ..., "label": ...}` dicts (required for approval/select/multiselect) |
| `allow_custom` | `False` | Adds a free-text "Something else" option to select/multiselect |
| `skippable` | `False` | Adds a Skip button; returns `None` when skipped |

## Input types

```python
import json

# text — free-form
name = await ctx.wait_for_user("What should we call this report?")

# approval — yes/no gate
decision = await ctx.wait_for_user(
    question="Deploy to production?", input_type="approval",
    options=[{"id": "approve", "label": "Approve"}, {"id": "reject", "label": "Reject"}],
)
if decision == "reject":
    return {"status": "cancelled"}

# select — single choice, returns the chosen id
fmt = await ctx.wait_for_user(
    question="Which output format?", input_type="select",
    options=[{"id": "pdf", "label": "PDF"}, {"id": "markdown", "label": "Markdown"}],
)

# multiselect — a JSON array string of chosen ids, e.g. '["market","tech"]'
topics = await ctx.wait_for_user(
    question="Which topics?", input_type="multiselect",
    options=[{"id": "market", "label": "Market"}, {"id": "tech", "label": "Tech"}],
)
selected = json.loads(topics) if topics else []
```

Studio submits multiselect answers as a JSON array string (a custom entry appears inside it
as `"__custom__:<text>"`); for `select` with `allow_custom=True` the custom text comes back
with that prefix already stripped. These formats were live-verified for TypeScript and Go
against the same gateway; Python was not separately live-tested.

`skippable=True` → handle `None`: `note = await ctx.wait_for_user(..., skippable=True); instructions = note or "default"`.
You can call `wait_for_user()` any number of times in one workflow, including inside `if`
blocks — each call gets its own pause index tracked correctly across replays.

## Replay safety (read before generating HITL code)

When `wait_for_user()` is called the first time, the workflow saves state and pauses. When
the user responds, AGNT5 re-runs the **entire workflow function from the top** — everything
before the pause executes again, but this time `wait_for_user()` finds the saved answer and
returns immediately instead of pausing.

```
First run:   generate_draft() → wait_for_user() → pauses
On resume:   generate_draft() → wait_for_user() → returns saved answer → publish()
```

**Rule: checkpoint side effects, never guard the pause.**

```python
draft = await ctx.step(generate_draft, topic, key="draft")        # replay returns the cached draft
await ctx.step(notify_reviewer, draft, key="notify")              # sent once, not on every resume

decision = await ctx.wait_for_user(   # never guard this call itself
    question=f"Approve this draft?\n\n{draft}", input_type="approval",
    options=[{"id": "approve", "label": "Approve"}, {"id": "discard", "label": "Discard"}],
)
```

Anything that must not repeat on resume — an LLM call, a notification, an external API call,
a charge — belongs in a `ctx.step(...)` before the pause: on replay the checkpointed result
returns without re-running it. Bare code in the workflow body (logging, string building) does
re-run; that is fine as long as it has no side effects. `ctx._is_replay` exists but is a
private attribute — use it at most to suppress duplicate log lines.

**Never wrap the pause in a bare `except:`.** `wait_for_user()` pauses by raising
`WaitingForUserInputException`, a `BaseException`; `except:` / `except BaseException:`
around it — or around an agent whose `AskUserTool` triggers it — swallows the pause and the
workflow continues with no answer. Catch `Exception`.

## Multi-step HITL with state

```python
name = await ctx.wait_for_user(question="What is your name?", input_type="text")
ctx.state.set("user_name", name)

role = await ctx.wait_for_user(
    question=f"Hi {name}, what is your role?", input_type="select",
    options=[{"id": "engineer", "label": "Engineer"}, {"id": "manager", "label": "Manager"}],
)
```

## Agent-level HITL tools

Let an agent itself ask a question or request approval mid-run. Both need a workflow context
(durable suspension requires it) — **instantiate inside the `@workflow` function**, not at
module level:

```python
from agnt5 import Agent, WorkflowContext, workflow
from agnt5.tool import AskUserTool, RequestApprovalTool   # not re-exported from `agnt5`

@workflow
async def agent_with_hitl(ctx: WorkflowContext, task: str) -> dict:
    agent = Agent(
        name="assistant", model="openai/gpt-4o-mini",
        instructions="Ask for clarification when ambiguous. Request approval before changes.",
        tools=[AskUserTool(ctx), RequestApprovalTool(ctx)],
    )
    result = await agent.run(task, context=ctx)
    return {"response": result.output}
```

| Tool | Behavior |
|---|---|
| `AskUserTool(ctx)` | Agent calls `ask_user` with a question; workflow pauses for a text reply |
| `RequestApprovalTool(ctx)` | Agent calls `request_approval`; workflow pauses with Approve/Reject |

## Answering a pause from your own backend

A paused run reports status `paused` (`RunStatus.PAUSED`), both while it waits for an answer
and during a durable `ctx.sleep()`. `awaiting_user_input` never appears. `agnt5 run` and
`client.run(...)` return at the first pause with `status: paused` and the run ID. In 0.13.6
that `RunResponse` has `is_error == True` and `raise_for_status()` raises
`RunError("Run failed with status: paused")`, so test `res.status == RunStatus.PAUSED` first.
Keep the ID. To find a paused run you lost, use `agnt5 inspect runs ls --status paused` (CLI
`20260930-a31e8d` or later; MCP `list_runs` does not list unfinished runs yet) or the gateway's
run list below. Follow the run on the gateway:

| Call | Use |
|---|---|
| `GET /v1/runs?component_name=<workflow>` | Find queued, assigned and paused runs (filters: `status`, `deployment_id`, `limit`) |
| `GET /v1/runs/{run_id}` or `client.get_status(run_id)` | Status (`paused`, `running`, `completed`, ...) |
| `GET /v1/runs/{run_id}/events` or `client.get_events(run_id)` | What the run is waiting on |
| `POST /v1/workflows/resume/{run_id}` with `{"user_response": ...}` | Answer the question |
| `POST /v1/runs/{run_id}/cancel` with optional `{"reason": "..."}` | Stop the run |

**Confirm the pause is a question before you answer.** Read the run's newest `workflow.paused`
event. For a question its `metadata` has `pause_reason: "user_input_required"`, `pause_index`
and `question` (Python and Go workers also emit `approval.requested`); for a durable sleep its
`data` has `reason: "timer"`. A resume sent while the run only sleeps is accepted, and its
answer goes to the next question without that question being shown.

```python
from agnt5 import Client, RunStatus


def pending_question(client: Client, run_id: str) -> dict | None:
    """Metadata of the question a paused run waits on; None while it sleeps or runs."""
    if client.get_status(run_id).status != RunStatus.PAUSED:
        return None
    paused = [e for e in client.get_events(run_id) if e.event_type == "workflow.paused"]
    latest = (paused[-1].metadata or {}) if paused else {}
    return latest if latest.get("pause_reason") == "user_input_required" else None
```

There is no Python `Client` method to answer or cancel in 0.13.6; call the gateway with a
service key that has the `workflow` scope. A `run`-only key gets 403 `INSUFFICIENT_SCOPES` on
resume and on cancel.

```bash
agnt5 service-keys create --name backend --project <project-id> --scopes run,workflow
curl -X POST "$AGNT5_GATEWAY_URL/v1/workflows/resume/<run_id>" \
  -H "X-API-KEY: $AGNT5_API_KEY" -H "Content-Type: application/json" \
  -d '{"user_response": "approve"}'
```

Keep the service key out of the shell you run the `agnt5` CLI in: the CLI reads
`AGNT5_API_KEY` too, and its control-plane commands answer 401 with a service key.

To pin the key to one environment add `--environment <environment-id>`: it takes the ID
(`env_id` in `agnt5 deployment list -o json`), not the name.

`user_response` reaches `wait_for_user()` as a **string**: send a plain string for text /
approval / select, a JSON array string (`"[\"market\",\"tech\"]"`) for multiselect, and
`"__skipped__"` to skip (the SDK turns it into `None`). A non-string JSON value is serialized
before delivery, so `null` arrives as the string `"null"`, not as a skip. Answer formats were
live-verified for TypeScript/Go; Python not separately. Client usage: `agnt5-client`.

## Edge cases

- **`text` input with unexpected content**: returns whatever the user typed as a plain
  string — validate/convert yourself (`float(raw)` etc., catch `ValueError`).
- **Conditional pauses**: fine to put `wait_for_user()` inside an `if` — only the branch that
  actually executes gets a pause index, and that stays correct across replays.

## Source

https://agnt5.com/docs/build/human-in-the-loop · https://agnt5.com/docs/build/tools
