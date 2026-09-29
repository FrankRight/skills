# Python SDK pitfalls

Verified against `agnt5` 0.13.6 on 29 Sep 2026 — from the SDK source, introspection of the
installed package, and live runs where an entry says so. Ordered by severity. Each entry gives
the symptom, the cause, the workaround, the Linear issue (if any) and when to remove it.
Entries without a Linear issue are documented behaviour that trips people up; keep those
until the SDK changes.

## 1. gpt-6 models reject the default temperature (AGNT5-1323)

- **Symptom**: `Agent(model="openai/gpt-6-luna", ...)` — or any gpt-6 model — fails with a
  400 that mentions `temperature`.
- **Cause**: `Agent` sends `temperature=0.7` unless told otherwise. Only `openai/gpt-5*`, `o1`
  and `o1-*` are treated as reasoning models that get no temperature; gpt-6 accepts only its
  default.
- **Workaround**: `Agent(..., temperature=None)` sends nothing (the official templates do
  this); `temperature=1` also works. With `lm.generate`, leave `temperature` unset. Whether
  the SDKs will expose gpt-6's other parameters is a pending decision (AGNT5-1285).
- **Remove when**: AGNT5-1323 ships.

## 2. `reasoning_effort`, `modalities` and `store` never reach OpenAI (AGNT5-1370, AGNT5-1327)

- **Symptom**: `lm.generate(..., reasoning_effort=ReasoningEffort.HIGH)` changes nothing.
  Passing the string `"high"` fails later with `AttributeError: 'str' object has no attribute
  'value'`; `ReasoningEffort("low")` raises `ValueError`.
- **Cause**: the Python layer serialises the three fields into the request config, but the
  native binding (`_core`) has no code path for them, so they are dropped before the HTTP
  call. The enum only has `MINIMAL`, `MEDIUM`, `HIGH`.
- **Workaround**: do not teach or rely on reasoning effort from Python; choose the model tier
  instead. Never pass strings.
- **Remove when**: AGNT5-1370 (binding forwards the fields) and AGNT5-1327 (enum values) ship.

## 3. Structured output is never parsed: `structured_output` / `parsed` / `object` are `None` (AGNT5-1371)

- **Symptom**: after `await lm.generate(..., response_format=MyModel)`,
  `response.structured_output`, `.parsed` and `.object` are all `None` although
  `response.text` is valid JSON.
- **Cause**: the properties read `_rust_response.object`, which the native response never
  populates in 0.13.6.
- **Workaround**: `MyModel.model_validate_json(response.text)` (or `json.loads`). When
  `response_format` is a Pydantic model, add `model_config = ConfigDict(extra="forbid")`: the
  schema is taken straight from `model_json_schema()`, which emits `additionalProperties:
  false` only with `extra="forbid"`, and OpenAI strict mode returns a 400 without it.
- **Remove when**: AGNT5-1371 ships.

## 4. Retries are not applied inside `ctx.step` (AGNT5-1372)

- **Symptom**: `@function(retries=3)` retries when run on its own (`agnt5 run`, `Client.run`),
  but a workflow calling it through `ctx.step(fn, ...)` fails on the first exception
  (live-verified).
- **Cause**: retries are executed by the platform, and step activations do not carry the
  function's policy.
- **Workaround**: retry inside the function body (loop plus backoff). Remember that
  `retries=N` is N attempts **in total** (`RetryPolicy(max_attempts=N)`), and that
  `agnt5 run <function>` prints the first failed attempt and exits 1 while the platform keeps
  retrying — check `agnt5 inspect runs describe <run-id>` for the real outcome.
- **Remove when**: AGNT5-1372 ships.

## 5. LLM judges on gpt-6 models score 0 instead of failing (AGNT5-1374 covers Go; Python behaves the same)

- **Symptom**: `Correctness()`, `LLMJudge(...)` and the other presets with a gpt-6 `model`
  return `score=0.0, passed=False` for every item; `explanation` starts with
  `LLM call failed:`.
- **Cause**: presets default to `temperature=0.0`, gpt-6 rejects it, and `llm_judge` converts
  any exception into a zero score.
- **Workaround**: keep judges on a non-gpt-6 model (`openai/gpt-4o-mini`, the default, is
  fine). Grep results for `LLM call failed` before trusting a 0.
- **Remove when**: the judge stops sending a temperature to gpt-6 (AGNT5-1374 / AGNT5-1323).

## 6. Triggered workflows receive the gateway envelope, with four keyword arguments

