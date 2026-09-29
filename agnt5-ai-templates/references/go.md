# Go templates

Verified against `github.com/agnt5dev/sdk-go` **v0.10.3** and the shipped Go templates
(`quickstart`, `weather-agent`, `code_reviewer`, `hitl_deep_research`, `tutor_agent`,
`travel_booking_customer_service`, `coding_agent`). Check the current version first:
`go list -m -versions github.com/agnt5dev/sdk-go`.

## Layout

```
<template-name>/
├── main.go                    # builds models + agents, registers every component, runs the worker
├── go.mod                     # module <template-name>
├── go.sum                     # commit it — the managed build needs it
├── agnt5.yaml
├── .env.example
├── README.md
└── src/<package_name>/        # package <package_name> (snake_case)
    ├── agents.go              # NewAgents(model) sets package-level *agnt5.Agent vars
    ├── tools.go               # only if agents need custom tools (New<X>Tool() functions)
    ├── functions.go           # step handlers (RegisterFunction shape) — only for distinct stages
    ├── workflows.go           # workflow handlers
    └── models.go              # shared input/output structs, constants (optional)
```

`main.go` imports `"<template-name>/src/<package_name>"`; when the module name has hyphens,
alias it: `hitl_deep_research "hitl-deep-research/src/hitl_deep_research"`. Everything
`main.go` touches is exported (capitalised).

## `go.mod`

```
module <template-name>

go 1.26.5

require github.com/agnt5dev/sdk-go v0.10.3
```

Run `go mod tidy` after writing the code (fills `go.sum` and indirect requirements), then
`go build ./...`. Keep the `go` directive ≤ 1.26.8 — the managed worker image is go1.26.8.

## `agnt5.yaml` and `.env.example`

```yaml
name: <template-name>
language: go
language_version: "1.26"
environment: dev

worker:
  command: "go run ."

deploy:
  resources:
    memory: 512Mi
    cpu: 500m
```

`.env.example`: one line per key, e.g. `OPENAI_API_KEY="your-openai-api-key-here"`. `agnt5 dev`
loads `.env`; the SDK itself never reads provider keys — the code passes them explicitly.

## `main.go`

```go
package main

import (
    "context"
    "log"
    "os"

    "github.com/agnt5dev/sdk-go/agnt5"

    "<template-name>/src/<package_name>"
)

func must(err error) {
    if err != nil {
        log.Fatal(err)
    }
}

func main() {
    model := agnt5.NewOpenAIModel(agnt5.OpenAIConfig{
        Model:  "gpt-4o-mini",                  // bare name — never "openai/gpt-4o-mini"
        APIKey: os.Getenv("OPENAI_API_KEY"),    // not read from the environment automatically
    })
    if err := <package_name>.NewAgents(model); err != nil {
        log.Fatal(err)
    }

    worker := agnt5.NewWorker("<template-name>", agnt5.WithServiceVersion("1.0.0"))

    // No auto-register in Go: list every agent, tool, function, scorer and workflow here.
    must(agnt5.RegisterAgent(worker, <package_name>.MyAgent))
    must(agnt5.RegisterTool(worker, <package_name>.MyTool))               // publishes the schema, runnable alone
    must(agnt5.RegisterFunction(worker, "my_stage", <package_name>.MyStage,
        agnt5.WithRetry(3, 1000, 30000), agnt5.WithBackoff("exponential", 2.0)))
    must(agnt5.RegisterWorkflow(worker, "<template_name>_workflow", <package_name>.MyWorkflow))

    if err := worker.Run(context.Background()); err != nil {
        log.Fatal(err)
    }
}
```

## Agents (`agents.go`)

```go
package <package_name>

import "github.com/agnt5dev/sdk-go/agnt5"

const myAgentPrompt = `You are <AgentName>, <one-line role>.

Your responsibilities:
1. ...

