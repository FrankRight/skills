---
name: agnt5-agents-tools
description: Configure AGNT5 agents and the tools they call - Agent(...) options (sampling settings, temperature=None for gpt-6, custom endpoints via provider env vars), agent.run(history=, prompt_context=) and AgentResult, custom @tool functions, built-in provider tools (web search, code interpreter, web fetch) and agnt5.tools web_fetch/web_search, MCP client/server integration, sandboxed code execution and its per-run lifecycle, before/after agent-model-tool callbacks (guardrails), agent memory (ctx.memory, ctx.conversation, automatic per-session history, failure policy), prompt caching, and multi-agent patterns (handoffs, agents-as-tools). Use when creating an agent, giving it a tool or capability, adding a guardrail callback, wiring an MCP server, routing between agents, or when an agent's model call is rejected.
---

# AGNT5 Agents and Tools

> **TypeScript or Go?** This file shows the Python API. Read [references/typescript.md](references/typescript.md) or [references/go.md](references/go.md) first: same sections, the exact signatures for that SDK, and what it does not support.

An **Agent** is an LLM that runs in a loop: reads its instructions, calls tools as needed,
keeps going until it has a final answer.

## Creating an agent

```python
from agnt5 import Agent

agent = Agent(
    name="researcher",
    model="openai/gpt-4o-mini",
    instructions="You are a research assistant. Answer questions with cited sources.",
)
```

| Parameter | Required | Description |
|---|---|---|
| `name` | yes | Identifier for this agent |
| `model` | yes | `"provider/model-name"`, e.g. `"openai/gpt-4o"`, `"anthropic/claude-sonnet-5"` |
| `instructions` | yes | System prompt |
| `tools` | no | Custom tools or other `Agent`s (auto-wrapped as agents-as-tools) |
| `built_in_tools` | no | `list[BuiltInTool]` — provider-hosted tools |
| `handoffs` | no | Agents to delegate full control to |
| `sandbox` | no | `Sandbox()` for isolated file/code execution |
| `max_iterations` | no | Max reasoning loops, default `10` |
| `temperature` / `max_tokens` / `top_p` | no | The only sampling settings. Default temperature `0.7` is sent unless you pass `temperature=None` (sends nothing) — required for `openai/gpt-6*`, which rejects any temperature; `gpt-5*` / `o1*` already send none. No `reasoning_effort` on `Agent` |
| `model_config` | no | Accepted but **not read** in 0.13.6 (stored on the agent, never used). Custom endpoints come from provider env vars read by the native layer: `OPENAI_BASE_URL`, `ANTHROPIC_BASE_URL`, `OPENROUTER_BASE_URL`, `DEEPSEEK_BASE_URL`, `MOONSHOT_BASE_URL`, `TOGETHER_BASE_URL`, plus `OPENAI_ORGANIZATION`, `OPENAI_PROJECT`, `OPENAI_REQUEST_TIMEOUT_SECS` |
| `cache` | no | `True` or `lm.PromptCache(...)` — provider prompt caching (see `agnt5-prompts`) |
| `callbacks` / `before_*_callback` / `after_*_callback` | no | Guardrail hooks — see Callbacks below |
| `skills` / `skills_dir` / `agents_md` | no | On-demand SKILL.md capabilities and AGENTS.md guidance — see `agnt5-agent-skills` |

