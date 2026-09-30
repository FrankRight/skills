# Go deploy notes

Verified against `github.com/agnt5dev/sdk-go` **v0.10.3** and managed deploys on 29 Sep 2026.
The CLI flow in the SKILL.md (`agnt5 deploy`, promote, rollback, scale) is identical; this file
covers what differs for a Go worker.

## Before `agnt5 deploy`

Managed Go workers are **built at pod start**: the image runs `go mod download` and
`go build ./...` on your uploaded source with **go1.26.8 linux/amd64**. Consequences:

1. Run the same locally first, or the deploy fails after a long wait:

   ```bash
   go mod tidy && go build ./... && go vet ./...
   ```

   A stale `go.sum` (missing entry after an import change) is the usual failure. Commit
   `go.sum`.
2. `go.mod`'s `go` directive must be ≤ `1.26.8` (the templates use `go 1.26.5`). Toolchain
   auto-download does not happen in the worker image.
3. Vendoring is unnecessary; all modules resolve through the proxy at build time (network
   inside the build is required — not verified for private modules; assume they need
   `GOPRIVATE` + credentials as secrets).
4. Every new deployment — `agnt5 deploy`, a promotion, a rollback — plus eval-worker starts
   and cold replicas pay the build time (a promotion creates a new deployment with new
   workers). Check `agnt5 deployment status --watch` until Ready before `agnt5 run`,
   experiments or webhooks tests; "connection refused"/no-worker errors during that window
   are not bugs.

`agnt5.yaml` (`language: go`, `worker.command: "go run ."`) is what the platform executes after
building. Keep `main.go` at the module root so `go run .` works.

## Secrets and provider keys

The Go SDK reads **no** provider credentials on its own — every model constructor takes an
explicit `APIKey` (see `agnt5-ai-templates/references/go.md`). So for a Go worker:

```bash
agnt5 secrets set --name OPENAI_API_KEY --type api_key                    # prompted
echo "sk-..." | agnt5 secrets set --name OPENAI_API_KEY --type api_key --stdin
agnt5 secrets set --name OPENAI_API_KEY --type api_key --environment <environment-id>
```

The secret name must equal the env var your code reads (`os.Getenv("OPENAI_API_KEY")`).
Workers read secrets when they start, so after setting or changing one, deploy again —
running workers keep the old value.
Studio **Settings → Integrations** provider credentials rely on the SDK's provider
auto-resolution, which Go does not have; whether they are also exported as env vars to a Go
worker was not verified — use `agnt5 secrets set` for Go. The built-in LLM judge scorers
(`agnt5-scorers`) run inside the worker and read `OPENAI_API_KEY`/`ANTHROPIC_API_KEY`/
`GOOGLE_API_KEY` the same way, so set the judge's key too.

Validate at startup and fail fast (the `code_reviewer` template's `LoadConfig().Validate()`
pattern) so a missing secret shows up in `agnt5 deploy debug <id> --logs`, not as a 401 on the
first run.

## Runtime environment

The platform injects `AGNT5_COORDINATOR_ENDPOINT`, `AGNT5_ENGINE_URL`, `AGNT5_PROJECT_ID`,
`AGNT5_DEPLOYMENT_ID`, `AGNT5_WORKER_ID` and friends; `agnt5.NewWorker` reads them. Defaults
worth knowing: pull dispatch (`AGNT5_WORKER_MODE=pull`, 0.8.0+), concurrency from
`AGNT5_MAX_CONCURRENCY` or `agnt5.WithMaxConcurrency(n)`. `--max-run-duration` on
`agnt5 deploy` bounds runs; honour `ctx.Done()` in long loops regardless (how the platform
limit surfaces inside a Go handler was not verified).

The platform also sets `AGNT5_ACTIVATION_ARTIFACT_SHA256` for deployed workers, so the
"durable activation degraded" warning you see under `agnt5 dev`
(`agnt5-project-init/references/go.md`) does not apply to them.

Logs from a deployed Go worker:

| You write | Where you can read it |
|---|---|
| `ctx.Logger().Info(...)` | the run's logs: MCP `get_run_logs`, or the run in Studio (`agnt5 inspect logs -r` currently returns 403) |
| `slog.InfoContext(ctx, ...)` with `agnt5.NewSlogHandler` installed | the run's logs, same as above |
| `log.Printf`, `fmt.Println`, `slog.Info` without a context | nowhere after deploy — not in the run's logs, `agnt5 logs <deployment-id>` (the platform's lifecycle log) or `agnt5 deploy debug --logs` while the worker is healthy. Locally they print in the `agnt5 dev` terminal. |
| anything printed just before a crash | `agnt5 deploy debug <deployment-id> --logs`, Pod Status → last output line |

So log anything you need after deploy through `ctx.Logger()` or `NewSlogHandler`. OTLP export
and core metrics: `agnt5-observe/references/go.md`.

## Verify a Go deployment

```bash
agnt5 deployment status --watch                 # wait for Ready (build time!)
agnt5 deploy debug <deployment-id> --logs       # build errors and crash output appear here
agnt5 run my_workflow --type workflow --input '{"message": "..."}' --deployment-id <deployment-id>
```

Build failures to recognise in the debug log: `missing go.sum entry` (tidy and redeploy),
`go.mod requires go >= 1.27` (lower the directive), `package ... is not in std` (module path vs
import path mismatch), compile errors that `go build ./...` would have caught locally.

## Calling the deployed worker

`agnt5.NewClient("", agnt5.WithAPIKey(key))` with a service key from
`agnt5 service-keys create --name <name> --project <project-id> [--environment <environment-id>]`
(an environment ID — `env_id` in `agnt5 deployment list -o json` — not a name);
`agnt5.WithClientDeploymentID(id)` pins a deployment. Details: `agnt5-client`.

## Not available in Go

Python base-image selection (`--base-image ghcr.io/agnt5dev/python-worker:...` — no Go
equivalent verified), SDK-side provider auto-resolution of Studio integrations, auto-capture
of third-party SDK calls.

## Go pitfalls for this skill

- Deploying with an untidy `go.sum` or a `go` directive above 1.26.8 fails only after the pod
  starts building — run `go mod tidy && go build ./...` first, every time.
- Running `agnt5 run` or an experiment against a deployment before it is Ready looks like a
  broken worker; it is the build.
- Diagnostics written with `log.Printf` or `fmt` are invisible once deployed; use
  `ctx.Logger()`.
- Bare model names and explicit `APIKey` are required (`"openai/gpt-4o-mini"` is a 400 in
  production exactly as locally).
- Reasoning models: Go agents with tools fail on gpt-6 over Chat Completions;
  stay on `gpt-4o-mini`/`gpt-4.1-mini` for deployed agents for now.
