---
name: agnt5-sdk-pitfalls
description: Known bugs and gotchas in the current AGNT5 SDK releases (Python 0.13.6, TypeScript @agnt5/sdk 0.10.5, Go sdk-go v0.10.3), each with its symptom, cause, workaround and Linear issue. Read the file for your language before writing or debugging AGNT5 code, when a run behaves differently from what the docs or another skill say, when a model call returns a 400, when a retry, checkpoint, trace or structured output seems missing, or when a workflow re-runs work after a pause. Entries are removed as the fixes ship.
---

# AGNT5 SDK pitfalls

The other skills describe how the SDKs are meant to work. This one lists where the current
releases don't, so you can work around them instead of debugging them from scratch.

Verified on 29 Sep 2026 against:

| SDK | Version | File |
|-----|---------|------|
| Python `agnt5` | 0.13.6 | [references/python.md](references/python.md) |
| TypeScript `@agnt5/sdk` | 0.10.5 | [references/typescript.md](references/typescript.md) |
| Go `github.com/agnt5dev/sdk-go` | v0.10.3 | [references/go.md](references/go.md) |

Read only the file for the language you are working in. Each entry gives the symptom you
will see, the cause, the workaround to apply now, and the Linear issue tracking the fix.

## Before you start

1. Check the installed version: `pip show agnt5`, `npm ls @agnt5/sdk`, or `go list -m github.com/agnt5dev/sdk-go`.
   If it is newer than the version above, the entry may already be fixed; check the Linear issue.
2. Apply the workaround in code, and leave a comment naming the Linear issue so it can be
   removed later, for example `# AGNT5-1323: gpt-6 rejects the default temperature`.
3. Do not "fix" the SDK behaviour from inside application code beyond the workaround given;
   the platform side may change with the fix.

## Cross-SDK entries

These affect more than one language and are repeated in each file:

- **gpt-6 models** (`openai/gpt-6-luna` and family) accept only their default temperature.
  The SDKs still send one, so calls fail with a 400 unless you opt out. Python
  `temperature=None`, TypeScript `temperature: 1`, Go: stay on a non-reasoning model.
  Whether the SDKs will expose gpt-6's other parameters is a pending decision (AGNT5-1285).
- **Retries inside workflow steps** are not applied in Python or TypeScript: a function with a
  retry policy is retried when run on its own, but a workflow calling it through `ctx.step`
  gets the first attempt's error (AGNT5-1372). `agnt5 run function` also prints the first
  failed attempt as the final result while the platform keeps retrying; check the stored run.
- **LLM judges on gpt-6 models** fail with a 400 in every SDK and on the platform, and the
  failure is recorded as a score of 0, not as an error. Use a non-gpt-6 judge model
  (AGNT5-1374 for Go; the Python presets behave the same).
- **Webhook and event triggers** hand the workflow the gateway's trigger envelope, not the raw
  webhook: the request body is at `event.data.body`. See `agnt5-webhooks-integrations` and its
  language references.

## Keeping this skill current

When an SDK release closes a Linear issue listed here, delete the entry and bump the version
in the table above. Entries without a Linear issue are documented behaviour, not bugs; keep
those until the SDK changes.