- **Symptom**: a webhook/event-triggered workflow declared `async def h(ctx, event: dict)`
  fails with `TypeError: got an unexpected keyword argument 'deployment_id'`
  (live-verified), or `event["body"]` raises `KeyError`.
- **Cause**: the gateway invokes the workflow with `event`, `deployment_id`, `target_kind`
  and `target_ref`. `event` is `{id, name, data, source, timestamp_ns}`; the webhook envelope
  (`_webhook, source, integration_id, event_type, idempotency_key, timestamp, headers, body`)
  is at `event["data"]`, and `body` is the raw request body string.
- **Workaround**: `async def h(ctx: WorkflowContext, event: dict, **_)` and
  `json.loads(event["data"]["body"])`. Leave `filter_expression`, `input_mapping`,
  `batch_window_ms` and `delay_expression` unset: the gateway skips a trigger that sets any of
  them (`skipped_unsupported_count`, AGNT5-1376) and their syntax is undocumented.
- **Remove when**: the envelope or the gateway's trigger support changes (documented
  behaviour, no Linear issue).

## 7. `client.run()` returns a pending receipt after `wait_timeout` (since 0.13.0)

- **Symptom**: `res.output` is `None` and `res.status_code == 202` for runs longer than
  300 s.
- **Cause**: 0.13.0 made `run()` return an accepted receipt instead of polling to completion;
  the run keeps executing server-side.
- **Workaround**: `if res.is_pending: res = client.wait_for_result(res.run_id, timeout=...)`
  on the sync `Client`; `AsyncClient` has no `wait_for_result` — poll `get_status` /
  `get_result`. Raise `wait_timeout` (max 86400) for long runs. Related: `api_key` must start
  with `agnt5_sk_` (`ValueError` otherwise) and `component_type` defaults to `"function"`.
- **Remove when**: never (documented); keep as a reminder.

## 8. `ctx.user` is `None` without a `user_id`

- **Symptom**: `AttributeError: 'NoneType' object has no attribute 'state'` on
  `ctx.user.state`; `ctx.memory.user.save(...)` raises `RuntimeError: User-scoped memory
  requires a user_id`.
- **Cause**: `WorkflowContext.user` returns `None` unless the run carries a `user_id`.
- **Workaround**: pass `user_id=` (and `session_id=`) to `Client.run` or
  `Client.stream_events` (`submit` has neither); they travel as `X-User-ID` / `X-Session-ID`
  headers. `user_id` / `session_id` keys in the input payload are honoured too, but the
  worker does not strip them, so the handler must accept those parameters. `agnt5 run` has
  no flag for either. Guard with `if ctx.user:`.
- **Remove when**: never (documented).

## 9. Calling a `@function` with a `WorkflowContext` raises — it does not "silently skip checkpointing"

- **Symptom**: `TypeError: Function 'x' requires FunctionContext as first argument. Usage:
  await x(ctx, ...)` inside a workflow (live-verified).
- **Cause**: the function wrapper type-checks its first argument. Older docs and skills
  claimed the call would run un-checkpointed.
- **Workaround**: `await ctx.step(x, ...)`. Plain coroutines that are not `@function`s do run
  un-checkpointed — wrap them in `ctx.step("name", coro)`.

## 10. `ctx.step(fn, key=...)` consumes a parameter named `key`

- **Symptom**: `TypeError: fn() missing 1 required positional argument: 'key'`
  (live-verified).
- **Cause**: `key=` is `ctx.step`'s own checkpoint-key parameter.
- **Workaround**: rename the function parameter, or pass it positionally.

## 11. Context detection is by name for functions and by annotation for tools

- **Symptom**: a `@function` whose first parameter is not named `ctx` shifts its arguments
  (`TypeError: missing 1 required positional argument`). A `@tool` with
  `ctx: WorkflowContext`, `ctx: str` or no `ctx` at all raises
  `ConfigurationError: Tool function 'x' first parameter must be 'ctx: Context'`
  (live-verified).
- **Cause**: `@function` sets `needs_context` from `params[0].name == "ctx"`; `@tool`
  requires the annotation to be exactly `agnt5.context.Context` (or absent). `@workflow`
  always passes the context first but only excludes a parameter named `ctx` from the input
  schema.
- **Workaround**: always `ctx: FunctionContext` / `ctx: WorkflowContext` / `ctx: Context`.

## 12. Direct workflow calls in tests take keyword arguments only

- **Symptom**: `await my_workflow("x")` fails with `TypeError: my_workflow() missing 1
  required positional argument: 'name'` (live-verified); `await my_workflow(name="x")` works.
- **Cause**: the auto-context wrapper forwards `**kwargs` only and drops positional args.
- **Workaround**: keywords. See `agnt5-testing`.