Output format:
LABEL:
[structured output]`

var (
    MyAgent *agnt5.Agent
    MyTool  agnt5.Tool
)

func NewAgents(model agnt5.LanguageModel) error {
    var err error
    if MyTool, err = NewMyTool(); err != nil {
        return err
    }
    MyAgent, err = agnt5.NewAgent("AgentName",
        agnt5.WithAgentModel(model),
        agnt5.WithAgentInstructions(myAgentPrompt),
        agnt5.WithAgentTools(MyTool),   // omit when there are no tools
        agnt5.WithAgentMaxTurns(5),     // default 10; ErrAgentMaxTurnsExceeded when exhausted
    )
    return err
}
```

Other options only when needed: `WithAgentHandoffs(h...)` (from `agnt5.NewHandoff(agent,
agnt5.WithHandoffDescription("..."))`), `WithAgentSandbox(runner)`, `WithAgentPromptCache(agnt5.EnablePromptCache())`,
`WithAgentSkillsFromDir("./skills", names...)`, `WithAgentGuidance("./AGENTS.md")`. There is
no temperature, max_tokens, built-in tools, callbacks or streaming on `NewAgent` — omit rather
than fake (`agnt5-agents-tools/references/go.md`).

## Tools (`tools.go`) — only if needed

```go
func NewMyTool() (agnt5.Tool, error) {
    return agnt5.NewTool("my_tool", func(c context.Context, args map[string]any) (any, error) {
        param, _ := args["param"].(string)          // numbers arrive as float64
        if ctx, ok := c.(*agnt5.Context); ok {
            ctx.Logger().Info("my_tool called", "param", param)
        }
        return doWork(c, param)                     // any JSON-serialisable value, or an error the model sees
    },
        agnt5.WithToolDescription("One line the model reads to decide when to call this."),
        agnt5.WithToolSchema(map[string]any{        // required — without it the model sees no parameters
            "type": "object",
            "properties": map[string]any{
                "param": map[string]any{"type": "string", "description": "Description of the parameter."},
            },
            "required": []string{"param"},
        }),
    )
}
```

## Functions (`functions.go`) — only for distinct stages

```go
type MyStageInput struct {
    Text string `json:"text"`
}

func MyStage(ctx *agnt5.Context, in MyStageInput) (string, error) {
    result, err := MyAgent.Run(ctx, agnt5.AgentInput{Message: in.Text})
    if err != nil {
        return "", err
    }
    return strings.TrimSpace(strings.TrimPrefix(result.Response, "LABEL:")), nil
}
```

Same signature serves `RegisterFunction` and `agnt5.Task`. Retries/backoff live on the
registration; there is no timeout option (use `context.WithTimeout`). Streaming: `ctx.Output`.

## Workflows (`workflows.go`)

```go
type MyWorkflowInput struct {
    Message string `json:"message"`
}

type MyWorkflowOutput struct {
    Status string `json:"status"`
    Output string `json:"output"`
}

func MyWorkflow(ctx *agnt5.Context, in MyWorkflowInput) (MyWorkflowOutput, error) {
    stage1, err := agnt5.Task(ctx, "stage1", MyStageInput{Text: in.Message}, MyStage) // checkpointed, Function node
    if err != nil {
        return MyWorkflowOutput{}, err
    }
    stage2, err := agnt5.Step(ctx, "stage2", func(context.Context) (string, error) {  // checkpointed closure
        res, err := OtherAgent.Run(ctx, agnt5.AgentInput{Message: stage1})
        if err != nil {
            return "", err
        }
        return res.Response, nil
    })
    if err != nil {
        return MyWorkflowOutput{}, err
    }
    return MyWorkflowOutput{Status: "completed", Output: stage2}, nil
}
```

