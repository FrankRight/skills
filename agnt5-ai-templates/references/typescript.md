# TypeScript templates

Verified against `@agnt5/sdk` **0.10.5**. Check the current version first:
`npm view @agnt5/sdk version`.

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
  "scripts": { "start": "npx tsx app.ts" },
  "dependencies": { "@agnt5/sdk": "^0.10.5" },
  "devDependencies": { "@types/node": "^22.0.0", "tsx": "^4.0.0", "typescript": "^5.0.0" }
}
```

## `agnt5.yaml`

```yaml
name: <template-name>
language: typescript
language_version: "22"
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

export const myAgent = new Agent({
    name: 'AgentName',
    model: LM.openai({ apiKey: process.env.OPENAI_API_KEY }),
    modelName: 'openai/gpt-4o-mini',
    instructions: 'You are <AgentName>, <one-line role>...',
});
```

## `src/functions.ts`

```typescript
import { fn } from '@agnt5/sdk';
import type { Context } from '@agnt5/sdk';

export const myStage = fn('my_stage').run(
    async (ctx: Context, input: { data: string }): Promise<{ result: string }> => {
        ctx.logger.info('Stage started');
        return { result: 'output' };
    },
);
```

## `src/workflows.ts`

```typescript
import { workflow } from '@agnt5/sdk';
import type { Context } from '@agnt5/sdk';
import { stage1, stage2 } from './functions.js';

export const myWorkflow = workflow(
    'my_workflow',
    async (ctx: Context, input: { message: string }) => {
        const result1 = await stage1(ctx, { data: input.message });
        const result2 = await stage2(ctx, { data: result1.result });
        return { status: 'completed', output: result2.result };
    },
);
```

Concurrent steps: `await Promise.all(items.map((item) => myStep(ctx, { item })))`, or the
`parallel` / `gather` / `batchExecute` helpers exported from `@agnt5/sdk`.

## `app.ts`

```typescript
import { Worker } from '@agnt5/sdk';

// Importing the modules registers their functions/workflows.
import './src/functions.js';
import './src/workflows.js';

async function main() {
    const worker = new Worker('<template-name>', {
        serviceVersion: '0.1.0',
        coordinatorEndpoint: process.env.AGNT5_COORDINATOR_ENDPOINT || 'http://localhost:34186',
    });
    await worker.run();
}

main().catch((error) => {
    console.error('Worker error:', error);
    process.exit(1);
});
```

## Write order

`src/agents.ts` → `src/functions.ts` → `src/workflows.ts` → `app.ts` → `package.json`,
`tsconfig.json`, `agnt5.yaml`, `.env.example`, `README.md`.
