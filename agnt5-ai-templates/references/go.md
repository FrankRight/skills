# Go templates

Verified against `github.com/agnt5dev/sdk-go` **v0.10.3**. Check the current version first:
`go list -m -versions github.com/agnt5dev/sdk-go`.

## Layout

```
<template-name>/
├── main.go        # worker, functions, workflows, agents — all inline
├── go.mod
├── agnt5.yaml
├── .env.example
└── README.md
```

## `go.mod`

```
module <template-name>

go 1.26

require github.com/agnt5dev/sdk-go v0.10.3
```

Run `go mod tidy` after writing `main.go` to fill in `go.sum` and indirect requirements.

## `agnt5.yaml`

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

## `main.go`

```go
package main

import (
    "context"
    "log"
    "os"

    "github.com/agnt5dev/sdk-go/agnt5"
)

type MyInput struct {
    Message string `json:"message"`
}

type MyOutput struct {
    Result string `json:"result"`
}

func main() {
    worker := agnt5.NewWorker("<template-name>", agnt5.WithMaxConcurrency(16))

    model := agnt5.NewOpenAIModel(agnt5.OpenAIConfig{
        Model:  "gpt-4o-mini",
        APIKey: os.Getenv("OPENAI_API_KEY"), // not read from the environment automatically
    })
    agent, err := agnt5.NewAgent("agent_name",
        agnt5.WithAgentModel(model),
        agnt5.WithAgentInstructions("You are <AgentName>, <one-line role>..."),
    )
    if err != nil {
        log.Fatal(err)
    }

    err = agnt5.RegisterFunction(worker, "my_function", func(ctx *agnt5.Context, in MyInput) (MyOutput, error) {
        ctx.Logger().Info("running function", "message", in.Message)
        res, err := agent.Run(ctx, agnt5.AgentInput{Message: in.Message})
        if err != nil {
            return MyOutput{}, err
        }
        return MyOutput{Result: res.Response}, nil
    })
    if err != nil {
        log.Fatal(err)
    }

    err = agnt5.RegisterWorkflow(worker, "my_workflow", func(ctx *agnt5.Context, in MyInput) (MyOutput, error) {
        // agnt5.Step checkpoints the result, so a restart skips completed steps.
        result, err := agnt5.Step(ctx, "process", func(context.Context) (string, error) {
            return in.Message, nil
        })
        if err != nil {
            return MyOutput{}, err
        }
        return MyOutput{Result: result}, nil
    })
    if err != nil {
        log.Fatal(err)
    }

    if err := worker.Run(context.Background()); err != nil {
        log.Fatal(err)
    }
}
```

Other model constructors: `NewAnthropicModel(AnthropicConfig{...})`,
`NewGoogleModel` / `NewGeminiModel(GoogleConfig{...})`, `NewAzureOpenAIModel`,
`NewOpenRouterModel`, `NewGroqModel`, `NewDeepSeekModel` (the last three take `OpenAIConfig`).
To expose an agent directly as a component: `agnt5.RegisterAgent(worker, agent)`.

## Write order

`main.go` → `go.mod` (then `go mod tidy`) → `agnt5.yaml`, `.env.example`, `README.md`.