Fan-out over N items (Go has no `ctx.parallel/batch/map`; the older "no per-iteration
primitive" comment is stale — keyed steps are the primitive):

```go
sem := make(chan struct{}, 5)
out := make([]Summary, len(items))
errs := make([]error, len(items))
var wg sync.WaitGroup
for i, item := range items {
    wg.Add(1)
    go func(i int, item Item) {
        defer wg.Done()
        sem <- struct{}{}
        defer func() { <-sem }()
        out[i], errs[i] = agnt5.TaskWithKey(ctx, "summarize", item.ID, SummarizeInput{Item: item}, Summarize)
    }(i, item)
}
wg.Wait()
if err := errors.Join(errs...); err != nil { return MyWorkflowOutput{}, err }
```

Durable sleep, state, idempotency: `agnt5-workflows`. HITL (`ctx.AskUser` — return its error):
`agnt5-human-in-the-loop`. Triggers: `agnt5-webhooks-integrations`. Scorers:
`RegisterScorer(worker, agnt5.ScorerConfig{...})` — `agnt5-scorers`.

## Models and providers

Bare model names, explicit keys:

| Constructor | Config | Key env var |
|---|---|---|
| `NewOpenAIModel(OpenAIConfig{Model, APIKey})` | `OpenAIConfig{BaseURL, APIKey, Model, Organization, Headers, HTTPClient, Path, APIKeyHeader, AuthScheme}` | `OPENAI_API_KEY` |
| `NewAnthropicModel(AnthropicConfig{Model, APIKey})` | `AnthropicConfig{BaseURL, APIKey, Model, Version, HTTPClient}` | `ANTHROPIC_API_KEY` |
| `NewGoogleModel` / `NewGeminiModel(GoogleConfig{Model, APIKey})` | `GoogleConfig{BaseURL, APIKey, Model, Version, HTTPClient}` | `GOOGLE_API_KEY` |
| `NewAzureOpenAIModel(AzureOpenAIConfig{Endpoint, APIKey, Deployment, APIVersion})` | | `AZURE_OPENAI_API_KEY` + endpoint |
| `NewOpenRouterModel`, `NewGroqModel`, `NewDeepSeekModel`, `NewMistralModel`, `NewTogetherModel`, `NewXAIModel`, `NewMoonshotModel`, `NewOllamaModel` | all `OpenAIConfig` | `OPENROUTER_API_KEY`, `GROQ_API_KEY`, `DEEPSEEK_API_KEY`, `MISTRAL_API_KEY`, `TOGETHER_API_KEY`, `XAI_API_KEY`, `MOONSHOT_API_KEY`, none |

Not in Go: Bedrock, Hugging Face, Fireworks constructors (use `NewOpenAIModel` with `BaseURL`
for OpenAI-compatible endpoints). Only the Google model strips a `google/`/`gemini/` prefix;
every other provider sends the string verbatim, so `"openai/gpt-4o-mini"` is a 400
`invalid model ID`.

Warnings:

- Default to `gpt-4o-mini` (or `gpt-4.1-mini`). gpt-6 / reasoning models: tool-using agents fail
  over Chat Completions without `reasoning_effort`, which Go cannot set (AGNT5-1325); an
  agent-level `Temperature`/`MaxTokens` would be rejected anyway (AGNT5-1303 unmerged); product
  decision pending (AGNT5-1285). Workaround if the user insists: an `OpenAIConfig.HTTPClient`
  whose `Transport` rewrites the JSON body.
- The Anthropic provider always sends `max_tokens: 1024` unless a `GenerateRequest` sets
  `MaxTokens`; agents cannot, so Anthropic-backed agent replies are capped at 1024 tokens.
  Wrap the model (a type whose `Generate` sets `request.MaxTokens` then delegates) when longer
  output matters.
- `NewAgent` without `WithAgentModel` returns `agnt5.ErrAgentModelRequired`.

## Write order

`src/<package_name>/tools.go` → `agents.go` → `functions.go` → `workflows.go` → `main.go` →
`go.mod` (then `go mod tidy && go build ./...`) → `agnt5.yaml`, `.env.example`, `README.md`.
Naming: agent `name` in `PascalCase` or `snake_case` (consistent within the template), Go
identifiers `PascalCase`, component names `snake_case`, workflow `<template_name>_workflow`.
Hand off to `agnt5-project-init` (`go mod tidy`, `.env`, `agnt5 dev`, `agnt5 run`).
