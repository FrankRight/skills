# Go agent skills and AGENTS.md

Verified against `github.com/agnt5dev/sdk-go` **v0.10.3** (skills and AGENTS.md guidance
shipped in 0.6.0). The product docs page says the Go SDK has none of this — that is wrong;
use the API below.

## Concept map

| Python (SKILL.md) | Go |
|---|---|
| `skills_dir="./skills", skills=["a", "b"]` | `agnt5.WithAgentSkillsFromDir("./skills", "a", "b")` |
| `skills_dir="./skills"` (whole pool) | `agnt5.WithAgentSkillsFromDir("./skills")` |
| `skills=[Skill.from_path(p)]` | `s, err := agnt5.SkillFromPath(p)` then `agnt5.WithAgentSkills(s)` |
| `discover_skills(dir)` | `agnt5.DiscoverSkills(dir) (map[string]agnt5.Skill, error)` |
| `agents_md="./AGENTS.md"` / list | `agnt5.WithAgentGuidance("./AGENTS.md", "./research/AGENTS.md")` |
| `discover_agents_md(start, stop_at_git=True)` | `agnt5.DiscoverAgentsMD(start, true) ([]string, error)` |
| `sandbox=Sandbox()` | `agnt5.WithAgentSandbox(runner)` — see `agnt5-agents-tools` |
| `SkillLoaded` stream event | `skill.loaded` journal event (there is no `Agent.stream`) |

The skill folder format is identical (`SKILL.md` with `name`/`description` front matter, optional
bundled files). `agnt5.Skill{Name, Description, Instructions, ResourcesDir}`; `ResourcesDir` is
the skill folder.

## Curated pool

```go
researcher, err := agnt5.NewAgent("researcher",
    agnt5.WithAgentModel(model),
    agnt5.WithAgentInstructions("Help the user analyze documents."),
    agnt5.WithAgentSkillsFromDir("./skills", "pdf-extraction", "sql-reporting"),
    agnt5.WithAgentSandbox(sandbox), // optional: lets bundled scripts run
)

reporter, err := agnt5.NewAgent("reporter",
    agnt5.WithAgentModel(model),
    agnt5.WithAgentInstructions("Build reports from warehouse data."),
    agnt5.WithAgentSkillsFromDir("./skills", "sql-reporting"),
)

generalist, err := agnt5.NewAgent("generalist", agnt5.WithAgentModel(model),
    agnt5.WithAgentInstructions("Use whichever skill fits the task."),
    agnt5.WithAgentSkillsFromDir("./skills")) // no names => every valid skill in the pool

pdf, err := agnt5.SkillFromPath("./my-skills/pdf-extraction") // one skill, no shared pool
analyst, err := agnt5.NewAgent("analyst", agnt5.WithAgentModel(model),
    agnt5.WithAgentInstructions("Analyze documents."), agnt5.WithAgentSkills(pdf))
```

- A name that is not in the pool fails at `NewAgent` with
  `agnt5: skill "x" not found in ./skills; available: a, b` — typos surface at startup.
- `DiscoverSkills` ignores malformed folders so one bad skill does not disable the pool.
  `agnt5.ResolveSkills(nil, dir)` loads all, `ResolveSkills([]string{}, dir)` loads none.
- Inspect a pool: `for name, s := range must(agnt5.DiscoverSkills("./skills")) { fmt.Println(name, s.Description) }`.

## How loading works (same model as Python)

`NewAgent` resolves the skills, renders a `<skills>` catalog (names + descriptions only) into
the system prompt after any AGENTS.md guidance, and adds a `load_skill` tool automatically when
the agent has at least one skill. `load_skill("pdf-extraction")` returns the instructions;
an unknown name returns a message listing the available skills. With a sandbox attached, the
skill's bundled files are copied into the sandbox under `skills/<name>/...` and the reply lists
those paths; without a sandbox the instructions still load but scripts cannot run.

Each load emits a `skill.loaded` event with `skill_name`, `instructions_length`,
`resources_materialized`. Read it from the trace (`agnt5 inspect trace -r <runId>`) or
`client.GetEvents(ctx, runID)`.

Building your own loop instead of `Agent`: `agnt5.RenderSkillsCatalog(skills)`,
`agnt5.NewLoadSkillTool(skills, sandbox) (agnt5.Tool, error)`, `agnt5.LoadAgentsMD(sources...)`,
`agnt5.RenderProjectGuidance(text)`.

## AGENTS.md — always-on guidance

```go
agent, err := agnt5.NewAgent("researcher",
    agnt5.WithAgentModel(model),
    agnt5.WithAgentInstructions("Help the user analyze documents."),
    agnt5.WithAgentGuidance("./AGENTS.md", "./research"), // files or directories (dir => <dir>/AGENTS.md)
    agnt5.WithAgentSkillsFromDir("./skills", "pdf-extraction"),
)

paths, err := agnt5.DiscoverAgentsMD("./research", true) // walks UP from ./research, outermost first, stops at .git
agent, err = agnt5.NewAgent("researcher", agnt5.WithAgentModel(model),
    agnt5.WithAgentInstructions("..."), agnt5.WithAgentGuidance(paths...))
```

Sources compose in order, most general first / most specific last; missing sources are
ignored (so a wrong path is silent — log `paths` at startup). Guidance renders before the skills
catalog: standing rules first, then the on-demand capability list. `LoadAgentsMD(sources...)`
returns the composed text if you want to inspect it.

## Paths at runtime

The helpers read the filesystem; relative paths resolve against the worker's working
directory. Under `agnt5 dev` that is the project root (`go run .`). Ship `skills/` and
`AGENTS.md` in the repo — they are not compiled into the binary. For managed deploys the
worker is built and started from the uploaded project directory; verify with an `os.Stat` at
startup. A missing skills directory fails `NewAgent` (`agnt5: read skills directory ...`), but
an existing empty one loads zero skills silently, and a missing `WithAgentGuidance` source is
ignored without error — log what resolved.

## Not available in Go

`Agent.stream` / `SkillLoaded` class, `skill.loaded` via a stream callback, Python's
`skills=` accepting names without `skills_dir` (Go: `WithAgentSkillsFromDir(dir, names...)`),
`agents_md=` accepting a list type (Go: variadic strings).

## Go pitfalls for this skill

- Skills need a `LanguageModel` that handles tool calls: the catalog is useless if the model
  cannot call `load_skill`. Reasoning models over Chat Completions fail on tool use in Go
  (AGNT5-1325) — use `gpt-4o-mini`/`gpt-4.1-mini`.
- Bundled scripts only run with `WithAgentSandbox`; `NewInMemorySandbox()` stores the files but
  executes nothing, so use it only in tests.
- `WithAgentGuidance` silently skips missing files; `WithAgentSkillsFromDir` errors on a
  missing directory or an unknown name, but an empty directory loads nothing without error —
  check the count at startup.
