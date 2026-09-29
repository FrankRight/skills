# Agent skills and AGENTS.md in TypeScript

Verified against `@agnt5/sdk` **0.10.5**. The skill format (`SKILL.md` folders) is identical;
only the agent options and helper names differ. Same section order as the Python SKILL.md.

## Write a skill

Unchanged: a folder with `SKILL.md` (`name` + `description` front matter, markdown body) and
optional bundled files. Keep `skills/` inside the project directory so it ships with the
worker; paths below are resolved from the worker's working directory (`npx tsx app.ts` runs
from the project root).

## Give an agent a curated skill pool

```typescript
import { Agent, LM, Sandbox } from '@agnt5/sdk';

const researcher = new Agent({
  name: 'researcher',
  model: LM.openai(),
  modelName: 'openai/gpt-4o-mini',
  instructions: 'Help the user analyze documents.',
  skillsDir: './skills',
  skills: ['pdf-extraction', 'sql-reporting'],
  sandbox: new Sandbox({ provider: 'e2b' }),
});

const reporter = new Agent({
  name: 'reporter',
  model: LM.openai(),
  modelName: 'openai/gpt-4o-mini',
  instructions: 'Build reports from warehouse data.',
  skillsDir: './skills',
  skills: ['sql-reporting'],
});
```

| Python | TypeScript `AgentOptions` |
|---|---|
| `skills_dir="./skills"` | `skillsDir: './skills'` |
| `skills=["a", "b"]` | `skills: ['a', 'b']` (`SkillInput[]` = names or `Skill` objects) |
| omit `skills=` | omit `skills` — loads every skill in `skillsDir` |
| `Skill.from_path(p)` | `Skill.fromPath(p)` (folder or the `SKILL.md` file itself) |
| `discover_skills(dir)` | `discoverSkills(dir)` → `Map<string, Skill>` |

A name missing from the pool throws at construction with the available names. A name given
without `skillsDir` also throws.

```typescript
import { Agent, LM, Skill, discoverSkills } from '@agnt5/sdk';

const analyst = new Agent({ ..., skills: [Skill.fromPath('./my-skills/pdf-extraction')] });

for (const [name, skill] of discoverSkills('./skills')) {
  console.log(`${name}: ${skill.description}`);           // Skill { name, description, instructions, resourcesDir? }
}
```

## How loading works

Same mechanism: a `<skills>` catalog (names + descriptions) is appended to the system prompt
and a built-in `load_skill` tool is added. With a `sandbox`, loading copies the skill's folder
into the workspace under `skills/<name>/`; without one, only the instructions come back.

Each load emits a `skill.loaded` event: `{ eventType: 'skill.loaded', skillName,
instructionsLength, resourcesMaterialized }`. There is no `SkillLoaded` class to
`instanceof`; narrow on `eventType`, and remember `agent.stream()` also yields the final
`AgentResult` (no `eventType`):

```typescript
for await (const item of agent.stream('Analyze this PDF')) {
  if ('eventType' in item && item.eventType === 'skill.loaded') {
    console.log(`Loaded: ${item.skillName} (${item.instructionsLength} chars, ${item.resourcesMaterialized} files)`);
  }
}
```

## AGENTS.md — always-on guidance

```typescript
import { Agent, LM, discoverAgentsMd, loadAgentsMd } from '@agnt5/sdk';

const agent = new Agent({
  ...,
  agentsMd: './AGENTS.md',                                    // file, or a directory containing AGENTS.md
  skillsDir: './skills',
  skills: ['pdf-extraction'],
});

new Agent({ ..., agentsMd: ['./AGENTS.md', './research/AGENTS.md'] });   // general first, specific last
new Agent({ ..., agentsMd: discoverAgentsMd('./research') });            // walks UP to the .git root
const text = loadAgentsMd(['./AGENTS.md', './research/AGENTS.md']);      // concatenated string, '' if none
```

`discoverAgentsMd(startDir, stopAtGit = true)` returns outermost-first paths; start it at the
most specific directory. Missing files are skipped silently. Guidance is injected before the
skills catalog on every prompt.

## When to use which

Identical to the Python table. Both options are optional; an agent without them is unchanged.

## Not available in TypeScript

- `SkillLoaded` / event classes for `instanceof` checks (events are plain objects)
- Nothing else is missing: `skills`, `skillsDir`, `agentsMd`, `Skill.fromPath`,
  `discoverSkills`, `discoverAgentsMd`, `loadAgentsMd` cover the Python surface

## TypeScript pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| `Error: skill "x" not found` at startup | typo, or `skills` given without `skillsDir` | fix the name / add `skillsDir` |
| Skill works locally, not in the deployed worker | `skills/` outside the project dir or wrong cwd | keep it under the project root, use `./skills` |
| Bundled script never runs | no `sandbox` on the agent | add `sandbox: new Sandbox({ provider })` |
| `item.eventType` is `undefined` on the last stream item | that item is the `AgentResult` | check `'eventType' in item` first |
| Agent not callable from Studio | agents are not auto-registered | `worker.registerAgents([agent])` |

## Source

https://agnt5.com/docs/build/skills (TypeScript tabs)