Run with `result = await agent.run("...")`. `AgentResult` fields: `output`, `tool_calls`
(e.g. `[{"name": "get_weather", "arguments": '{"city": "Paris"}', "iteration": 1}]`),
`handoff_to`, `handoff_metadata` (the handoff tool's result dict; `None` without a handoff).
`run()` and `stream()` also take `history=[Message, ...]` (prior turns prepended to the
conversation) and `prompt_context={"var": value}` (fills `{{var}}` placeholders in
`instructions`). `agent.cumulative_cost_usd` sums the LLM cost of every run on that `Agent`
instance. Stream with `async for event in agent.stream("..."):` and check `event.event_type`:

| Event type | When it fires |
|---|---|
| `agent.started` / `agent.completed` / `agent.failed` | Agent loop begins / ends with a final answer / errors |
| `lm.content_block.started` / `.delta` / `.completed` | LLM response block streaming (`block_type` is `text` or `thinking`) |
| `tool_call.started` / `tool_call.completed` / `tool_call.failed` | A tool call |
| `skill.loaded` | The agent loaded a SKILL.md (see `agnt5-agent-skills`) |

`agent.iteration.started` / `.completed` are recorded on the run's events (the trace) but are not
yielded by `agent.stream()`.

These are the in-process names. When the agent runs as a worker component and a client reads
its stream (`Client.stream_events`, SSE), the worker renames the content-block events to the
cross-SDK names: `lm.message.start` / `.delta` / `.stop` for text and `lm.thinking.start` /
`.delta` / `.stop` for thinking. Match those in client code. TypeScript uses the `lm.message.*`
names in process as well — see [references/typescript.md](references/typescript.md).

Inside a workflow, pass `context=ctx`: `await agent.run(task, context=ctx)`. With a context
the agent loads and saves its conversation history automatically, scoped by `user_id`, else
`session_id`, else the run id — so a run started without a session starts fresh every time.

## Custom tools

```python
from agnt5.tool import tool
from agnt5.context import Context

@tool
async def get_weather(ctx: Context, city: str) -> str:
    """Get the current weather for a city.

    Args:
        city: Name of the city to check.
    """
    return f"Sunny, 22°C in {city}"

agent = Agent(name="assistant", model="openai/gpt-4o-mini",
              instructions="...", tools=[get_weather])
```

- First param must be `ctx: Context` (injected, not exposed to the model).
- Every other param becomes a model-fillable field — type hints + docstring `Args:` build the
  JSON schema.
- Sync functions auto-wrap in a thread pool. Tools register globally at import time.
- `@tool(...)` options: `name` / `description` (override the model-facing values; default:
  function name / first docstring line), `input_schema` (with `auto_schema=False`),
  `recovery_policy` (what happens if the worker dies mid-call; default
  `ActivationRecoveryPolicy.UNKNOWN_OUTCOME`, use `IDEMPOTENT_RETRY` for safe-to-repeat tools),
  `durable` (default `True`). `confirmation=True` is not enforced yet — for approval, use
  `agnt5-human-in-the-loop`.

## Built-in tools (no custom code)

Run on the provider's infrastructure — use `built_in_tools=`, **not** `tools=`:

```python
from agnt5.lm import BuiltInTool

agent = Agent(
    name="researcher", model="openai/gpt-4o-mini",
    instructions="Always use web_search for current information.",
    built_in_tools=[BuiltInTool.WEB_SEARCH],
)
```

| Tool | What it does | Provider |
|---|---|---|
| `BuiltInTool.WEB_SEARCH` | Live web search with cited results | OpenAI, Anthropic, Gemini (grounding) |
| `BuiltInTool.CODE_INTERPRETER` | Run code in a provider-hosted sandbox | OpenAI only |
| `BuiltInTool.FILE_SEARCH` | Search files uploaded to the provider | OpenAI only |
| `BuiltInTool.WEB_FETCH` | Fetch the content of a specific URL | Anthropic only |

`built_in_tools`, `tools`, and `sandbox` can all be set on the same agent at once.
`WEB_SEARCH` was live-tested on OpenAI only; the Anthropic (`web_search_20260209`) and Gemini
(`google_search`) mappings exist in the SDK core but were not live-tested. The `Agent`
docstring's "OpenAI Responses API only" note is stale.

**Provider-agnostic alternatives** (run in your worker, work with any model) — factories that
return a `Tool` for `tools=[...]`:

```python
from agnt5.tools import web_fetch, web_search

agent = Agent(..., tools=[web_fetch(max_bytes=200_000), web_search(max_results=5)])
```

`web_search` uses Brave, Tavily, or SearXNG — set `AGNT5_BRAVE_SEARCH_API_KEY`,
`AGNT5_TAVILY_API_KEY`, or `AGNT5_SEARXNG_URL` (or `AGNT5_WEB_SEARCH_PROVIDER`).

## MCP tools

Use `MCPClient` to pull tools from an existing external MCP server instead of writing custom
`@tool` code:

```python
from agnt5.mcp import MCPClient

mcp = MCPClient(id="repo-tools")
mcp.add_streamable_http_server("deepwiki", "https://mcp.deepwiki.com/mcp")
await mcp.connect()   # must connect before get_tools()/list_tools()

agent = Agent(name="repository_researcher", model="openai/gpt-4o-mini",
              instructions="...", tools=mcp.get_tools())
```

Transports: `add_streamable_http_server(name, url)` (current-spec remote), `add_sse_server(name, url, api_key=...)`
(legacy remote), `add_stdio_server(name, command, args=[...])` (local subprocess). Use
`async with mcp:` to connect/disconnect around one block.

| Method | What it does |
|---|---|
| `connect()` / `disconnect()` | Open/close connections to all configured servers |
| `get_tools()` | Return discovered tools as AGNT5 `Tool` objects for `Agent(tools=...)` |
| `list_tools()` | List every discovered tool with its source server |
| `list_server_tools(server)` | List tools from one server |
| `call_tool(server, name, args)` | Call a tool on a specific server |
| `call_tool_auto(name, args)` | Call the first matching tool name across connected servers |
| `is_connected(server)` | Check whether one server is connected |
| `connected_servers()` | Return connected server names |

To expose AGNT5 tools/agents/workflows *as* an MCP server instead, use `MCPServer`:

```python
from agnt5.mcp import MCPServer

server = MCPServer(id="my-server", name="My AGNT5 Server", version="1.0.0",
                    tools={"greet": greet}, agents={"assistant": assistant})
await server.run_stdio()   # or server.run_http(host, port, path)
```

## Sandboxed code execution

Pass `Sandbox()` when an agent needs to write/read/list files or run `python`/`javascript`/
`bash` before answering. AGNT5 wires the standard sandbox capabilities automatically; custom
tools can reach the same workspace via `ctx.sandbox`.

```python
from agnt5 import Agent, Sandbox

agent = Agent(
    name="coder", model="openai/gpt-4o-mini",
    instructions="Use the sandbox workspace to write files and run code before answering.",
    sandbox=Sandbox(),   # or Sandbox(provider="daytona") to pick a specific provider
)
```

`Sandbox(...)` options: `provider=`, `template=`, `env={...}`, `cpu_cores=`, `memory_mib=`,
`timeout_secs=`, `auto_destroy=True`. `sandbox_tools(sandbox)` returns the same tools for use
without `sandbox=`; `InMemorySandbox()` is a no-network stand-in for tests.

Lifecycle: the agent closes its sandbox in a `finally` after **every** `run()`/`stream()`, and
with `auto_destroy=True` (default) that destroys the provider sandbox — the next run creates a
fresh one, so files written in one run do not survive to the next. A module-level `Sandbox()`
is one object shared by every concurrent run of that agent; create it inside the workflow
when runs can overlap.

Provider credentials (E2B, Daytona, Vercel, Northflank, Together), local vs deployed setup,
and troubleshooting: [references/sandbox-providers.md](references/sandbox-providers.md).

## Multi-agent patterns

**Agents as tools** — coordinator invokes a specialist, gets the result back, keeps
reasoning. Pass the `Agent` directly in `tools=[...]`; it auto-wraps as `ask_<name>`.

```python
coordinator = Agent(
    name="coordinator", model="openai/gpt-4o-mini",
    instructions="Use the researcher and analyst to answer complex questions.",
    tools=[research_agent, analyst_agent],
)
```

**Handoffs** — calling agent transfers control entirely; the specialist's output becomes the
final result. Good for routing/triage.

```python
from agnt5 import handoff

triage_agent = Agent(
    name="triage", model="openai/gpt-4o-mini",
    instructions="Route the user to the right specialist.",
    handoffs=[
        handoff(billing_agent, "Transfer for billing/payment questions"),
        handoff(technical_agent, "Transfer for technical/product issues"),
    ],
)
result = await triage_agent.run("My payment failed but I was still charged.")
print(result.handoff_to)   # "billing"
```

Pass agents directly (`handoffs=[billing_agent, technical_agent]`) for default-config
handoffs without `handoff()`. `handoff()` params: `agent` (required), `description`
(shown to the LLM), `tool_name` (default `transfer_to_{name}`), `pass_full_history`
(default `True`), `join_policy` (`ChildJoinPolicy.REQUIRED` default; `DETACHED` lets the
parent finish without waiting on the child).

| Use handoffs when… | Use agents-as-tools when… |
|---|---|
| One specialist should fully own the rest of the conversation | The coordinator needs to synthesize results from multiple specialists |
| It's a routing/triage decision | It's a sub-task within a larger reasoning loop |

For HITL tools (`AskUserTool`, `RequestApprovalTool`), use the `agnt5-human-in-the-loop`
skill.

## Callbacks (guardrails, caching, redaction)

Hooks run around the agent loop, each model call, and each tool call. Return `None` to
continue normally; return a value to short-circuit (skip the model/tool and use that value);
wrap in `override(...)` when the replacement value is itself `None`.

```python
from agnt5 import Agent, ToolCallbackContext

BLOCKED = {"delete_account"}

def guard_tools(cb: ToolCallbackContext):
    if cb.tool_name in BLOCKED:
        return {"error": f"{cb.tool_name} is not allowed"}   # tool is not executed
    return None

agent = Agent(name="support", model="openai/gpt-4o-mini", instructions="...",
              tools=[...], before_tool_callback=guard_tools)
```

Hooks: `before_agent_callback` / `after_agent_callback` (`AgentCallbackContext`),
`before_model_callback` / `after_model_callback` (`ModelCallbackContext`, request, response),
`before_tool_callback` / `after_tool_callback` (`ToolCallbackContext`: `tool_name`,
`arguments`, `iteration`). Or bundle them: `callbacks=AgentCallbacks(before_tool=...)`.

## Memory

Only `WorkflowContext` has `ctx.memory` / `ctx.conversation` — `FunctionContext` does not
(an agent run from a function gets memory through the workflow's `context=ctx`):

```python
await ctx.memory.set("theme", "dark")                 # KV, session-scoped by default
theme = await ctx.memory.get("theme", "light")
await ctx.memory.working.merge({"step": "research"})   # dict scratchpad, session-scoped (not per run)
await ctx.memory.user.save("Prefers concise answers", kind="preference")  # semantic, per user
hits = await ctx.memory.user.search("answer style", limit=5)
await ctx.conversation.add("user", message)            # session chat history
history = await ctx.conversation.get_messages(limit=20)
```

Scopes: `ctx.memory.session`, `.user` (raises `RuntimeError` without a `user_id`), `.run`,
`.global_()`. Semantic memory is best-effort by default
(`AGNT5_MEMORY_FAILURE_POLICY=best_effort`): when the memory service is off, `save()` returns
`None` and `search()` returns `[]` with no error — set
`AGNT5_MEMORY_FAILURE_POLICY=require_memory_or_fail` to raise instead. The old
`agnt5.memory.SemanticMemory` / `ConversationMemory` classes are deprecated — built-in
vector-backed `SemanticMemory.store()` now raises.

Related: direct model calls (`lm.generate` / `lm.stream`, structured output, gpt-6 quirks) are
in `agnt5-models`.

## Source

https://agnt5.com/docs/build/agents · https://agnt5.com/docs/build/tools · https://agnt5.com/docs/build/mcp · https://agnt5.com/docs/build/sandboxes