## 13. `ctx.batch` / `ctx.map` defaults

- **Symptom**: items time out after 30 s; a function called with a scalar item receives
  `value=` instead of its own parameter name.
- **Cause**: `timeout_per_item` defaults to 30 s; non-dict items are wrapped as
  `{"value": item}`.
- **Workaround**: pass dict items matching the function signature (or accept `value`); set
  `timeout_per_item` explicitly. `map` is `batch(..., continue_on_failure=False)`.

## 14. Pauses are `BaseException`s

- **Symptom**: a workflow with a bare `except:` (or `except BaseException:`) around
  `ctx.wait_for_user()`, `ctx.sleep()`, or an agent/tool that pauses never pauses — it
  continues with no answer.
- **Cause**: `WaitingForUserInputException` and `DurableSleepSuspension` subclass
  `BaseException`, so they pass `except Exception` but not a bare `except:`.
- **Workaround**: catch `Exception` only in workflow code and tools.

## 15. No non-retryable error type

- **Symptom**: a permanent failure (bad input) is retried N times before failing.
- **Cause**: `agnt5.exceptions` has no "do not retry" exception; the platform retries every
  exception per policy.
- **Workaround**: validate before side effects, or return an error value instead of raising
  when the failure is permanent.

## 16. Memory: silent no-ops and missing accessors

- **Symptom**: `ctx.memory.user.save(...)` returns `None` and `search()` returns `[]` with no
  error; `ctx.memory` / `ctx.conversation` on a `FunctionContext` raise `AttributeError`;
  `ctx.memory.working` data shows up in other runs of the same session.
- **Cause**: the default `AGNT5_MEMORY_FAILURE_POLICY=best_effort` swaps in a disabled
  service when memory is not configured; only `WorkflowContext` defines `memory` /
  `conversation`; working memory is session-scoped, not per-run.
- **Workaround**: set `AGNT5_MEMORY_FAILURE_POLICY=require_memory_or_fail` where memory must
  work; use `ctx.memory` from the workflow (or pass `context=ctx` to the agent); use
  `ctx.memory.run` for per-run data.

## 17. Agents close (and destroy) their sandbox after every run

- **Symptom**: files written during one `agent.run()` are gone on the next; concurrent runs
  of an agent that shares a module-level `Sandbox()` interfere with each other.
- **Cause**: `Agent.run` / `stream` call `sandbox.close()` in a `finally`; with
  `auto_destroy=True` (default) the provider sandbox is destroyed and the `Sandbox` object
  forgets it, so the next run creates a new one. One module-level object is shared by all
  concurrent runs.
- **Workaround**: treat a sandbox as per-run scratch space and create `Sandbox()` inside the
  workflow when runs can overlap. `auto_destroy=False` skips the destroy but still closes and
  forgets the sandbox — it leaks, it does not persist.

## 18. Sandbox provider auto-detection

- **Symptom**: `SandboxProviderError vercel from_env: VERCEL_TEAM_ID and VERCEL_PROJECT_ID
  are required with VERCEL_TOKEN` on **every** `Sandbox()`, even `provider="e2b"`
  (live-verified); sandboxes unexpectedly run on Together; `AGNT5_SANDBOX_PROVIDER` has no
  effect.
- **Cause**: `load_providers_from_env()` builds every configured provider on each sandbox
  start and raises on a half-configured one; `TOGETHER_API_KEY` (usually set for LLM calls)
  also registers the Together sandbox provider; `provider="auto"` takes the first configured
  provider in the order e2b, daytona, vercel, northflank, together; nothing reads
  `AGNT5_SANDBOX_PROVIDER`.
- **Workaround**: pass `provider=` explicitly; keep the Vercel variables complete or absent.

## 19. `Agent(model_config=ModelConfig(...))` is ignored

- **Symptom**: `base_url`, `api_key`, `timeout` and `headers` in `ModelConfig` have no
  effect.
- **Cause**: `Agent.__init__` stores `model_config` and nothing reads it. The docs still list
  it as the custom-endpoint option.
- **Workaround**: provider env vars read by the native layer — `OPENAI_BASE_URL`,
  `ANTHROPIC_BASE_URL`, `OPENROUTER_BASE_URL`, `DEEPSEEK_BASE_URL`, `MOONSHOT_BASE_URL`,
  `TOGETHER_BASE_URL`, `OPENAI_ORGANIZATION`, `OPENAI_PROJECT`,
  `OPENAI_REQUEST_TIMEOUT_SECS`. `temperature` / `max_tokens` / `top_p` on `Agent` are the
  sampling knobs that work.

