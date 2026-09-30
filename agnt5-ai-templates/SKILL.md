---
name: agnt5-ai-templates
description: Generate a complete AGNT5 project (agents, tools, functions, workflows, worker entry point, config) from a description of what it should do, or scaffold from an existing template with agnt5 create --template. Use when the user says "build me an agent that...", "scaffold a multi-agent workflow for...", "generate AGNT5 code for...", or describes specific agents/workflows/functions to create - not for a bare "create a new project" request with nothing to generate (use agnt5-project-init). Python by default; TypeScript and Go supported.
---

# AGNT5 Template Generation

When asked to create a new AGNT5 template from a description (a "blueprint"), follow this guide
— **do not ask the user to point at a reference template**. For a blank project with nothing
to generate, use `agnt5-project-init` instead.

Default to **Python**. For TypeScript or Go, read the matching reference file before writing
code: [references/typescript.md](references/typescript.md),
[references/go.md](references/go.md). For a non-OpenAI model or provider env vars, read
[references/providers.md](references/providers.md).

## First: is there a ready-made template?

The CLI has no command that lists templates. The catalog is a JSON file with every template,
its languages and versions:

```bash
curl -s https://templates.agnt5.com/templates/templates.json \
  | jq -r '.templates | to_entries[] | "\(.key): \(.value.languages | keys | join(", "))"'
```

It lists `quickstart`, `weather-agent`, `code_reviewer`, `coding_agent`,
`travel_booking_customer_service`, `tutor_agent` and `hitl_deep_research`, each in python,
typescript and go. Read it before recommending a template by name; never guess one.

```bash
agnt5 version update                                     # every time -- a stale CLI can mis-extract templates
agnt5 create my-weather-agent --template python/weather-agent   # optional: --version v1.0.0 or name@v1.0.0
```

The project takes the name you passed (`my-weather-agent`): the CLI rewrites the `name:` in the
template's `agnt5.yaml`. CLIs older than `20260930-a31e8d` registered the template's own name
(`quickstart`, `agnt5-customer-service`) instead; with one of those, scaffold with `--local`,
edit `name:`, then link: `agnt5 init --new --name <name> --workspace <ws> -y`.

Known template caveats — fix these right after scaffolding:

- Every Go template, `python/quickstart`, `typescript/quickstart` and `python/weather-agent`
  have a `deploy.resources` block in `agnt5.yaml`. It is not applied; delete it.
- `python/weather-agent`: `agnt5.yaml` sets `deploy.dockerfile: ./Dockerfile`,
  `ignore_file: .dockerignore` and `registry.url`, but the template ships neither file.
  Without a root `Dockerfile` the deploy is a code bundle anyway and the missing ignore file
  adds no exclusions, so delete those three settings rather than rely on them.
- `typescript/quickstart`: the `digest` workflow calls its functions directly
  (`await fetchTopIds(ctx, { limit })`). A direct call is not checkpointed and runs again on
  every replay. Wrap each call in `ctx.step`, keyed when calls run concurrently, then check
  it with `npx tsc --noEmit`:

  ```typescript
  const ids = await ctx.step('fetch_top_ids', () => fetchTopIds(ctx, { limit }));
  const stories = await Promise.all(
    ids.map((storyId) =>
      ctx.step('fetch_story', () => fetchStory(ctx, { storyId }), { key: String(storyId) }),
    ),
  );
  ```

- `typescript/quickstart` ships both `package-lock.json` and `pnpm-lock.yaml`. Keep the one
  for the package manager you use and delete the other, so installs don't disagree.
- `python/weather-agent` still calls the deprecated `ctx.task(...)` (use `ctx.step(...)`,
  `agnt5-workflows`) and imports `setup_module_logger` from the private `agnt5._telemetry` in
  `app.py` and `test.py` (use the public `from agnt5 import get_logger`, `agnt5-observe`).
- `go/quickstart`: `main.go` picks `claude-3-5-haiku-20241022` when `ANTHROPIC_API_KEY` is
  set and `gpt-5-mini` otherwise. Change `newSummarizerModel()` to the model you want.

Generate from scratch (below) when nothing is a close match or the user wants a custom
combination.

