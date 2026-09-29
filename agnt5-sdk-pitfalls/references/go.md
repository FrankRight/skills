# Go SDK pitfalls

Verified against `sdk-go` **v0.10.3** on 29 Sep 2026 (source at `github.com/agnt5dev/sdk-go`,
live runs on managed infrastructure). Ordered by severity: each entry is symptom → cause →
workaround → tracking → when to remove. Entries without a Linear ID are SDK design facts, not
bugs; keep them until the SDK changes.

## 1. Provider `400 invalid model ID` on every model call

Cause: the model name carries a `provider/` prefix (`"openai/gpt-4o-mini"`, copied from Python).
Go providers send the string verbatim; only `NewGoogleModel` strips `google/`/`gemini/`.
Workaround: bare names — `agnt5.OpenAIConfig{Model: "gpt-4o-mini"}`; the provider is the
constructor. Tracking: none (by design). Remove: never.

## 2. Provider `401`/authentication error although the key is set

Cause: model constructors never read `OPENAI_API_KEY` etc.; `APIKey` was left empty, or the
process was started with `go run .` outside `agnt5 dev`, which is what loads `.env`.
Workaround: `APIKey: os.Getenv("OPENAI_API_KEY")` explicitly, validate at startup, run under
`agnt5 dev` or export the vars. Deployed: `agnt5 secrets set --name OPENAI_API_KEY`.
Tracking: none (by design). Remove: never.

## 3. Workflow never pauses; `AskUser` "answer" is empty

Cause: the first `ctx.AskUser`/`ctx.RequestApproval` call returns
`("", *agnt5.WaitingForUserInputError)` and the handler ignored or swallowed the error.
`ctx.Sleep` under the durable runtime behaves the same way. Workaround: `if err != nil { return
Out{}, err }` (wrapping with `%w` is fine; `agnt5.IsWaitingForUserInput(err)` to test).
Tracking: none (by design). Remove: never.

## 4. gpt-6 / reasoning models: tool-using agents fail; explicit params rejected

Symptom: agents with tools error over Chat Completions; a `GenerateRequest` with
`Temperature`/`MaxTokens` is rejected. Cause: Go cannot send `reasoning_effort`
(AGNT5-1325); reasoning models reject sampling params (AGNT5-1303, fix unmerged); product
decision on extra params pending (AGNT5-1285). Workaround: `gpt-4o-mini`/`gpt-4.1-mini`, or an
`OpenAIConfig.HTTPClient` with a `RoundTripper` that rewrites the JSON body. Remove when
AGNT5-1325/AGNT5-1285 ship a Go option.

## 5. LLM judge scorer fails with `invalid model ID` or a 400 on gpt-6

Cause: a raw `EvalScorerSpec{Name: "llm_judge", Config: {"model": "openai/gpt-4.1-mini"}}` is
sent verbatim (presets and `NewLLMJudge` split the prefix); any gpt-6 judge fails because the
judge always sends `temperature: 0`. Workaround: bare `"model": "gpt-4.1-mini"` (+ optional
`"provider"`), non-reasoning judge models; the judge runs in your worker and needs
`OPENAI_API_KEY`/`ANTHROPIC_API_KEY`/`GOOGLE_API_KEY` in its env. Tracking: AGNT5-1374.
Remove when AGNT5-1374 is fixed.

## 6. Anthropic-backed agents stop mid-answer

Cause: `AnthropicModel.Generate` sends `max_tokens: 1024` unless the request sets `MaxTokens`,
and `NewAgent` has no max-tokens option. Workaround: a `LanguageModel` wrapper that sets
`req.MaxTokens` before delegating (`agnt5-agents-tools/references/go.md`), or `ctx.Generate`
with `MaxTokens` for long outputs. Tracking: none known. Remove when `NewAgent` gains a
max-tokens option or the provider default changes.

## 7. Tool is called with empty arguments

Cause: `agnt5.NewTool` without `agnt5.WithToolSchema` — the provider is shown
`{"type":"object","properties":{}}`. Workaround: always pass a JSON Schema with `properties`
and `required` (MCP tools: `WithToolSchema(t.InputSchema)`). Tracking: none (by design).
Remove: never.

## 8. Step/state values come back empty or zero on replay

Cause: checkpoints and `ctx.State()` values are JSON: unexported struct fields encode as `{}`,
and numbers decode as `float64` (`raw.(int)` fails). Workaround: exported fields with `json`
tags for every step result type; decode numbers as `float64` or store strings. Tracking: none.
Remove: never.

## 9. Fan-out or branch replays the wrong checkpoint / `ErrNondeterministicReplay`

Cause: unkeyed `Step`/`Task` inside goroutines, loops or `if` branches — keys come from call
order. Workaround: `StepWithKey`/`TaskWithKey`/`WithSleepKey` with a stable per-item key (an ID,
not a slice index). There is no `ctx.parallel/batch/map`; use goroutines + `sync.WaitGroup` +
a semaphore. Tracking: none. Remove: never.

