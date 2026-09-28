# skills

A collection of [Agent Skills](https://agentskills.io) for AI coding agents working with [AGNT5](https://agnt5.com).
Written against AGNT5 Python SDK **0.13.6** (`@agnt5/sdk` 0.10.5, `sdk-go` v0.10.3) and the September 2026 CLI.

## Install

```bash
npx skills add https://github.com/agnt5dev/skills --full-depth
```

To install all skills to Claude Code only, without prompts:

```bash
npx skills add https://github.com/agnt5dev/skills --full-depth -s '*' -a claude-code -y
```

> `--full-depth` is needed to pick up the [investigation skills](#investigate), which live in the
> `investigate/` subfolder. Without it, only the top-level skills are found.

You'll be prompted to select which skills to install, which agents to target, and the installation scope.

### Options

| Option | Description |
|--------|-------------|
| `-g, --global` | Install to user directory instead of project |
| `-a, --agent <agents...>` | Target specific agents (e.g., `claude-code`, `codex`) |
| `-s, --skill <skills...>` | Install specific skills by name (use `'*'` for all skills) |
| `-l, --list` | List available skills without installing |
| `--full-depth` | Search all subfolders for skills (needed for `investigate/`) |
| `--copy` | Copy files instead of symlinking to agent directories |
| `-y, --yes` | Skip all confirmation prompts |
| `--all` | Install all skills to all agents without prompts |

**Examples:**

```bash
# Install a specific skill to Claude Code only
npx skills add https://github.com/agnt5dev/skills -s agnt5-ai-templates -a claude-code

# Install the investigation skills globally to Claude Code
npx skills add https://github.com/agnt5dev/skills --full-depth \
  -s agnt5-run-investigation -s agnt5-pattern-analysis -a claude-code -g -y

# Install all skills globally without prompts
npx skills add https://github.com/agnt5dev/skills --full-depth --all -g

# List available skills without installing
npx skills add https://github.com/agnt5dev/skills --full-depth --list
```

## Available Skills

Skills are grouped by the AGNT5 lifecycle: **build** it, **run** it, **improve** it.

### Build

| Skill | Description |
|-------|-------------|
| `agnt5-project-init` | Install and authenticate the CLI, create or link a project, and run it locally (`uv sync`, `.env`, `agnt5 dev`, `agnt5 run`, troubleshooting). |
| `agnt5-ai-templates` | Generate a complete AGNT5 project from a description (Python, TypeScript, Go), or scaffold from a template. |
| `agnt5-workflows` | Functions (retries/backoff/timeouts) and durable workflows: keyed steps, parallel fan-out, durable sleep, cron, state, idempotency keys. |
| `agnt5-agents-tools` | Agents and their tools: custom/built-in/MCP tools, sandboxes, callbacks, memory, handoffs, agents-as-tools. |
| `agnt5-agent-skills` | Give an AGNT5 agent its own SKILL.md/AGENTS.md system at runtime — on-demand capabilities and standing project guidance. |
| `agnt5-human-in-the-loop` | Add durable human approval, input, or selection pauses to a workflow. |
| `agnt5-webhooks-integrations` | Webhook and event triggers (Stripe, GitHub, Sentry, Slack, Standard Webhooks), chat bots, and calling workflows from your app. |
| `agnt5-prompts` | Versioned, code-bundled Prompt artifacts, runtime model overrides, and prompt caching. |

### Run

| Skill | Description |
|-------|-------------|
| `agnt5-deploy` | Secrets and provider credentials, `agnt5 deploy`, verify, `agnt5 deployment promote`, roll back, and scale. |
| `agnt5-observe` | Look up runs, traces, logs, and metrics; control automatic OpenAI/Agents SDK/ADK call capture. |

### Improve

| Skill | Description |
|-------|-------------|
| `agnt5-scorers` | Pick built-in deterministic/LLM-as-judge scorers or write and deploy a custom `@scorer`. |
| `agnt5-experiments` | Curate and version eval datasets, run a component or prompt against them, compare results, and gate CI. |
| `agnt5-online-evals` | Sample and score production runs asynchronously, with alerting on quality drops. |
| `agnt5-quality-cases` | Track a regression or production issue to a verified fix; automate it with the self-improvement loop. |

### Investigate

These skills work on your live AGNT5 data through the AGNT5 MCP server. They are playbooks for
an agent analyzing runs, not guides for writing code. They need the AGNT5 MCP connected. See
[investigate/README.md](investigate/README.md) for the tools they use.

| Skill | Description |
|-------|-------------|
| `agnt5-run-investigation` | Find why one run failed, was slow, cost too much, or answered wrong: reads the trace, logs, and deployment, compares with a healthy run, and returns a root cause with quoted evidence and a hand-off prompt for a coding agent. |
| `agnt5-pattern-analysis` | Find recurring behaviors across a project's runs (failure modes, cost or latency regressions, problems in one cohort or deployment) and report each with frequency and trace evidence. |

**Install just these two skills** (globally, to Claude Code):

```bash
npx skills add https://github.com/agnt5dev/skills --full-depth \
  -s agnt5-run-investigation -s agnt5-pattern-analysis -a claude-code -g -y
```

- `--full-depth` is required, because both skills live in `investigate/`.
- Drop `-g` to install into the current project instead of your user directory.
- Replace `-a claude-code` with another agent (e.g. `codex`) or leave it off to be asked.
- Re-run the same command to update to the latest version, then start a new agent session.

## Usage

Once installed, invoke a skill in your agent by describing the task it handles. For example:

- "Create a new empty AGNT5 project" → uses `agnt5-project-init`
- "Create a new AGNT5 template for a document processing pipeline" → uses `agnt5-ai-templates`
- "Set up and run this AGNT5 worker locally" → uses `agnt5-project-init`
- "Why did run 01a0d57b… fail?" → uses `agnt5-run-investigation`
- "Find patterns in today's runs for project a5sre" → uses `agnt5-pattern-analysis`

> Review skills before use — they run with full agent permissions.