## Python project layout

```
<template-name>/
├── app.py                 # Worker entry point
├── pyproject.toml
├── agnt5.yaml
├── .env.example
├── README.md
└── src/<package_name>/
    ├── __init__.py        # re-exports
    ├── agents.py
    ├── workflows.py
    ├── functions.py       # only if there are distinct processing stages
    └── tools.py           # only if agents need custom tools
```

Package name and directory name = template name in `snake_case` (e.g. `blog_creation`).

## Config files

`pyproject.toml` — before writing it, check the latest `agnt5` on PyPI
(`pip index versions agnt5` or https://pypi.org/project/agnt5); **0.13.6** was current when this skill
was written:

```toml
[project]
name = "<template-name>-agnt5"
version = "1.0.0"
description = "<one-line description>"
requires-python = ">=3.11"
dependencies = [
    "agnt5~=0.13.6",          # add extras when wrapping those libraries: agnt5[openai], [openai-agents], [google-adk]
    "python-dotenv>=1.2.1",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/<package_name>"]

[tool.ruff]
line-length = 100
target-version = "py312"
```

`agnt5.yaml`:

```yaml
name: <template-name>
language: python
language_version: "3.12"
environment: dev

worker:
  command: "uv run python app.py"   # what `agnt5 dev` and the deployed worker run; inferred from the language when omitted

# deploy:                           # optional: dockerfile, ignore_file, base_image, build_args, registry
# variables: {}                     # optional key/value map
```

Don't add `deploy.resources`: it is not applied (the platform sizes workers).

Full schema and what gets bundled: `agnt5-deploy`. The managed Python worker image runs
**Python 3.14** (`ghcr.io/agnt5dev/python-worker:3.14`), so pick dependencies with 3.14
wheels even though `requires-python` says `>=3.11`.

`.env.example` — one line per required key, e.g. `OPENAI_API_KEY="your-openai-api-key-here"`
(key names per provider: [references/providers.md](references/providers.md)).

## Agents (`agents.py`)

```python
from agnt5 import Agent
from <package_name>.tools import some_tool  # only if the agent uses custom tools

agent_prompt = """You are <AgentName>, <one-line role>.

Your responsibilities:
1. ...

Output format:
LABEL:
[structured output]"""

my_agent = Agent(
    name="AgentName",
    model="openai/gpt-4o-mini",
    instructions=agent_prompt,
    tools=[some_tool],          # omit entirely if no tools — never pass tools=[]
)

__all__ = ["my_agent"]
```

Optional `Agent` kwargs, only when needed: `max_iterations` (default 10), `max_tokens`,
`built_in_tools=[BuiltInTool.WEB_SEARCH]`, `handoffs=[...]`, `sandbox=Sandbox()`,
`cache=True`. Full options, built-in tools, MCP, sandboxes, callbacks, memory, handoffs vs.
agents-as-tools: **`agnt5-agents-tools`**.

## Tools (`tools.py`) — only if needed

```python
from agnt5.context import Context
from agnt5.tool import tool

@tool
async def my_tool(ctx: Context, param: str) -> str:
    """One-line description the model reads to decide when to call this.

    Args:
        param: Description of the parameter.
    """
    return ...

__all__ = ["my_tool"]
```

First param is always `ctx: Context` (hidden from the model); type hints + docstring `Args:`
build the schema.

## Functions (`functions.py`) — only for distinct stages

```python
from agnt5 import FunctionContext, function
from <package_name>.agents import my_agent

@function
async def my_stage(ctx: FunctionContext, input_data: str) -> str:
    """One-line description of this stage."""
    result = await my_agent.run(input_data, context=ctx)
    output = result.output
    if output.strip().startswith("LABEL:"):
        output = output.replace("LABEL:", "", 1).strip()
    return output

__all__ = ["my_stage"]
```

Streaming: make the `@function` an async generator (`yield` chunks) only when the caller needs
real-time output. Retries/backoff/timeouts: `agnt5-workflows`.

## Workflows (`workflows.py`)

```python
from agnt5 import WorkflowContext, workflow
from <package_name>.functions import stage1, stage2

@workflow
async def my_workflow(ctx: WorkflowContext, message: str) -> dict:
    """Describe the stages."""
    result1 = await ctx.step(stage1, message, key="stage1")
    result2 = await ctx.step(stage2, message, result1, key="stage2")
    return {"status": "completed", "output": result2}

__all__ = ["my_workflow"]
```

Parallel fan-out (`ctx.parallel` / `gather`), durable sleep, cron schedules,
state, idempotency: **`agnt5-workflows`**. Human approval/input pauses: **`agnt5-human-in-the-loop`**.
Webhook/event triggers and chat bots: **`agnt5-webhooks-integrations`**.

## Worker entry point (`app.py`)

```python
#!/usr/bin/env python3
"""<Template Name> — AGNT5 Worker."""

import asyncio
import logging
import sys

from agnt5 import Worker
from <package_name>.agents import agent1, agent2
from <package_name>.functions import stage1, stage2
from <package_name>.tools import tool1
from <package_name>.workflows import my_workflow

logging.basicConfig(level=logging.INFO, format="%(asctime)s %(name)s %(levelname)s %(message)s")
logger = logging.getLogger(__name__)

async def main() -> int:
    try:
        worker = Worker(
            service_name="<template-name>",
            service_version="1.0.0",
            workflows=[my_workflow],
            functions=[stage1, stage2],
            agents=[agent1, agent2],
            tools=[tool1],
            scorers=[my_scorer],      # only if you wrote @scorer functions — unlisted scorers never register
        )
        await worker.run()
    except Exception as e:
        logger.error("Worker failed: %s", e, exc_info=True)
        return 1
    return 0

if __name__ == "__main__":
    sys.exit(asyncio.run(main()))
```

The coordinator endpoint comes from `AGNT5_COORDINATOR_ENDPOINT` (set by `agnt5 dev`), so
don't hardcode it. Alternative to explicit lists: `Worker(service_name=..., auto_register=True)`
discovers components in the packages listed under `[tool.hatch.build.targets.wheel]`. Omit any
list (and its import) whose file you didn't create. `Worker(max_concurrency=...)` caps
in-flight invocations (default 100, or `AGNT5_MAX_CONCURRENCY`) — raise it for IO-bound LLM
workflows, lower it for CPU-bound work.

## Step-by-step

1. **Language** — Python unless the user asks for TypeScript/Go (then read that reference).
2. **Parse the blueprint** — agents (names, roles), workflow stages and order, tools needed,
   input parameters, output shape. HITL **only if explicitly requested**; handoffs **only if a
   coordinator must route to specialists at runtime**.
3. **Write files in order**: `tools.py` → `agents.py` → `functions.py` → `workflows.py` →
   `__init__.py` → `app.py` → `pyproject.toml`, `agnt5.yaml`, `.env.example`, `README.md`.
4. **Naming**: agent `name=` in `PascalCase`; instance variables `snake_case_agent`; functions
   `verb_noun`; workflow `<template_name>_workflow`.
5. **Defaults — keep it minimal**:
   - No `tools.py` unless agents need an external API not covered by `built_in_tools` or MCP.
     For web search/fetch prefer `BuiltInTool.WEB_SEARCH` / `WEB_FETCH`, or
     `agnt5.tools.web_search()` / `web_fetch()` for provider-agnostic tools.
   - No `functions.py` for a simple linear flow — the workflow can call agents directly.
   - No handoffs for fixed sequences — use workflow steps.
   - No HITL unless asked.
   - Model `openai/gpt-4o-mini` unless the user names another; ask when unsure rather than
     guessing a model name. For `openai/gpt-6*` pass `temperature=None` to `Agent` (the
     default 0.7 is rejected with a 400) and do not use `reasoning_effort` — the native
     binding never sends it. Details: `agnt5-models`.
   - Keep `prompts/` and `skills/` inside the project and out of `.gitignore`: they resolve
     against the worker's working directory and must ship in the deploy bundle.
6. **Hand off** to `agnt5-project-init` (`uv sync`, `.env`, `agnt5 dev`, `agnt5 run`).

## Source

https://agnt5.com/docs/build/agents · https://agnt5.com/docs/build/workflows · https://agnt5.com/docs/install-cli
