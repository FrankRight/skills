# Project setup and local development in TypeScript

Verified against `@agnt5/sdk` **0.10.5**. Steps 0 (CLI install, auth) and 4 (trigger a run)
in the SKILL.md are language-independent. This file covers what changes for a TypeScript
project.

## 1. Create or link the project

```bash
agnt5 create my-project --language typescript          # blank TypeScript starter
agnt5 init --language typescript                        # link the current directory
agnt5 create my-digest --template typescript/quickstart # scaffold from a template (agnt5-ai-templates)
```

A TypeScript `agnt5.yaml`:

```yaml
name: my-project
language: typescript
language_version: ">=22"
environment: dev

worker:
  command: "npx tsx app.ts"    # what `agnt5 dev` and the deployed pod run
```

## 2. Install dependencies and configure `.env`

```bash
node --version           # 22 or newer (managed workers run Node 24)
npm install              # writes package-lock.json — commit it
cp .env.example .env     # then fill in real keys, e.g. OPENAI_API_KEY=sk-...
npx tsc --noEmit         # type-check; `tsx` strips types without checking
```

`agnt5 dev` loads `.env` into the worker and reloads on `.env` edits. If you start the worker
by hand (`npx tsx app.ts`), add `import 'dotenv/config';` as the first line of `app.ts`
(`npm install dotenv`), as the `weather-agent` and `coding_agent` templates do.

Registration model: `fn()`, `workflow()`, `tool()` and `scorer()` register when their module
is imported, so `app.ts` must import every module (`import './src/functions.js';`). Agents are
**not** auto-registered: `worker.registerAgents([agent])` before `worker.run()`. The
`autoRegister` worker option is typed but never read.

## 3. Start the worker

```bash
agnt5 dev                # runs agnt5.yaml worker.command, hot reload on .ts .js .env
agnt5 dev -d             # detached; agnt5 dev status | logs | stop
agnt5 dev -v             # verbose; AGNT5_DEBUG=1 raises the SDK log level to DEBUG
```

The SDK banner prints `Service`, `Worker ID`, `Coordinator`, `Runtime: node`, then the
component list. `agnt5 dev` sets `AGNT5_COORDINATOR_ENDPOINT`; the SDK default is
`http://localhost:34186`, so keep the `process.env.AGNT5_COORDINATOR_ENDPOINT || 'http://localhost:34186'`
fallback and never hardcode a remote endpoint.

Put this in `app.ts` — an unhandled promise rejection otherwise terminates the worker process
mid-run (AGNT5-1352):

```typescript
process.on('unhandledRejection', (reason) => {
  console.error('unhandledRejection', reason);
});
```

## 4. Trigger a run

Same `agnt5 run` / Studio flow. TypeScript-specific notes:

- Workflow and function inputs are one JSON object matching the handler's second parameter
  (`--input '{"userEmail": "..."}'`).
- `agnt5 run my_function` prints the first failed attempt and exits 1 while the platform is
  still retrying a function with `.retry()`; confirm with `agnt5 inspect runs describe <runId>`
  (AGNT5-1372).
- Studio shows no input form unless the component declares `inputSchema` (types are erased).

## Common errors

| Error | Fix |
|---|---|
| `Cannot find package '@agnt5/sdk'` | `npm install` (quickstart README: install before `agnt5 dev`) |
| `ERR_MODULE_NOT_FOUND ... './src/functions'` | ESM needs the `.js` suffix on relative imports even for `.ts` sources: `./src/functions.js` |
| `SyntaxError: Cannot use import statement outside a module` | add `"type": "module"` to `package.json` |
| `TS7006` / type errors appear only in CI | `tsx` does not type-check; run `npx tsc --noEmit` locally |
| Component missing from the banner / `agnt5 components` | module not imported in `app.ts`; for agents, `worker.registerAgents([...])` |
| `ECONNREFUSED ...:34186` at startup | `agnt5 dev` is not running, or a stale `AGNT5_COORDINATOR_ENDPOINT` |
| `OPENAI_API_KEY` errors / runs fail immediately | key missing from `.env`; when running by hand add `import 'dotenv/config'` |
| `ConfigurationError: Provider 'openai' does not match model prefix` | `modelName` prefix must match the `LM` provider |
| `TS2345: ... not assignable to parameter of type 'ContextImpl'` | `new AskUserTool(ctx as ContextImpl)` |
| Worker exits with no stack trace mid-run | unhandled rejection — add the handler above (AGNT5-1352) |
| Every failed run reports `EXECUTION_ERROR` | the worker collapses error codes (AGNT5-1358); read the message / your own logs |
| `getBindingType()` is not `'napi'` | the platform-specific optional dependency (`@agnt5/sdk-linux-x64-gnu`, `-linux-arm64-gnu`, `-darwin-arm64`) did not install; re-run `npm install` on a supported platform |
| Anything else | `agnt5 dev -v`, then `agnt5 inspect logs -r <runId>` |

After moving the project directory: `npm install && agnt5 init && agnt5 dev`.

## Not available in TypeScript

- `uv sync` / `pyproject.toml` (npm + `package.json`), `Worker(auto_register=True)`
- `agnt5[openai]`-style extras: capture libraries are plain npm dependencies (`openai`, `@openai/agents`, `ai`, `@google/adk`)
- Trace output for TS runs in `agnt5 inspect trace` (AGNT5-1320) — use logs

## Source

https://agnt5.com/docs/quickstart (TypeScript tab) · https://agnt5.com/docs/build/local-development
