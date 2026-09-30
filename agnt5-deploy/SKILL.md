---
name: agnt5-deploy
description: Ship an AGNT5 worker to managed infrastructure - set secrets and AI provider credentials, agnt5 deploy to preview/staging/production (including CI flags), the agnt5.yaml schema and what the code bundle includes (ignore files, prompts/ and skills/, Python 3.14 image), verify with deployment status/errors/logs and deploy debug, promote a verified build to production with agnt5 deployment promote, roll back, and scale replicas. Use for "deploy this", "promote to production", "roll back", "set the OpenAI key for production", "why did my deploy fail", "what goes in agnt5.yaml", or environment-scoped configuration.
---

# AGNT5 Deploy

> **TypeScript or Go?** The commands here apply to every language; the language-specific parts (setup, packaging, runtime behaviour) are in [references/typescript.md](references/typescript.md) and [references/go.md](references/go.md).

Deploying moves your worker off your laptop onto AGNT5's managed infrastructure. Prereq:
`agnt5 auth login`.

## Secrets and AI provider credentials (before deploying)

```bash
agnt5 secrets set --name OPENAI_API_KEY --type api_key            # prompted securely
echo "sk-..." | agnt5 secrets set --name OPENAI_API_KEY --type api_key --stdin
agnt5 secrets set --name OPENAI_API_KEY --type api_key --environment <environment-id>  # env-only override
agnt5 secrets list [--environment <environment-id>]
```

Run inside the project directory (or pass `--project`). An environment-scoped secret
overrides the project-scoped one of the same name for deployments serving that environment.
`--environment` takes the environment **ID**, not its name: `agnt5 deployment list -o json`
shows it as `env_id` on each deployment.

Workers read secrets as environment variables when they start. Setting or changing a secret
does not reach workers that are already running: they keep the old value until you deploy
again (any new deployment starts new workers, which read the current values).

Or use **Studio → Settings → Integrations**: add the provider at the narrowest scope
(workspace / project / environment). First-class Studio providers: `openai`, `anthropic`,
`google`/`gemini`, `groq`, `openrouter`, `mistral`, `deepseek`, `xai`; SDK-only (set as
secrets/env): `azure`, `bedrock`, `ollama`, `huggingface`. Code never sees the raw key —
AGNT5 injects it at runtime. Inbound webhook sources are set up there too (see
`agnt5-webhooks-integrations`).

## Deploy

```bash
agnt5 deploy                     # preview (default environment)
agnt5 deploy --env staging --min-replicas 1 --max-replicas 4
agnt5 deploy --dry-run           # validate config, show what would deploy
agnt5 deploy --env production --interactive=false --no-wait   # CI: no prompts, don't block
```

Other useful flags: `--replicas`, `--wait-timeout 10m`, `--max-run-duration 1h|forever`,
`--base-image ghcr.io/agnt5dev/python-worker:3.14`, `--skip-validation`, `--workspace`.
Full list: `agnt5 deploy --help`.

Output includes a Studio deployment URL, the deployment ID, and the `agnt5 logs
<deployment-id>` command. `agnt5 deploy` targets `--env` (default `preview`); the
`environment:` key in `agnt5.yaml` does not change that. **Usual flow: deploy to preview →
verify → promote the same code forward** — don't rebuild per environment.

## Verify / debug a deployment

```bash
agnt5 deployment list [--status failed] [--limit 50]
agnt5 deployment status --watch        # replicas, uptime of the latest deployment
agnt5 deployment errors --since 1h     # scheduling failures, image pull errors
agnt5 deploy debug <deployment-id> --logs   # timeline, pod status, crash output
agnt5 logs <deployment-id> --follow    # the platform's lifecycle log for the deployment
```

