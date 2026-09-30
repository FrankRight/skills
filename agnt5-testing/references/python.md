# Python testing reference

Verified against agnt5 0.13.6 (`src/agnt5/function.py`, `workflow.py`, `scorer.py`,
`sandbox.py`, `lm/base.py`, `lm/events.py`, `serverless.py`, `agent/core.py`) and the SDK's own
`pytest.ini` / `tests/unit`. The fake-model tests below pass under pytest with pytest-asyncio.

## pytest setup

```ini
# pytest.ini
[pytest]
asyncio_mode = auto            # pytest-asyncio: async def tests run without a marker
markers =
    llm: needs provider API keys
    integration: needs a running AGNT5 worker / gateway
```

Load `.env` in `conftest.py` if your tests need provider keys, and skip `llm` tests when the
key is missing (`pytest.mark.skipif(not os.getenv("OPENAI_API_KEY"), ...)`).

## Calling components directly

| Component | Direct call | Notes |
|---|---|---|
| `@function` without `ctx` | `await fn(a, b=1)` | The wrapper builds a local `FunctionContext` (`run_id="local-..."`); positional or keyword args |
| `@function(ctx, ...)` | `await fn(FunctionContext(run_id="test", correlation_id="corr", parent_correlation_id="parent"), ...)` | Raises `TypeError` if the first argument is not a `FunctionContext` |
| `@workflow` | `await wf(**kwargs)` | Keyword args only; a `WorkflowContext` with an in-memory state adapter is created; `ctx.step(...)` executes the function inline |
| `@workflow(ctx, ...)` with your own ctx | `await wf(ctx, **kwargs)` | Only if you already hold a `WorkflowContext` (e.g. inside another workflow) |
| `Agent` | `await agent.run("message")` | Needs a model - fake it (below) or mark the test `llm` |
| `@tool` | `await my_tool.invoke(ctx, a=2, b=3)` with a `FunctionContext` | `@tool` returns a `Tool` instance (first parameter must be `ctx: Context`); assert on `my_tool.input_schema`, derived from type hints |
| `@scorer` | `await run_scorer(name, ScorerRequest(...))` | Also resolves `structured_assertions` and the judge built-ins; other deterministic built-ins raise `ValueError: Scorer not found` (use the `agnt5.eval` functions) |

`FunctionContext(run_id, correlation_id, parent_correlation_id, attempt=0, retry_policy=None,
is_streaming=False, ...)`; `ctx.attempt` is whatever you pass. `timeout_ms` is applied with
`asyncio.wait_for`; retries/backoff are platform behaviour and never run locally.

## Fake model

`Agent.run()` calls the model's `stream()` when the agent has no tools, and `generate()` when it
has tools or model callbacks. Implement both. From `stream()` the agent takes the final text out
of the `LMCompleted` event's `output_data["text"]`, so a `stream()` that yields nothing leaves
`result.output` as `''`.

```python
import json

from agnt5 import Agent
from agnt5.context import Context
from agnt5.lm import GenerateRequest, GenerateResponse, LanguageModel, LMCompleted
from agnt5.tool import tool


class FakeModel(LanguageModel):
    """Plays canned replies in order and records every request."""

    def __init__(self, replies: list[GenerateResponse]):
        self.replies = list(replies)
        self.requests: list[GenerateRequest] = []

    async def generate(self, request: GenerateRequest) -> GenerateResponse:   # agents with tools
        self.requests.append(request)
        return self.replies.pop(0)

    async def stream(self, request: GenerateRequest):                         # agents without tools
        self.requests.append(request)
        reply = self.replies.pop(0)
        yield LMCompleted(name="fake", correlation_id="fake-lm", parent_correlation_id="",
                          output_data={"text": reply.text, "tool_calls": reply.tool_calls or []})


@tool
async def lookup_order(ctx: Context, order_id: str) -> str:
    """Look up an order.

    Args:
        order_id: The order ID.
    """
    return f"Order {order_id} is in transit."


async def test_agent_answers():
    model = FakeModel([GenerateResponse(text="The order is in transit.")])
    agent = Agent(name="support", model=model, instructions="Be brief.", temperature=None)
    result = await agent.run("Where is my order?")
    assert result.output == "The order is in transit."
    assert "Be brief." in (model.requests[0].system_prompt or "")


async def test_agent_calls_a_tool():
    model = FakeModel([
        GenerateResponse(text="", tool_calls=[
            {"id": "call_1", "name": "lookup_order", "arguments": json.dumps({"order_id": "42"})}]),
        GenerateResponse(text="Order 42 is in transit."),
    ])
    agent = Agent(name="support_tools", model=model, instructions="Use tools.",
                  tools=[lookup_order], temperature=None)
    result = await agent.run("Where is order 42?")
    assert result.output == "Order 42 is in transit."
    assert [call["name"] for call in result.tool_calls] == ["lookup_order"]
    assert "Order 42 is in transit." in str(model.requests[1].messages[-1])   # tool result went back
```

