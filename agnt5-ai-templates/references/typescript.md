# TypeScript templates

Verified against `@agnt5/sdk` **0.10.5**. Check the current version first:
`npm view @agnt5/sdk version`. For the full API mapping read the `references/typescript.md`
of `agnt5-workflows`, `agnt5-agents-tools` and `agnt5-human-in-the-loop`; known TypeScript
bugs are listed in `agnt5-sdk-pitfalls/references/typescript.md`.

## Layout

```
<template-name>/
├── app.ts            # Worker entry point
├── package.json
├── tsconfig.json
├── agnt5.yaml
├── .env.example
├── README.md
└── src/
    ├── agents.ts
    ├── tools.ts      # only if agents need custom tools
    ├── functions.ts  # always create — fn(...) is the standard step unit in TS
    └── workflows.ts
```

## `package.json`

```json
{
  "name": "<template-name>",
  "version": "0.1.0",
  "type": "module",
  "private": true,
  "engines": { "node": ">=22" },
  "scripts": { "start": "npx tsx app.ts", "typecheck": "tsc --noEmit" },
  "dependencies": { "@agnt5/sdk": "^0.10.5", "tsx": "^4.21.0" },
  "devDependencies": { "@types/node": "^22.0.0", "typescript": "^5.9.3" }
}
```

`tsx` lives in `dependencies`, not `devDependencies`: the managed worker installs with
`npm install --production` and would otherwise download an unpinned `tsx` on every cold start
(AGNT5-1375).

## `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2022"],
    "types": ["node"],
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "dist",
    "rootDir": ".",
    "declaration": true,
    "sourceMap": true
  },
  "include": ["*.ts", "src/**/*.ts"]
}
```

`NodeNext` enforces the `.js` suffix on relative imports (`./src/functions.js`), which is what
Node needs at runtime under `"type": "module"`. `tsx` never type-checks; run
`npx tsc --noEmit` before handing the project over.

## `agnt5.yaml`

```yaml
name: <template-name>
language: typescript
language_version: ">=22"
environment: dev

worker:
  command: "npx tsx app.ts"

deploy:
  resources:
    memory: 512Mi
    cpu: 500m
```

## `src/agents.ts`

```typescript
import { Agent, LM } from '@agnt5/sdk';
import { myTool } from './tools.js';   // only if the agent uses custom tools

export const myAgent = new Agent({
    name: 'AgentName',
    model: LM.openai({ apiKey: process.env.OPENAI_API_KEY }),
    modelName: 'openai/gpt-4o-mini',
    // temperature: 1,   // required for openai/gpt-6-* models: the SDK otherwise sends 0.7 and OpenAI returns 400 (AGNT5-1302)
    instructions: 'You are <AgentName>, <one-line role>...',
    tools: [myTool],     // omit entirely if no tools — never pass tools: []
});
```

`model` is the provider client (`LM.openai()`, `LM.anthropic()`, ...); `modelName` is the
`provider/model` string and must match the client's provider. Optional: `maxIterations`
(default 10), `builtInTools: ['web_search']`, `handoffs`, `sandbox: new Sandbox({ provider })`,
`cache: true`, `skillsDir`/`skills`, `agentsMd`. Agents used as tools are exposed under their
own `name` (not `ask_<name>`).

## `src/tools.ts` — only if needed

```typescript
import { tool } from '@agnt5/sdk';
import type { Context } from '@agnt5/sdk';

export const myTool = tool(
    'my_tool',
    {
        description: 'One-line description the model reads to decide when to call this.',
        inputSchema: {
            type: 'object',
            properties: { param: { type: 'string', description: 'Description of the parameter.' } },
            required: ['param'],
        },
    },
    async (ctx: Context, args: { param: string }): Promise<string> => {
        ctx.logger.info('my_tool called', { param: args.param });
        return `result for ${args.param}`;
    },
);
```

`inputSchema` is mandatory in practice: without it the model sees a tool with no parameters.
The first handler parameter is always `ctx` (hidden from the model).

## `src/functions.ts`

```typescript
import { fn } from '@agnt5/sdk';
import type { Context } from '@agnt5/sdk';
import { myAgent } from './agents.js';