## 20. Runtime overrides apply to `lm.generate` only

- **Symptom**: `ctx.runtime.llm.model = ...` or `ctx.runtime.prompts[...]` changes nothing
  for an `Agent` or `lm.stream`.
- **Cause**: only `lm.generate` consults the runtime options.
- **Workaround**: route calls that must be overridable through `lm.generate`.

## 21. Import traps

- `from agnt5 import Event` gives `agnt5.responses.Event` (the client-side event record),
  not the `agnt5.events.Event` base class — import that one from `agnt5.events`.
- `agnt5.eval.ScorerResult` is the Rust result type returned by the local scorer functions;
  a deployable `@scorer` returns `agnt5.ScorerResult` (also `agnt5.eval.ScorerResultPy`).
  Do not mix them.
- `webhook` is importable (`from agnt5 import webhook`) but missing from `agnt5.__all__`;
  `event` is listed.
- `agnt5.agent` is an older function decorator that builds an `Agent` from a factory (its
  docstring still shows `model_name` and `OpenAILanguageModel`); the documented pattern is
  `Agent(...)` instances registered with `Worker(agents=[...])`.
- Docstrings in `workflow.py` still show `ctx.task(...)`; it is deprecated — use `ctx.step`.

## 22. Serverless workflows get a different context

- **Symptom**: `AttributeError` on `ctx.parallel` / `gather` / `batch` / `map`, or on
  `ctx.step(fn, *args)`, under `ServerlessApp`.
- **Cause**: `serverless.py` provides `WorkerlessContext` with `step(name, fn_or_awaitable)`,
  `get` / `set` / `delete`, `sleep`, `wait_for_user`, `wait_for_signal`, `emit` and
  `yield_if_needed` only.
- **Workaround**: see `agnt5-serverless`.

## 23. Streaming deltas are transient

- `output.delta`, `lm.*.delta` and thinking deltas are not stored and are not replayed on
  reconnect; only lifecycle, step and tool/model boundary events are durable. Return the
  final result from streaming functions and never use deltas as an audit record.

## 24. The worker image is Python 3.14; `prompts/` and `skills_dir` resolve against the working directory

- **Symptom**: a package that installs into your local 3.12 venv fails in the deployed worker
  (Python 3.14 live-observed); `Prompt(id=...)` fails closed in production; `Skill 'x' not
  found`.
- **Cause**: the managed image is `ghcr.io/agnt5dev/python-worker:3.14`; the prompt manifest
  searches `Path.cwd()/prompts`; a relative `skills_dir` resolves against the process cwd;
  gitignored directories are left out of the code bundle.
- **Workaround**: keep `prompts/` and `skills/` committed and not ignored; build `skills_dir`
  from `__file__`; test with 3.14 locally.

## 25. The `Agent` docstring says built-in `WEB_SEARCH` is OpenAI-only

- Stale: the SDK core maps `BuiltInTool.WEB_SEARCH` for OpenAI (`web_search_preview`),
  Anthropic (`web_search_20260209`) and Gemini (`google_search`). Only OpenAI was live-tested.

## 26. HITL answer formats and resuming from a backend

- `multiselect` returns a JSON array string (`'["a","c"]'`), not a comma-separated list —
  `json.loads` it. Skip becomes `None` only for the answer `"__skipped__"` (what Studio
  sends); posting `{"user_response": null}` to `POST /v1/workflows/resume/{run_id}` delivers
  the string `"null"`. No `Client` method resumes a run in 0.13.6 — call the endpoint with
  the `X-API-KEY` header. Formats live-verified for TypeScript/Go; Python not separately.

## 27. An explicit `Worker(...)` registers only the scorers you list

- `Worker(workflows=[...], functions=[...])` without `scorers=[...]` never registers your
  `@scorer` functions (only the built-in scorer names appear). Add `scorers=[...]` or use
  `auto_register=True`. `max_concurrency` defaults to 100 (`AGNT5_MAX_CONCURRENCY`).

## 28. Judge preset `include_input` defaults

- `Faithfulness`, `Coherence`, `Conciseness` and `LLMJudge` default to
  `include_input=False`; the other presets default to `True`. Pass it explicitly when the
  criteria depend on the input.

## 29. Capture settings are read once; an invalid content mode falls back to metadata-only

- `AGNT5_CAPTURE*` is read when `Worker` / `ServerlessApp` boots; an invalid
  `AGNT5_CAPTURE_CONTENT_MODE` logs a warning and uses `metadata-only`, so prompt and
  response text silently disappear from traces.
