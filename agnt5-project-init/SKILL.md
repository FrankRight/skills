---
name: agnt5-project-init
description: Set up and run an AGNT5 project locally - install, update, and authenticate the agnt5 CLI; create a brand-new empty project (agnt5 create / agnt5 init) or link an existing directory; install dependencies with uv, configure .env API keys, start agnt5 dev (hot reload, detached mode, dev status/logs/stop), trigger runs with agnt5 run or Studio, and troubleshoot a worker that won't start or connect. Use for "install the agnt5 CLI", "log in", "create a new blank AGNT5 project", "run this locally", "start the dev worker", "agnt5 dev fails", or "trigger my workflow" - not for generating agents/workflows from a description (use agnt5-ai-templates).
---

# AGNT5 Project Setup and Local Development

> **TypeScript or Go?** The commands here apply to every language; the language-specific parts (setup, packaging, runtime behaviour) are in [references/typescript.md](references/typescript.md) and [references/go.md](references/go.md).

Skip any step that is already done: `agnt5 auth status` shows a signed-in user → skip 0;
`agnt5.yaml` has project metadata → skip 1.

## 0. Install, update, and authenticate the CLI

Install is one-time, machine-wide. **Before every `agnt5 create`/`agnt5 init` run**, update
the CLI — a stale binary can carry bugs already fixed upstream (an outdated build has
mis-extracted templates when scaffolding into the current directory). `agnt5 version update`
(alias `agnt5 upgrade`) no-ops when already current.

```bash
curl -LsSf https://agnt5.com/cli.sh | bash   # macOS, Linux, WSL2 -- first-time install only
brew install agnt5/tap/agnt5                 # macOS only, alternative first-time install
agnt5 version update                         # run every time before create/init
agnt5 auth login                             # device-code sign-in; add --no-browser over SSH/containers
agnt5 auth status                            # confirms signed-in user + environment
```

CI / non-interactive: `agnt5 auth login --api-key agnt5_sk_...` or
`export AGNT5_API_KEY=agnt5_sk_...`.

**`command not found: agnt5`** — the installer writes to `~/.agnt5/bin` and appends it to
`PATH`; open a new terminal or reload the shell. If still missing:
`export PATH="$HOME/.agnt5/bin:$PATH"` (bash/zsh) or `fish_add_path "$HOME/.agnt5/bin"` (fish).
If `agnt5 version` still fails, the binary didn't download — re-run the installer and read
its output.

## 1. Create or link the project

**Starting fresh (no directory yet)** — `agnt5 create` makes a new directory, scaffolds a
blank starter project, and registers it on AGNT5:

```bash
agnt5 create my-project                        # new dir, blank starter, registered on AGNT5
agnt5 create my-project --workspace my-team    # pick the workspace up front
agnt5 create my-project --local                # skip registering on AGNT5 (fully offline)
```

**Already in a directory you want to become the project** — `agnt5 init` (alias `link`):

```bash
agnt5 init my-project                  # scaffold + link the current directory
agnt5 init                             # inspect current dir, offer to scaffold or link
agnt5 init --new --name my-project     # force: create a fresh empty project + link
agnt5 init --project <project-id>      # link to an existing project instead of creating one
```

What `init` does: empty directory → scaffolds a minimal starter, then links; existing code
without `agnt5.yaml` → adds a minimal one, then links; already an AGNT5 directory → links or
updates the link.

**Do not pass `--template <language>/<name>`** for a blank project — scaffolding from a
template or a description is `agnt5-ai-templates`' job.

