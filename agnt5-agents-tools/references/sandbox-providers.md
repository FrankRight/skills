# Sandbox providers — credentials and setup

`Sandbox()` auto-detects the first configured provider; pass `provider=` explicitly when more
than one is configured.

| Provider | Selector | Required env vars |
|---|---|---|
| E2B | `e2b` | `E2B_API_KEY` |
| Daytona | `daytona` | `DAYTONA_API_KEY` |
| Vercel Sandbox | `vercel` | `VERCEL_OIDC_TOKEN` alone, or `VERCEL_TOKEN` + `VERCEL_TEAM_ID` + `VERCEL_PROJECT_ID` |
| Northflank | `northflank` | `NORTHFLANK_API_TOKEN`, `NORTHFLANK_PROJECT_ID` |
| Together Code Interpreter | `together` | `TOGETHER_API_KEY` |

**Local dev** — put credentials in `.env` and start with the explicit env file:

```bash
AGNT5_SANDBOX_PROVIDER=e2b
E2B_API_KEY=e2b_...
```
```bash
agnt5 --env-file .env dev
```

**Deployed workers** — Studio → Settings → Integrations → add the sandbox provider
integration, store the credential at the narrowest scope that works, then deploy/restart so
the worker picks it up. The credential is never shown to the model — only the worker uses it
to create/manage the sandbox.

**Validate a provider before relying on it** — scaffold the `sandbox-smoke` template and run
its checks:

```bash
agnt5 create --template python/sandbox-smoke sandbox-smoke
cd sandbox-smoke && cp .env.example .env
agnt5 --env-file .env dev

agnt5 --env-file .env run sandbox_lifecycle_check --type workflow \
  --input '{"provider": "e2b", "code": "print(6 * 7)", "language": "python"}'
agnt5 --env-file .env run sandbox_agent_tools_check --type workflow --input '{"provider": "e2b"}'
agnt5 --env-file .env run sandbox_coding_agent_check --type workflow \
  --input '{"provider": "e2b", "model": "openai/gpt-4o-mini"}'
```

| Symptom | Check |
|---|---|
| `Sandbox provider 'auto' is not configured` | Worker has no supported provider env vars — restart with `agnt5 --env-file .env dev` or update the deployed worker's environment |
| Provider creation fails | Provider key is valid and the account has sandbox access enabled |
| File ops fail but code execution works | Run `sandbox_agent_tools_check` to isolate write/list/read/execute |
| Shutdown doesn't complete | Provider-side quota, active sandbox limits, provider API status |

(`--type workflow` assumes the template's checks are workflows; if `agnt5 components` lists them as functions, drop the flag.)

Source: https://agnt5.com/docs/integrations/sandbox-providers
