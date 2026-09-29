# Deploying a TypeScript worker

Verified against `@agnt5/sdk` **0.10.5**. Secrets, `agnt5 deploy` flags, verification,
promotion, rollback and scaling in the SKILL.md are language-independent. This file covers
what the managed Node worker actually runs and the checks a TypeScript project needs first.

## What the managed worker runs

- Base image: `ghcr.io/agnt5dev/node-worker:24` (Node 24). Override with
  `agnt5 deploy --base-image <image>`.
- The pod runs `npm install --production` inside the bundled project, then the `agnt5.yaml`
  `worker.command` (`npx tsx app.ts`).
- `--production` skips `devDependencies`. With `tsx` in `devDependencies`, `npx tsx` downloads
  an unpinned `tsx` on every cold start (slow, and a new `tsx` release can break startup).
  Move it to `dependencies`:

```json
{
  "type": "module",
  "scripts": { "start": "npx tsx app.ts", "typecheck": "tsc --noEmit" },
  "dependencies": { "@agnt5/sdk": "^0.10.5", "tsx": "^4.21.0" },
  "devDependencies": { "@types/node": "^22.0.0", "typescript": "^5.9.3" }
}
```

  Alternative: compile with `tsc` and set `worker.command: "node dist/app.js"` — only if the
  compiled `dist/` is part of the uploaded bundle (whether the bundler honours `.gitignore`
  is not verified; keep `tsx` in `dependencies` unless you have checked).
- Commit `package-lock.json` so the install is reproducible.
- `AGNT5_COORDINATOR_ENDPOINT` and project secrets are injected as environment variables;
  `.env` is local only. `Client` inside a deployed backend still defaults to
  `http://localhost:34181` — set `AGNT5_GATEWAY_URL` and `AGNT5_API_KEY` there.

## Pre-deploy checklist

```bash
npx tsc --noEmit                      # tsx never type-checks
grep -n '"tsx"' package.json          # must be under dependencies
grep -n unhandledRejection app.ts     # process.on handler present
grep -n registerAgents app.ts         # every Agent registered
agnt5 secrets list                    # OPENAI_API_KEY etc. present for the target env
agnt5 deploy --dry-run
```

## Deploy, verify, promote

Same commands as the SKILL.md (`agnt5 deploy --env staging`, `agnt5 deployment status --watch`,
`agnt5 deploy debug <deployment-id> --logs`, `agnt5 deployment promote --latest --env production`).

Smoke test the deployed worker:

```bash
agnt5 run my_workflow --type workflow --input '{"message": "..."}' --env production
agnt5 run my_agent --type agent --input '{"message": "..."}' --env production
```

Reading logs after deploy:

- `ctx.logger.*` and `getLogger('...')` records reach the run's logs
  (`agnt5 inspect logs -r <runId>`) and stdout.
- Plain `console.log` / `console.error` reach only the deployment logs
  (`agnt5 logs <deployment-id> --follow`), never a run's logs.
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

See `agnt5-client` for the full surface.

## TypeScript pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| Slow cold start, or startup fails with an `npx` download error | `tsx` in devDependencies, installed with `--production` | move `tsx` to `dependencies` |
| Deploy succeeds, worker restarts in a loop | type error / bad import that `tsx` only hits at runtime | `npx tsc --noEmit` before deploying |
| Agents work locally, missing after deploy | registered in a dev-only code path | `worker.registerAgents([...])` unconditionally |
| Pod dies after one failed run | unhandled rejection | `process.on('unhandledRejection', ...)` |
| Backend calls fail with `ECONNREFUSED 127.0.0.1:34181` | `Client` default gateway | set `AGNT5_GATEWAY_URL` |
| Sandbox provider ignored in the deployed worker | `AGNT5_SANDBOX_PROVIDER` is not read by the TS SDK | `new Sandbox({ provider: 'e2b' })` + the provider's key as a secret |

## Source

https://agnt5.com/docs/run/deploying · https://agnt5.com/docs/run/environments
