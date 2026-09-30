# Investigation skills

Skills for an agent that **investigates a user's AGNT5 data** through the AGNT5 MCP server.
These are different from the top-level `agnt5-*` skills, which teach a coding agent how to
build with AGNT5 (CLI, SDK, Studio). These teach an analyst agent how to reason over runs,
traces, logs, and metrics — what to look at, in what order, and what counts as evidence.

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

`agnt5 mcp --services projects,runs,traces,analytics,deployments` exposes just the categories
these skills use. Each skill also maps its tools to `agnt5` CLI commands for when MCP is not
available.

## MCP tools used

Both skills are written against tools that exist today:

- Runs: `list_runs` and `get_run_summary` (finished runs only; a run that is still queued,
  running or paused has no summary yet), `get_run_logs`
- Traces: `get_trace_excerpt`, `get_trace`, `list_traces` (e.g. `min_duration_ms` for slow runs)
- Analytics: `get_analytics_dashboard`, `get_component_breakdown`, `get_error_breakdown`,
  `get_llm_usage`, `get_runs_timeseries`, `get_latency_timeseries`
- Deployments: `list_deployments`, `get_deployment_events`
- Projects: `list_projects`

Online-eval verdicts are not available through these tools (`list_scores` does not return
them); the skills read them per run from Studio's **Online evals** tab or the API in
`agnt5-online-evals`.

## Planned tools (no SQL)

Pattern analysis is limited today to the fixed filters above (component, status, deployment,
time). These typed tools would make it sharper; update the skill when they land:

| Tool | Replaces in the skill |
|---|---|
| Shared **filter object** (metadata.*, model, tool_called, error_type, duration/cost ranges, score, text) | Manual enumeration of runs |
| `describe_run_attributes(filter)` — keys and top values | Guessing cohort fields from sampled traces |
| `aggregate_runs(filter, group_by, metrics)` | Fixed breakdowns in triage |
| `count_runs(filter)` | "Estimated from sample" frequency → exact frequency |
| `list_runs(filter, sample=random\|stratified)` | Manual sub-window sampling |
| `get_traces_batch(ids)` | One `get_trace_excerpt` call per run |