`agnt5 logs <deployment-id>` (and Studio's deployment logs, and MCP `get_deployment_logs`)
show what the platform did — bundle, scheduling, readiness, traffic switch. They do not show
your worker's own stdout/stderr (`print`, `console.log`, Go `log.Printf`). When a worker
crashes, `agnt5 deploy debug <deployment-id> --logs` shows the exit code and the last line it
printed under **Pod Status**. For output from a healthy worker, log through the SDK's run
logger and read the run's logs (`agnt5-observe`).

Smoke-test a deployment by ID with the same `agnt5 run` command as local dev (flags in
`agnt5-project-init`):

```bash
agnt5 run my_workflow --type workflow --input '{"message": "..."}' --deployment-id <deployment-id>
```

`--env <name>` runs against the deployment that environment routes to, and fails with
"environment <name> has no active deployment" when it has none. This needs CLI
`20260930-a31e8d` or later (`agnt5 version update`): older CLIs ignored `--env`, so
`--env preview` ran on production. With a service key in `AGNT5_API_KEY`, `--env` is not
applied and the key's own environment is used.

## Environments — promote, don't rebuild

An **environment** (preview/staging/production by default) serves traffic from its live
deployment.

```bash
agnt5 deployment promote --latest --env staging
agnt5 deployment promote <deployment-id> --env production        # asks for confirmation
agnt5 deployment promote --latest --env production --yes          # CI
```

Promotion creates a **new** deployment in the target environment from the same code bundle
or image: it gets a new deployment ID and new workers (Go workers build again when they
start). The environment keeps serving its current deployment until the new one is ready,
then traffic switches and the old one drains. Nothing is rebuilt by the CLI, so the code you
verified is the code that runs, but the new workers read the current secrets.

**Rollback** (no CLI command yet) — Studio → Deployments → environment tab → Rollback, the
MCP tool `rollback_deployment` (`environment_id`, optional `deployment_id`, `confirm: true`),
or `POST https://api.agnt5.com/api/v1/deployments/rollback` with
`{"environment_id": "...", "deployment_id": "..."}` (omit `deployment_id` for the one
before). Like promotion, it creates a new deployment from the earlier deployment's code.

> Rollback changes which code serves traffic, not your data or secrets — if the bad deploy
> also changed a secret or external state, revert those separately.

## Scale / stop / resume (Studio, MCP, or API)

Scale: Studio Scale action, or `POST https://api.agnt5.com/api/v1/deployments/<id>/scale-up`
(`scale-down`). Stop: Studio Terminate (image/record persist). Resume: Studio Start.

Control-plane REST calls (`https://api.agnt5.com/api/v1/...`) need a **personal API key**
(Studio → Settings → Profile → API keys) sent as `X-API-KEY`; service keys are rejected
there with 401.

From an MCP client, register the CLI's built-in server — it uses your `agnt5 auth login`
session (Claude Code: `claude mcp add agnt5 -- agnt5 mcp`) — then use `scale_deployment`,
`rollback_deployment`, `terminate_deployment`, `start_deployment`, `promote_deployment`,
`list_promotion_history`. Run it without `--services`: the promotion and rollback tools are in
no category, so any `--services` list hides them.

## `agnt5.yaml` and what gets bundled

```yaml
name: my-project                  # project display name
language: python                  # python | typescript | go
language_version: "3.12"
environment: dev                  # recorded; agnt5 deploy targets --env (default preview)
worker:
  command: "uv run python app.py" # what agnt5 dev and the deployed worker run (inferred when omitted); watch / healthCheck / env are dev-only
deploy:
  dockerfile: ./Dockerfile        # optional; only a Dockerfile in the project root switches to an image build
  ignore_file: .agnt5ignore       # code-bundle ignore file; default .agnt5ignore (a missing file adds nothing)
  base_image: ghcr.io/agnt5dev/python-worker:3.14   # code-bundle base image (same as --base-image)
  build_args: {KEY: value}
  registry: {url: ..., username: ...}   # password via env, never in the file
variables: {}                     # optional key/value map
```

Leave out `deploy.resources`: it is not applied, and `agnt5 deploy` warns "deploy.resources in
agnt5.yaml is not applied; worker CPU/memory are managed by the platform". The blank Python
scaffold and several templates still include it; delete it.

Two build paths. With a `Dockerfile` in the project root the CLI builds your image
(`.dockerignore` applies; `--force-code-bundle` overrides). Otherwise it uploads a **code
bundle** onto the base image, excluding `.git`, `.venv`, `node_modules`, `__pycache__`, build
output and similar, plus every pattern in `.agnt5ignore` and `.gitignore` (negations and a
bare `*` are ignored); `--force-dockerfile` forces the image path. Consequences: `.env` is
normally gitignored and never ships — secrets come from `agnt5 secrets set`; `prompts/` and
`skills/` must **not** be gitignored, because the worker resolves them against its working
directory at runtime; the Python image runs Python 3.14.

## Calling the deployed worker from your app

Use `agnt5.Client` (see `agnt5-client`) or copy the prefilled curl from the component page
in Studio. Create a service key (sent to `https://gw.agnt5.com` as `X-API-KEY`):

```bash
agnt5 service-keys create --name <name> --project <project-id> [--environment <environment-id>] [--scopes run,workflow] [--expires-in 30d]
agnt5 service-keys list --project <project-id>
agnt5 service-keys revoke <key-id>
```

`--environment` takes the environment ID (`env_id` in `agnt5 deployment list -o json`), not a
name. Scope `run` (the default) starts runs and reads them; resuming or cancelling a run
needs `workflow` as well, otherwise the gateway answers 403 `INSUFFICIENT_SCOPES`.

## Source

https://agnt5.com/docs/run/deploying · https://agnt5.com/docs/run/environments
