---
name: agnt5-agent-skills
description: Give AGNT5 agents on-demand capabilities via SKILL.md folders (Agent skills=/skills_dir=, load_skill tool, skill.loaded events) and always-on project guidance via AGENTS.md (agents_md=, discover_agents_md), instead of stuffing everything into one large prompt. Use when the user wants an AGNT5 agent to load specialized instructions only when relevant, share a skills pool across multiple agents, bundle scripts with a skill, or apply standing conventions with AGENTS.md.
---

# AGNT5 Agent Skills

A **skill** is a folder with a `SKILL.md` file. Only `name`/`description` stay in the agent's
context; the body loads on demand when the agent decides the task matches.

## Write a skill

```text
skills/
  pdf-extraction/
    SKILL.md
    scripts/
      extract.py
```

```markdown
---
name: pdf-extraction
description: Extract tables and text from PDF files
---

# PDF extraction

1. Write the PDF bytes to `input.pdf` in the sandbox.
2. Run `scripts/extract.py input.pdf` to produce `output.json`.
3. Read `output.json` and summarize the tables it contains.
```

Plain markdown — skills written for other agent frameworks work unchanged. The `skills/`
folder ships in the same deployment bundle as your worker code.

## Give an agent a curated skill pool

```python
from agnt5 import Agent, Sandbox

researcher = Agent(
    name="researcher", model="openai/gpt-4o-mini",
    instructions="Help the user analyze documents.",
    skills_dir="./skills", skills=["pdf-extraction", "sql-reporting"],
    sandbox=Sandbox(),
)

reporter = Agent(
    name="reporter", model="openai/gpt-4o-mini",
    instructions="Build reports from warehouse data.",
    skills_dir="./skills", skills=["sql-reporting"],
)
```

Two agents can draw different subsets from the same pool — each sees only the names you
list, so its catalog/context cost stays small. A name not in the pool fails at construction
with the list of available skills (typos surface immediately, not at runtime).

**Load every skill in the pool**: omit `skills=` entirely.

```python
agent = Agent(name="generalist", model="openai/gpt-4o-mini",
              instructions="Use whichever skill fits the task.", skills_dir="./skills")
```

**Load one skill by path, no shared pool**:

```python
from agnt5 import Agent, Skill
agent = Agent(name="analyst", model="openai/gpt-4o-mini", instructions="Analyze documents.",
              skills=[Skill.from_path("./my-skills/pdf-extraction")])
```

**Inspect a pool without building an agent**:

```python
from agnt5 import discover_skills
pool = discover_skills("./skills")
for name, skill in pool.items():
    print(f"{name}: {skill.description}")
```

## How loading works

The SDK adds a `<skills>` catalog (names + descriptions only) to the system prompt and
registers a built-in `load_skill` tool. When a task matches, the agent calls
`load_skill("pdf-extraction")` and gets the full instructions back — only loaded skills
consume context, everything else stays out of the prompt.

If a **sandbox** is attached, loading a skill also copies its bundled files into the sandbox
workspace under `skills/<name>/`, so the agent can run them with sandbox tools with no extra
setup. Without a sandbox, `load_skill` still returns instructions, but bundled scripts can't
run.

Each load emits a `skill.loaded` event (`skill_name`, `instructions_length`,
`resources_materialized` = bundled files copied into the sandbox):

```python
from agnt5 import SkillLoaded
async for event in agent.stream("Analyze this PDF"):
    if isinstance(event, SkillLoaded):
        print(f"Loaded: {event.skill_name} ({event.instructions_length} chars)")
```

## AGENTS.md — always-on guidance

Use `AGENTS.md` for conventions that apply to *every* task (coding style, project structure);
use skills for capabilities the agent reaches for only sometimes.

```python
agent = Agent(
    name="researcher", model="openai/gpt-4o-mini",
    instructions="Help the user analyze documents.",
    agents_md="./AGENTS.md", skills_dir="./skills", skills=["pdf-extraction"],
)
```

Pass a list to layer multiple files — read in order, **most general first, most specific
last**:

```python
agent = Agent(..., agents_md=["./AGENTS.md", "./research/AGENTS.md"])
```

Collect every `AGENTS.md` from a directory **up** to the repo root:

```python
from agnt5 import Agent, discover_agents_md
agent = Agent(..., agents_md=discover_agents_md("./research"))  # root first, most specific last
```

`discover_agents_md` only walks **upward** from the start directory — start it at the most
specific directory, not the root, or subdirectory files are missed. It stops at the first
directory containing `.git` (`stop_at_git=True`); with no `.git` above it, it walks to the
filesystem root. Guidance loads before the skills catalog: standing rules first, then the
on-demand capability list.

## When to use which

| Use | For | Loaded |
|---|---|---|
| `AGENTS.md` | Standing rules for every task | Always, every prompt |
| Skills | Capabilities reached for occasionally | On demand, when a task matches |

Both optional — an agent with neither behaves exactly as before; they only change the prompt
when configured.

TypeScript uses camelCase names (`skillsDir`, `agentsMd`, `discoverSkills`, …) from `@agnt5/sdk`.

## Source

https://agnt5.com/docs/build/skills
