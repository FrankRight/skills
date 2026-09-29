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

Output includes a Studio deployment URL and the `agnt5 logs <deployment-id>` command.
**Usual flow: deploy to preview → verify → promote the same build forward** — don't
redeploy per environment.

## Verify / debug a deployment

```bash
agnt5 deployment list [--status failed] [--limit 50]
agnt5 deployment status --watch        # replicas, uptime of the latest deployment
agnt5 deployment errors --since 1h     # scheduling failures, image pull errors
agnt5 deploy debug <deployment-id> --logs   # timeline + diagnostics for a failed deploy
agnt5 logs <deployment-id> --follow
```

Smoke-test the deployed worker with the same `agnt5 run` command as local dev plus `--env`:
`agnt5 run my_workflow --type workflow --input '{"message": "..."}' --env production`
(flags in `agnt5-project-init`).

## Environments — promote, don't redeploy

An **environment** is a named pointer to a deployment (preview/staging/production by
default). The deployment image is immutable; promote/rollback just move the pointer, so the
exact build verified in staging is what serves production.

```bash
agnt5 deployment promote --latest --env staging
agnt5 deployment promote <deployment-id> --env production        # asks for confirmation
agnt5 deployment promote --latest --env production --yes          # CI
```

The environment keeps serving its current deployment until the promoted one is ready, then
traffic switches.

**Rollback** (no CLI command yet) — Studio → Deployments → environment tab → Rollback, or
`POST /api/v1/deployments/rollback`. It points the environment at the previously serving
deployment (a pointer move, not a rebuild).

> Rollback changes which code serves traffic, not your data or secrets — if the bad deploy
> also changed a secret or external state, revert those separately.

## Scale / stop / resume (Studio or API)

Scale: Studio Scale action, or `POST /api/v1/deployments/<id>/scale-up|scale-down`. Stop:
Studio Terminate (image/record persist). Resume: Studio Start. From Claude with the AGNT5 MCP
connected: `scale_deployment`, `rollback_deployment`, `terminate_deployment`,
`start_deployment`.

## `agnt5.yaml` and what gets bundled

```yaml
name: my-project                  # project display name
language: python                  # python | typescript | go
language_version: "3.12"
environment: dev                  # default target for agnt5 deploy
worker:
  command: "uv run python app.py" # used by agnt5 dev (inferred when omitted); watch / healthCheck / env are dev-only
deploy:
  dockerfile: ./Dockerfile        # optional; a Dockerfile in the project root switches to an image build
  ignore_file: .agnt5ignore       # recorded; default .agnt5ignore
  base_image: ghcr.io/agnt5dev/python-worker:3.14   # code-bundle base image (same as --base-image)
  build_args: {KEY: value}
  registry: {url: ..., username: ...}   # password via env, never in the file
  resources: {memory: 512Mi, cpu: 500m}
variables: {}                     # optional key/value map
```

Two build paths. With a `Dockerfile` in the project root the CLI builds your image
(`.dockerignore` applies; `--force-code-bundle` overrides). Otherwise it uploads a **code
bundle** onto the base image, excluding `.git`, `.venv`, `node_modules`, `__pycache__`, build
output and similar, plus every pattern in `.agnt5ignore` and `.gitignore` (negations and a
bare `*` are ignored); `--force-dockerfile` forces the image path. Consequences: `.env` is
normally gitignored and never ships — secrets come from `agnt5 secrets set`; `prompts/` and
`skills/` must **not** be gitignored, because the worker resolves them against its working
directory at runtime; the Python image runs Python 3.14.

## Calling the deployed worker from your app

Use `agnt5.Client` (see `agnt5-webhooks-integrations`) or copy the prefilled curl from the
component page in Studio. Create an API key with
`agnt5 service-keys create --name <name> --project <project-id> [--environment <env>]`.

## Source

https://agnt5.com/docs/run/deploying · https://agnt5.com/docs/run/environments
