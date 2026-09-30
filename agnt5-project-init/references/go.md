# Go project setup and local development

Verified against `github.com/agnt5dev/sdk-go` **v0.10.3** and the shipped Go templates. CLI
install/auth (step 0 of the SKILL.md) is language-independent.

## Toolchain

- `go version` ≥ the `go` directive in `go.mod`. The SDK's own `go.mod` says `go 1.26.5`; the
  managed worker image runs **go1.26.8 linux/amd64**, so keep your directive ≤ `1.26.8`
  (`go 1.26.5` as the templates do) or the deploy build fails.
- `go list -m -versions github.com/agnt5dev/sdk-go` shows released SDK versions.

## 1. Create or link (Go)

There is no blank Go scaffold: `agnt5 create --language go` and `agnt5 init --language go`
fail with "scaffolding for language … is not supported yet". Two paths work:

```bash
# A. Start from the quickstart template (agnt5-ai-templates covers the others)
agnt5 create my-project --template go/quickstart --local
# edit agnt5.yaml: name: my-project   (otherwise the project registers as "quickstart")
cd my-project && agnt5 init --new --name my-project --workspace <ws> -y

# B. Write main.go, go.mod, agnt5.yaml yourself (layout below), then link the directory
agnt5 init --new --name my-project --workspace <ws> -y
```

Layout the templates use (`agnt5-ai-templates/references/go.md` has the full version):

```
my-project/
├── main.go            # builds models/agents, registers every component, runs the worker
├── go.mod / go.sum    # module <my-project>
├── agnt5.yaml         # language: go, worker.command: "go run ."
├── .env.example
└── src/my_project/    # package my_project: agents.go, tools.go, functions.go, workflows.go
```

`agnt5.yaml`:

```yaml
name: my-project
language: go
language_version: "1.26"
environment: dev

worker:
  command: "go run ."
```

The Go templates also carry a `deploy.resources` block; delete it — it is not applied
(`agnt5-deploy`).

## 2. Dependencies and `.env`

```bash
go mod tidy              # fills go.sum; rerun after every import change
go build ./...           # compile check before starting the worker
cp .env.example .env     # OPENAI_API_KEY=sk-... etc.
```

`agnt5 dev` loads `.env` into the worker's environment. The SDK itself reads nothing from
`.env` — it only sees `os.Getenv` — so `go run .` started by hand without exporting the keys
gives empty API keys and provider 401s. Model constructors never read provider keys
implicitly; every template does `APIKey: os.Getenv("OPENAI_API_KEY")` explicitly.

## 3. Start the worker

```bash
agnt5 dev                # runs worker.command ("go run ."), hot reload on .go/.env changes
agnt5 dev -d             # detached; agnt5 dev status | logs | stop
agnt5 dev -v
```

`agnt5 dev` sets the connection env for the process (`AGNT5_COORDINATOR_ENDPOINT`,
`AGNT5_ENGINE_URL`, `AGNT5_PROJECT_ID`, `AGNT5_DEPLOYMENT_ID`, ...); `agnt5.NewWorker(name)`
picks them up — never hardcode them. Optional knobs: `AGNT5_MAX_CONCURRENCY` (or
`agnt5.WithMaxConcurrency(16)`), `AGNT5_WORKER_MODE` (`pull` default since 0.8.0; `push` to
opt back in). A healthy start logs `Connected to coordinator` and the registered components
(`agnt5 components --dev` lists them).

A local Go worker also prints:

```
[WARN] agnt5 durable activation degraded: activation artifact identity is unavailable; configure activation_artifact_sha256; legacy checkpoints remain enabled
```

This is expected under `agnt5 dev` and needs no action. The platform sets the deployment's
artifact identity (`AGNT5_ACTIVATION_ARTIFACT_SHA256`) only for deployed workers. Without it
the SDK falls back to its own step checkpoints, so `Step`/`Task` results are still reused on
replay. One visible difference: a local `ctx.Sleep` waits inside the worker process, so the
run shows `assigned` rather than `paused` while it sleeps.

What you see where, locally: `fmt`/`log` output and your own `slog` handler print in the
`agnt5 dev` terminal (`agnt5 dev logs` when detached). `ctx.Logger()` lines do not print
there; they go to the run's logs (MCP `get_run_logs`, Studio), see `agnt5-observe`.

Minimal `main.go`:

```go
func main() {
    model := agnt5.NewOpenAIModel(agnt5.OpenAIConfig{Model: "gpt-4o-mini", APIKey: os.Getenv("OPENAI_API_KEY")})
    if err := my_project.NewAgents(model); err != nil { log.Fatal(err) }

    worker := agnt5.NewWorker("my-project", agnt5.WithServiceVersion("1.0.0"))
    must(agnt5.RegisterAgent(worker, my_project.Assistant))                 // no auto_register in Go
    must(agnt5.RegisterWorkflow(worker, "my_workflow", my_project.MyWorkflow))
    if err := worker.Run(context.Background()); err != nil { log.Fatal(err) }
}
```

## 4. Trigger a run

Identical CLI: `agnt5 run my_workflow --type workflow --input '{"message": "..."}'`,
`agnt5 run my_agent --type agent --input '{"message": "..."}'` (agent input is
`agnt5.AgentInput{Message}`), Studio **Run**, `agnt5 inspect runs ls`, `agnt5 inspect trace -r`.

## Common Go errors

| Error | Fix |
|---|---|
| `missing go.sum entry for module providing package ...` | `go mod tidy` |
| `package my-project/src/my_project is not in std` / import errors | module name in `go.mod` must match the import prefix; hyphenated modules need an import alias (`my_project "my-project/src/my_project"`) |
| `go: updates to go.mod needed` / `go.mod requires go >= 1.26.5` | upgrade the local Go; keep the directive ≤ 1.26.8 for deploys |
| Provider `400 invalid model ID` | model name has a `provider/` prefix — Go takes bare names (`gpt-4o-mini`) |
| Provider `401` / `authentication` | key empty: `.env` not loaded (use `agnt5 dev`) or wrong `os.Getenv` name |
| `agnt5: agent model is required` | `NewAgent` without `WithAgentModel` |
| `agnt5: duplicate component` | same name registered twice (`RegisterTool` + a tool with the same name, or two agents) |
| Component missing in Studio | not registered in `main()`; Go has no import-side-effect registration |
| Worker exits immediately, no components | `worker.Run` returned an error you did not log; wrap it in `log.Fatal` |
| Tool runs with empty arguments | `NewTool` without `WithToolSchema` — the model sees no parameters |
| Anything else | `agnt5 dev -v`; read the worker output in the terminal (`agnt5 dev logs` when detached) |

Unit tests without a runtime: `agnt5.StaticModel{Content: "..."}` and
`agnt5.ScriptedModel{Responses: []agnt5.GenerateResponse{...}}` satisfy `LanguageModel`;
`agnt5.NewInMemorySandbox()`, `agnt5.NewInMemoryStateStore()` (`agnt5.WithStateStore`) — see
`agnt5-testing`.

## Not available in Go

`uv sync`/virtualenv steps, `Worker(auto_register=True)`, `.env` auto-loading by the SDK,
`agnt5 dev` Python traceback notes (Go workers stop on `Ctrl+C` via context cancellation).