`Agent(model=...)` accepts any `agnt5.lm.LanguageModel` (`isinstance` check in
`agent/core.py`). Tool calls are dicts `{"id", "name", "arguments": <JSON string>}`. A custom
model that returns complete tool calls in `LMCompleted` can opt into the streaming path for tools
with `supports_streaming_tools = True`.

## Sandbox

```python
from agnt5 import InMemorySandbox

async with InMemorySandbox() as sb:                       # sandbox_id="memory"
    await sb.write_file("notes.txt", "hello")
    assert (await sb.read_file("notes.txt")).content == b"hello"   # missing path -> FileNotFoundError
    out = await sb.execute_code("print(1)", language="python")    # echo: stdout == "[python] print(1)", exit_code 0
```

`InMemorySandbox` stands in wherever a `Sandbox` is accepted (`Agent(sandbox=...)`, sandbox
tools); real execution needs `Sandbox` and a provider (`agnt5-agents-tools`).

## Scorers offline

```python
from agnt5 import ScorerRequest, run_scorer
from agnt5.eval import ScorerInput, contains, exact_match, json_schema, trace_scorer, TraceAssertion

r = await run_scorer("my_scorer", ScorerRequest(output=..., expected=..., input=..., trace=None, config=None))
assert r.passed and r.score >= 0.7
assert exact_match(ScorerInput(output="a", expected="a")).passed
assert contains(ScorerInput(output="Order 42 is in transit"), "in transit").passed
assert json_schema(ScorerInput(output='{"x":1}'), {"type": "object", "required": ["x"]}).passed
t = trace_scorer(ScorerInput(output=..., trace=events), [TraceAssertion.max_lm_calls(3), TraceAssertion.no_errors()])
```

Built-in judge names (`correctness`, `llm_judge`, ...) run through `run_scorer` too, but they
call a provider - mark those tests `llm`. Scorer semantics: `agnt5-scorers`.

## Serverless endpoint offline

```python
import json
from agnt5.serverless import serve

app = serve(service_name="test", workflows=[hello])       # no signing_secret -> unsigned accepted
status, payload, _ = await app.handle_http(
    method="POST", path="/agnt5/invoke", headers={},
    body=json.dumps({"protocol_version": "workerless.v1", "run_id": "r1",
                     "component_type": "workflow", "component_name": "hello", "input": {"name": "Ada"}}).encode())
assert status == 200 and payload["status"] == "completed" and payload["output"] == {"message": "hello Ada"}
```

Post again with `"checkpoint": payload["checkpoint"]` to assert that steps replay, and with
`"metadata": {"pause_index": "0", "user_response": "approve"}` to resume a `wait_for_user`.
Signals, earlier answers that must be resent, and signing timestamps are covered in
`agnt5-serverless`.

## Against a running gateway

```python
from agnt5 import Client
client = Client()                                    # AGNT5_GATEWAY_URL: http://localhost:34181 for `agnt5 dev up`, gw.agnt5.com for deployed
res = client.run("greet", {"name": "Ada"}, wait_timeout=60)
if res.is_pending:
    res = client.wait_for_result(res.run_id, timeout=120)
res.raise_for_status()
```

`client.eval` / `client.batch_eval` add scorers (`agnt5-experiments`). Mark these
`integration` and run them in CI only after `agnt5 deploy --env preview`.