| Flag | Applies to | Effect |
|---|---|---|
| `--local` | `create` | Scaffold locally without registering on AGNT5 |
| `--workspace <id\|name>` | `create`, `init` | Skip the workspace picker |
| `-y, --yes` | `create`, `init` | Never prompt; fail with an explanation if a choice needs asking |
| `--dry-run` | `create`, `init` | Show what would happen without executing |
| `--new` / `--name <name>` | `init` | Create a fresh empty project and link to it |
| `--project <id>` | `init` | Link to an existing project instead of scaffolding |
| `--language <lang>` | `create`, `init` | Scaffolding language (default `agnt5.yaml`'s, else `python`) |

**Non-interactive (agents, CI):** the CLI prompts for a workspace when the account has more
than one — pass `--workspace` and `-y`. Verify with `agnt5 info` (the linked project).

## 2. Install dependencies and configure `.env`

```bash
uv sync                  # creates .venv from pyproject.toml
cp .env.example .env     # then fill in real keys, e.g. OPENAI_API_KEY=sk-...
```

The uv warning `VIRTUAL_ENV does not match the project environment path` is harmless.
Required keys vary by template — check `.env.example`. Deployed workers get secrets
separately (`agnt5-deploy`). The managed Python worker image runs **Python 3.14**, so a
dependency that only resolves on your local 3.12 will break the deploy — prefer packages with
3.14 wheels.

## 3. Start the worker

```bash
agnt5 dev                # foreground, hot reload
agnt5 dev -d             # detached; then: agnt5 dev status | agnt5 dev logs | agnt5 dev stop
agnt5 dev -v             # verbose SDK/runtime logging
agnt5 dev --no-watch     # disable hot reload
```

A healthy start prints a banner, the worker command, the coordinator it connected to, the
registered components (workflows / agents / tools / functions), and a project-scoped **Studio
URL** (`https://app.agnt5.com/projects/<project-id>/components`) — open that exact link. The
banner text changes between CLI builds; what matters is `Connected to coordinator` and your
components listed. `agnt5 components` lists what registered.

Hot reload watches the project root, `src/`, and `src/<package>/` for `.py .ts .js .go .env`
changes, so `.env` edits need no restart. `Ctrl+C` stops the worker; the
`CancelledError`/`KeyboardInterrupt` traceback after it is expected.

## 4. Trigger a run

**Studio**: open the printed Studio URL, pick the component, set input JSON, **Run**. The run
shows a live trace tree and handles HITL pauses.

**CLI** (no `--env` ⇒ local dev worker):

```bash
agnt5 run hello_world --input '{"name": "Alice"}'                   # function (default type), streams
agnt5 run my_workflow --type workflow --input '{"message": "..."}'  # JSON on completion
agnt5 run workflow my_workflow -i '{"message": "..."}'              # same, positional type
agnt5 run my_tool --type tool --input '{"adults": 2}'
agnt5 run my_agent --type agent --input '{"message": "..."}'        # agent input needs "message"
```

`--type` defaults to `function` and auto-detects on a miss. Other flags: `--timeout 2m`
(client-side only; the run keeps going), `--env production` / `--deployment-id <id>` to hit a
deployed worker. There is no session/user flag — session-scoped runs come from `Client.run(...,
session_id=...)` (`agnt5-client`). Output is JSON when piped (`| jq`). A function with
`retries=` shows only its first failed attempt here while the platform keeps retrying. To
inspect what happened: `agnt5 inspect runs ls`, `agnt5 inspect trace -r <run-id>` (see
`agnt5-observe`).

## Common errors

| Error | Fix |
|---|---|
| `ModuleNotFoundError: No module named 'agnt5'` | `uv sync` |
| `... OPENAI_API_KEY must be set` / runs fail immediately | Key missing from `.env` |
| Worker won't register / "no project" errors | `agnt5 init` to link a project, re-run `agnt5 dev` |
| Auth errors | `agnt5 auth login` (`agnt5 auth logout` first if you switched accounts) |
| Wrong workspace / project not found | `agnt5 workspace list`, then `agnt5 workspace use <name>` |
| A component is missing | `agnt5 components`; check it is imported/registered in `app.py` (or `Worker(auto_register=True)`) |
| `TypeError: Function 'x' requires FunctionContext as first argument` | Inside a workflow call it through `ctx.step(x, ...)` (`agnt5-workflows`) |
| `ConfigurationError: Tool function 'x' first parameter must be 'ctx: Context'` | Annotate the first tool parameter exactly `ctx: Context` (`from agnt5.context import Context`) |
| `TypeError: got an unexpected keyword argument 'deployment_id'` on a triggered workflow | Declare it `async def h(ctx, event: dict, **_)` (`agnt5-webhooks-integrations`) |
| `400` mentioning `temperature` on `openai/gpt-6*` | `Agent(..., temperature=None)` / `lm.generate(temperature=None)` (`agnt5-sdk-pitfalls`) |
| Anything else | `agnt5 dev -v` and read the worker log, or `agnt5 dev logs` when detached |

After moving the project directory: `uv sync && agnt5 init && agnt5 dev` (choose "Link to
existing" in `init`).

## Source

https://agnt5.com/docs/install-cli · https://agnt5.com/docs/build/local-development
