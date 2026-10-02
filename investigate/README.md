# Investigation skills

Skills for an agent that **investigates a user's AGNT5 data** through the AGNT5 MCP server.
These are different from the top-level `agnt5-*` skills, which teach a coding agent how to
build with AGNT5 (CLI, SDK, Studio). These teach an analyst agent how to reason over runs,
their events and logs, and deployments — what to look at, in what order, and what counts as
evidence.

| Skill | Use when |
|---|---|
| [agnt5-run-investigation](agnt5-run-investigation/SKILL.md) | One run failed, was slow, expensive, or produced a wrong answer |
| [agnt5-pattern-analysis](agnt5-pattern-analysis/SKILL.md) | Find recurring behaviors across many runs in a project |

## Connect the tools

The AGNT5 CLI ships the MCP server. Sign in once (`agnt5 auth login`), then register
`agnt5 mcp` (stdio) with your MCP client; it uses the CLI's login. For Claude Code:

```bash
claude mcp add agnt5 -- agnt5 mcp
```

Use CLI `20261002-b0b8c8` or later (`agnt5 version update`); older CLIs have no
`get_run_events`. Register it without `--services`: `get_run_events` and
`get_run_input_output` are in no category, so any `--services` list hides them.

The same tools are on the hosted server, `https://mcp.agnt5.com/mcp`: sign in with OAuth, or
send a personal API key in `X-API-KEY` exactly as Studio shows it. Each skill also maps its
tools to `agnt5` CLI commands for when MCP is not available.

## MCP tools used

Both skills are written against tools that exist today:

- Runs: `list_runs` (it also returns queued, running and paused runs, on its first page),
  `get_run_summary`, `get_run_events` (a run's steps, LLM and tool calls, attempts and errors,
  in order), `get_run_logs`, `get_run_input_output`
- Deployments: `list_deployments`, `get_deployment_events`, `get_deployment_logs`
- Projects: `list_projects`

The MCP server has no trace or analytics tools. The skills use `agnt5 inspect trace -r` or
Studio for a span tree, and Studio → **Analytics** and **Metrics** for project-wide numbers.

Online-eval verdicts are not listed by these tools: `list_scores` does not return them, and
`get_online_eval_result` needs the result's `decision_id`. The skills read them per run from
Studio's **Online evals** tab or the API in `agnt5-online-evals`.

## Planned tools (no SQL)

Pattern analysis is limited today to the fixed `list_runs` filters (component, status,
deployment, time). These typed tools would make it sharper; update the skill when they land:

| Tool | Replaces in the skill |
|---|---|
| Shared **filter object** (metadata.*, model, tool_called, error_type, duration/cost ranges, score, text) | Manual enumeration of runs |
| `describe_run_attributes(filter)` — keys and top values | Guessing cohort fields from sampled runs |
| `aggregate_runs(filter, group_by, metrics)` | Counting `list_runs` pages in triage |
| `count_runs(filter)` | "Estimated from sample" frequency → exact frequency |
| `list_runs(filter, sample=random\|stratified)` | Manual sub-window sampling |
| Batch `get_run_events(run_ids)` | One `get_run_events` call per run |
