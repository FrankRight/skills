# Deploying a TypeScript worker

Verified against `@agnt5/sdk` **0.10.5**. Secrets, `agnt5 deploy` flags, verification,
promotion, rollback and scaling in the SKILL.md are language-independent. This file covers
what the managed Node worker actually runs and the checks a TypeScript project needs first.

## What the managed worker runs

- Base image: `ghcr.io/agnt5dev/node-worker:24` (Node 24). Override with
  `agnt5 deploy --base-image <image>`.
- The pod installs your dependencies **from the lockfile, devDependencies included**, then
  runs the `agnt5.yaml` `worker.command` (`npx tsx app.ts`). With `package-lock.json` it runs
  `npm ci --include=dev`. Other lockfiles are honoured only in some cases; without a usable
  one it runs `npm install --include=dev`, which resolves versions afresh. `tsx` can
  therefore stay in `devDependencies`, as the templates have it.
- Commit `package-lock.json`, and run `npm install` after every `package.json` edit:
  `npm ci` refuses to install when the two disagree ("`npm ci` can only install packages when
  your package.json and package-lock.json … are in sync").
- The code bundle always leaves out `node_modules/`, `dist/` and `build/` (plus your
  `.gitignore` and `.agnt5ignore` patterns), so a compiled `worker.command: "node dist/app.js"`
  finds nothing to run. Keep `npx tsx app.ts`.
- `AGNT5_COORDINATOR_ENDPOINT` and project secrets are injected as environment variables
  when the worker starts; `.env` is local only. `Client` inside a deployed backend still
  defaults to `http://localhost:34181` — set `AGNT5_GATEWAY_URL` and `AGNT5_API_KEY` there.

## Pre-deploy checklist

```bash
npx tsc --noEmit                      # tsx never type-checks
npm ci --include=dev                  # the install the pod runs; fails if package-lock.json is stale
grep -n unhandledRejection app.ts     # process.on handler present
grep -n registerAgents app.ts         # every Agent registered
agnt5 secrets list                    # OPENAI_API_KEY etc. present for the target env
agnt5 deploy --dry-run
```

## Deploy, verify, promote

Same commands as the SKILL.md (`agnt5 deploy --env staging`, `agnt5 deployment status --watch`,
`agnt5 deploy debug <deployment-id> --logs`, `agnt5 deployment promote --latest --env production`).

Smoke test the deployed worker by deployment ID (printed by `agnt5 deploy`):

```bash
agnt5 run my_workflow --type workflow --input '{"message": "..."}' --deployment-id <deployment-id>
agnt5 run my_agent --type agent --input '{"message": "..."}' --deployment-id <deployment-id>
```

Reading logs after deploy:

- `ctx.logger.*` and `getLogger('...')` records reach the run's logs: MCP `get_run_logs` or
  the run in Studio (`agnt5 inspect logs -r` currently returns 403). `ctx.logger` attribute
  values must be strings — `{ attempt: String(ctx.attempt) }`; a number fails the run with
  ``Failed to convert JavaScript value `Number …` into rust type `String` ``.
- Plain `console.log` / `console.error` from a deployed worker are not shown anywhere:
  not in the run's logs, and not in `agnt5 logs <deployment-id>`, which carries the
  platform's lifecycle log. Locally they print in the `agnt5 dev` terminal. If the worker
  crashes, `agnt5 deploy debug <deployment-id> --logs` shows its last output line.
- There are no trace spans for TypeScript runs and every failure is reported as
  `EXECUTION_ERROR`, so log `err.name` and `err.message` yourself before
  rethrowing.

## Calling the deployed worker from your app

```typescript
import { Client } from '@agnt5/sdk';

const client = new Client({
  gatewayUrl: process.env.AGNT5_GATEWAY_URL ?? 'https://gw.agnt5.com',
  apiKey: process.env.AGNT5_API_KEY,          // agnt5 service-keys create --name <name> --project <id>
});
```

`--environment` on `agnt5 service-keys create` takes an environment ID (`env_id` in
`agnt5 deployment list -o json`). See `agnt5-client` for the full surface.

## TypeScript pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| Worker never becomes ready after a `package.json` change | `package-lock.json` out of sync, so the pod's `npm ci` fails (check with `npm ci --include=dev` locally) | `npm install`, commit the lockfile, redeploy |
| Deploy succeeds, worker restarts in a loop | type error / bad import that `tsx` only hits at runtime | `npx tsc --noEmit` before deploying |
| Run fails with ``Failed to convert JavaScript value `Number …` into rust type `String` `` | non-string `ctx.logger` attribute | `String(value)` |
| Agents work locally, missing after deploy | registered in a dev-only code path | `worker.registerAgents([...])` unconditionally |
| Pod dies after one failed run | unhandled rejection | `process.on('unhandledRejection', ...)` |
| Backend calls fail with `ECONNREFUSED 127.0.0.1:34181` | `Client` default gateway | set `AGNT5_GATEWAY_URL` |
| Sandbox provider ignored in the deployed worker | `AGNT5_SANDBOX_PROVIDER` is not read by the TS SDK | `new Sandbox({ provider: 'e2b' })` + the provider's key as a secret |

## Source

https://agnt5.com/docs/run/deploying · https://agnt5.com/docs/run/environments