## 10. Retries configured with `WithRetry` do not happen inside a workflow

Cause: `WithRetry`/`WithBackoff` are registration metadata for direct invocations. In Python and
TypeScript retries are not applied to a function called inside a workflow step (AGNT5-1372);
Go step-level behaviour was **not tested**. Workaround: retry inside the step body when it
matters. Tracking: AGNT5-1372. Remove when AGNT5-1372 is resolved and Go is verified.

## 11. Managed deploy fails or stays "starting" for minutes

Cause: managed Go workers compile at pod start (`go mod download` + `go build ./...` on
go1.26.8 linux/amd64); a stale `go.sum` or a `go` directive above 1.26.8 fails there, and every
cold replica/eval worker pays the build time. Workaround: `go mod tidy && go build ./...`
before every deploy; keep `go 1.26.5`; wait for Ready before `agnt5 run --env`/experiments.
Tracking: none. Remove when the platform pre-builds Go images.

## 12. Webhook handler finds no `body`

Cause: trigger input is the outer event record; the webhook envelope (`body`, `headers`,
`idempotency_key`, ...) is at `event.data`, not top level (the product docs' Go example is
wrong). Workaround: typed struct with `Event.Data.Body`, or
`event["event"].(map[string]any)["data"].(map[string]any)["body"].(string)`. Tracking: none
filed for the docs. Remove when the docs match.

## 13. HITL multiselect/skip answers look wrong

Cause: via the resume API a multiselect answer arrives as a JSON array **string**
(`["a","c"]`), and a JSON `null` skip arrives as the literal `"null"` (SDK-side `__skipped__`
becomes `""`). Workaround: `json.Unmarshal` first, fall back to a comma split; treat `""` and
`"null"` as skipped. Tracking: none filed (live test). Remove when the gateway normalises the
shapes.

## 14. `client.Run` returns 404 for a workflow

Cause: `Run`/`Submit`/`Eval` default to the `functions` collection. Workaround:
`agnt5.WithRunComponentType(agnt5.ComponentTypeWorkflow)` / `WithSubmitComponentType` /
`EvalRequest.ComponentType`. Tracking: none. Remove: never.

## 15. `client.BatchEval` "succeeds" but items failed

Cause: `BatchEval` returns `*BatchEvalResult` and no error; per-item transport errors are in
`Results[i].Error`. Workaround: check `Status`/`FailedItems()` as well as `FailingItems()`.
Tracking: none. Remove: never.

## 16. Session memory is not shared between runs

Cause: `ctx.Memory()` session/user namespaces come from the run's `session_id`/`user_id`
metadata; without them they fall back to the run ID. Workaround: `client.Session(id)`,
`agnt5.WithRunSessionID`, or `session_id` on chat inputs. Tracking: none. Remove: never.

## 17. `ScorerRequest.Input["key"]` does not compile

Cause: `Input`/`Output`/`Expected` are `any` (the product-docs Go snippet indexes `Input`).
Workaround: `in, _ := req.Input.(map[string]any)`. Tracking: none filed for the docs.
Remove when the docs are corrected.

## 18. Handoff specialist lacks the conversation

Cause: `NewHandoff` defaults `PassFullHistory` to `false` (Python defaults to `True`).
Workaround: `agnt5.WithHandoffFullHistory(true)`. Tracking: none. Remove if the default
changes.

## 19. `ErrAgentMaxTurnsExceeded` loses everything the agent did

Cause: the error carries no partial result; `WithAgentMaxTurns(0)` silently means 10.
Workaround: record findings from tools into `ctx.Memory().KV(agnt5.MemoryScopeRun)` (see the
`hitl_deep_research` template) and size `MaxTurns` for the longest tool chain. Tracking: none.
Remove when the result is returned alongside the error.

## 20. Agent output does not stream; provider SDK calls missing from traces

Cause: built-in providers implement no `Stream`; only calls through an `agnt5.LanguageModel`
(`agent.Run`, `ctx.Generate`) are traced — there is no `AGNT5_CAPTURE*` auto-capture in Go.
Workaround: `ctx.Output` after the run; route vendor calls through a `LanguageModel`
implementation or at least a `Step`. Tracking: none. Remove when streaming providers land.

## 21. Product docs say Go lacks features it has

Skills/AGENTS.md (`WithAgentSkillsFromDir`, `WithAgentGuidance`, `DiscoverAgentsMD`), evaluator
presets (`agnt5.Correctness{}` ...), `client.BatchEval`, `agnt5.TraceScorer`/`TraceAssertion`,
`WithAgentSandbox` + `SandboxTools`, and the full built-in deterministic scorer set all exist in
v0.10.3. Trust `go doc github.com/agnt5dev/sdk-go/agnt5 <Symbol>` over the docs. Tracking:
none filed for the docs (these references are the correction). Remove when the docs are
corrected.