export const myStage = fn('my_stage')
    .inputSchema({
        type: 'object',
        properties: { data: { type: 'string' } },
        required: ['data'],
    })
    .run(async (ctx: Context, input: { data: string }): Promise<{ result: string }> => {
        ctx.logger.info('Stage started');
        const result = await myAgent.run(input.data, ctx);
        let output = result.output.trim();
        if (output.startsWith('LABEL:')) output = output.slice('LABEL:'.length).trim();
        return { result: output };
    });
```

Declare `inputSchema` on every `fn` and `workflow`: TypeScript types are erased, so Studio and
`agnt5 run` know the fields only from the schema. Retries/backoff/timeouts
(`.retry({ maxAttempts: 3 }).backoff({ type: 'exponential' }).timeout(10_000)`) apply to
standalone runs of the function; inside a workflow step use `executeWithRetry` — see
`agnt5-workflows`.

## `src/workflows.ts`

```typescript
import { workflow } from '@agnt5/sdk';
import type { Context } from '@agnt5/sdk';
import { stage1, stage2 } from './functions.js';

export const myWorkflow = workflow(
    'my_workflow',
    async (ctx: Context, input: { message: string }) => {
        const result1 = await ctx.step('stage1', () => stage1(ctx, { data: input.message }));
        const result2 = await ctx.step('stage2', () => stage2(ctx, { data: result1.result }));
        return { status: 'completed', output: result2.result };
    },
    {
        inputSchema: {
            type: 'object',
            properties: { message: { type: 'string' } },
            required: ['message'],
        },
    },
);
```

Every function call inside a workflow goes through `ctx.step(name, () => ..., { key })`. A
bare `stage1(ctx, ...)` is not checkpointed and re-runs on every replay (HITL resume, durable
sleep, crash recovery) — AGNT5-1373. Concurrent steps:
`await Promise.all(items.map((item) => ctx.step('process', () => process(ctx, { item }), { key: item.id })))`
— the `key` keeps replay matching the right checkpoint. There is no `ctx.batch`/`ctx.map`.

## `app.ts`

```typescript
import { Worker } from '@agnt5/sdk';

// Importing the modules registers their functions/workflows/tools.
import './src/functions.js';
import './src/workflows.js';
import { myAgent } from './src/agents.js';

process.on('unhandledRejection', (reason) => {
    console.error('unhandledRejection', reason);   // otherwise the worker process dies mid-run (AGNT5-1352)
});

async function main() {
    const worker = new Worker('<template-name>', {
        serviceVersion: '0.1.0',
        coordinatorEndpoint: process.env.AGNT5_COORDINATOR_ENDPOINT || 'http://localhost:34186',
    });
    worker.registerAgents([myAgent]);   // agents are NOT registered on import; autoRegister is ignored
    await worker.run();
}

main().catch((error) => {
    console.error('Worker error:', error);
    process.exit(1);
});
```

Omit the agents import and `registerAgents` call only when the template has no agents.
`agnt5 dev` sets `AGNT5_COORDINATOR_ENDPOINT` and loads `.env`; the fallback is for running
`npx tsx app.ts` by hand (then also `import 'dotenv/config'` first).

## Write order

`src/tools.ts` → `src/agents.ts` → `src/functions.ts` → `src/workflows.ts` → `app.ts` →
`package.json`, `tsconfig.json`, `agnt5.yaml`, `.env.example`, `README.md`.

Before handing off to `agnt5-project-init` (`npm install`, `.env`, `agnt5 dev`, `agnt5 run`):
`npm install && npx tsc --noEmit` must pass.
