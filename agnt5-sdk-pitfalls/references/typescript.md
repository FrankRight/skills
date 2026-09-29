# TypeScript SDK pitfalls

Verified against `@agnt5/sdk` 0.10.5 on 29 Sep 2026. One entry per known bug or gotcha,
ordered by how badly it bites. Each entry says when to delete it.

## 1. Direct function calls are not checkpointed

- **Symptom**: a charge, email or LLM call runs again after a HITL resume, a durable sleep or a crash recovery.
- **Cause**: `myFn(ctx, input)` inside a workflow emits events but writes no checkpoint; only `ctx.step` does. The docs page that says `fn(...).run(...)` checkpoints on its own is wrong for 0.10.5.
- **Workaround**: `await ctx.step('name', () => myFn(ctx, input), { key })` for every call; unique `key` in loops and `Promise.all`.
- **Linear**: AGNT5-1373
- **Remove when**: the `fn()` wrapper checkpoints through the activation client, or the docs and SDK agree.

## 2. An unhandled promise rejection kills the worker

- **Symptom**: the worker process exits mid-run with no AGNT5 error; all in-flight runs fail.
- **Cause**: Node's default `unhandledRejection` behaviour is to exit; the SDK installs no handler.
- **Workaround**: `process.on('unhandledRejection', (reason) => console.error('unhandledRejection', reason));` at the top of `app.ts`, and `try/catch` in tool and chat handlers.
- **Linear**: AGNT5-1352
- **Remove when**: `Worker` installs its own handler.

## 3. Agents are not auto-registered; `autoRegister` is dead

- **Symptom**: functions and workflows show up in `agnt5 components`, the agent does not; `agnt5 run my_agent --type agent` fails.
- **Cause**: `fn()`, `workflow()`, `tool()`, `scorer()` register on import; `Agent` only joins `AgentRegistry`, which `Worker.run()` never reads. `PlatformWorkerOptions.autoRegister` is typed but never referenced.
- **Workaround**: `worker.registerAgents([agent, chatBot])` before `worker.run()`.
- **Linear**: none filed (verified in `worker.ts`).
- **Remove when**: `Worker.run()` registers `AgentRegistry.all()` or `autoRegister` is implemented.

## 4. `fn().retry()` is ignored inside `ctx.step`; the CLI shows the first failed attempt

- **Symptom**: a function with `.retry({ maxAttempts: 3 })` fails once inside a workflow and the step fails. Separately, `agnt5 run <function>` prints an error and exits 1 while the platform keeps retrying and the run later succeeds.
- **Cause**: retry config is registered with the platform for top-level function runs only; nested calls go through the step callback with no retry loop.
- **Workaround**: `ctx.step('x', () => executeWithRetry(() => myFn(ctx, input), { retryPolicy: { maxAttempts: 3 }, backoffPolicy: 'exponential', context: ctx }))`; after `agnt5 run`, check `agnt5 inspect runs describe <runId>` for the final status.
- **Linear**: AGNT5-1372
- **Remove when**: nested function invocations honour `FunctionOptions.retries` and the CLI waits for the final attempt.

## 5. gpt-6 models reject the default temperature; `ReasoningEffort` type is narrow

- **Symptom**: `Agent.run` / `LM.generate` on `openai/gpt-6-*` returns HTTP 400 about `temperature`. `reasoningEffort: 'low'` fails to type-check; `'minimal'` type-checks but 400s on `gpt-6-luna`.
- **Cause**: `Agent` sends `temperature: 0.7` unless set; the reasoning-model detector only matches `gpt-5*`, `o1*`, `o3*`, `o4*`. `ReasoningEffort` is `'minimal' | 'medium' | 'high'`.
- **Workaround**: `temperature: 1` on the agent / `config.temperature: 1` on `generate`; `reasoningEffort: 'low' as ReasoningEffort` (`'low'`/`'none'` work at runtime).
- **Linear**: AGNT5-1302 (temperature), AGNT5-1285 (gpt-6 parameter support, decision pending), AGNT5-1327 (`ReasoningEffort`).
- **Remove when**: the detector covers gpt-6 (or omits temperature by default) and the union includes `'low'`/`'none'`.

## 6. Webhook/event-triggered workflows receive an event record, not the envelope

- **Symptom**: `JSON.parse(input.body)` throws — `input.body` is `undefined`.
- **Cause**: the run input is `{ event: { id, name, data: <envelope>, source, timestamp_ns }, deployment_id, target_kind, target_ref }`; the webhook envelope (`_webhook, source, integration_id, event_type, idempotency_key, timestamp, headers, body`) sits at `input.event.data`. The docs show the envelope at the top level.
- **Workaround**: `const env = '_webhook' in input ? input : input.event.data; const payload = JSON.parse(env.body);`
- **Linear**: none filed (verified against the gateway's event dispatch and a live run).
- **Remove when**: the docs describe the `event.data` shape or the gateway unwraps it.

## 7. HITL answers arrive as strings, including `"null"`

- **Symptom**: `topics.split(',')` yields garbage; a skipped question is not `null`.
- **Cause**: resume-API answers are strings: multiselect is a JSON array string (`'["a","c"]'`), a skip is the string `"null"`, approval/select is the id string.
- **Workaround**: treat `null`, `'null'` and `''` as skipped; `JSON.parse` the multiselect string with a `split(',')` fallback.
- **Linear**: none filed.
- **Remove when**: `waitForUser` returns parsed values / real `null`.

## 8. `waitForUser` has no timeout; `waitForSignal` throws

- **Symptom**: a paused run waits forever; `ctx.waitForSignal(...)` throws `ConfigurationError` locally and fails on managed workers.
- **Cause**: no timeout option on `waitForUser`; `waitForSignal` is implemented only for the serverless runtime.
- **Workaround**: enforce deadlines outside the run (operator/cron); replace signals with a webhook-triggered workflow or a polling `ctx.step`.
- **Linear**: AGNT5-1355
- **Remove when**: `waitForUser` accepts a timeout and `waitForSignal` works on managed workers.

## 9. `tsx` is installed unpinned on every cold start

- **Symptom**: slow deployed cold starts; startup breaks when a new `tsx` release lands.
- **Cause**: the pod runs `npm install --production` (drops devDependencies) and then `npx tsx app.ts`, so `npx` fetches `tsx` at start.
- **Workaround**: move `tsx` to `dependencies`; commit `package-lock.json`.
- **Linear**: AGNT5-1375
- **Remove when**: the node worker image bundles `tsx` or installs devDependencies.

## 10. No trace spans; every failure is `EXECUTION_ERROR`

- **Symptom**: `agnt5 inspect trace -r <runId>` and Studio's Trace tab are empty for TS runs; failed runs never show the real error class.
- **Cause**: the TypeScript worker exports no OTel spans; `processMessage` maps every thrown error to `EXECUTION_ERROR`.
- **Workaround**: use `ctx.logger` and journal events (`client.getEvents`); log `err.name` / `err.message` before rethrowing.
- **Linear**: AGNT5-1320 (spans), AGNT5-1358 (error code)
- **Remove when**: spans appear for TS runs and error codes are propagated.

## 11. `maxIterations` returns the last tool result as the answer

- **Symptom**: `result.output` is a tool's JSON string after a long agent run.
- **Cause**: on hitting `maxIterations` the agent returns `messages[messages.length - 1].content`, which is the last tool result when the loop ended after a tool call.
- **Workaround**: raise `maxIterations`, or check `result.toolCalls.length` / the final message role and re-prompt for a summary.
- **Linear**: AGNT5-1353
- **Remove when**: the agent makes a final model call (or flags truncation) at the iteration cap.

## 12. `saga()` hides compensation failures

- **Symptom**: a compensating action throws, but `saga` rethrows only the original step error and the caller never learns the rollback is incomplete.
- **Cause**: each compensation error is caught and only written to `ctx.logger.error` while unwinding.
- **Workaround**: wrap each compensation in its own `try/catch` that logs, or write the unwind loop yourself with `ctx.step`.
- **Linear**: AGNT5-1354
- **Remove when**: `saga` surfaces compensation errors (e.g. `AggregateError`).

## 13. Handoffs are constructor-only and unbounded

- **Symptom**: no way to add a handoff after construction; two agents that hand off to each other loop until each hits `maxIterations`.
- **Cause**: `handoffs` are turned into transfer tools in the `Agent` constructor; there is no depth counter.
- **Workaround**: build the agent graph once; avoid mutual handoffs; keep `maxIterations` low on specialists.
- **Linear**: AGNT5-1357
- **Remove when**: a handoff depth limit exists.

## 14. `withTimeout` leaks its timer; child-workflow helpers run in-process

- **Symptom**: a resolved `withTimeout` keeps the event loop alive until the timer fires; `executeChildWorkflow` / `fanOut` / `batchExecute` / `race` create no child run and share the parent `ctx` (same step counter, same checkpoints).
- **Cause**: `setTimeout` is never cleared; child helpers call the handler directly.
- **Workaround**: prefer `ctx.step` + `Promise.all`; if you need a real child run, call it through `Client.run` from a step.
- **Linear**: AGNT5-1359
- **Remove when**: the helpers dispatch real child runs and clear timers.

## 15. `LM.generate` / `stream` ignore `AbortSignal`

- **Symptom**: cancelling a run (`ctx.signal` aborted) does not stop an in-flight model call.
- **Cause**: the request has no signal parameter and the provider call does not observe `ctx.signal`.
- **Workaround**: check `ctx.signal.aborted` between iterations; keep `maxOutputTokens` bounded.
- **Linear**: AGNT5-1356
- **Remove when**: `GenerateRequest` accepts a signal.

## 16. Any `beforeModel` / `afterModel` callback disables token streaming

- **Symptom**: `lm.message.delta` events stop as soon as a model callback is configured.
- **Cause**: `canStreamModel()` returns `false` when either callback is set.
- **Workaround**: use `beforeTool` / `afterAgent` for guardrails where possible; accept non-streamed output otherwise.
- **Linear**: none filed (verified in `agent.ts`).
- **Remove when**: model callbacks are applied on the streaming path.

## 17. Tools without `inputSchema` have no parameters

- **Symptom**: the model calls the tool with `{}` or never fills arguments.
- **Cause**: no schema inference; the default schema is `{ type: 'object', properties: {}, required: [] }`.
- **Workaround**: always pass `inputSchema` (or `zodToJsonSchema` / `typeBoxToJsonSchema`).
- **Linear**: none — by design.
- **Remove when**: never; keep as guidance.

## 18. Agent-as-tool is named `<agent.name>`, not `ask_<name>`

- **Symptom**: instructions that mention `ask_researcher` do not match any tool.
- **Cause**: `Agent.addTool` wraps a sub-agent under its own `name` with a single `message` argument.
- **Workaround**: reference the agent's `name` in instructions.
- **Linear**: none filed (docs page shows `ask_lookup`).
- **Remove when**: naming is aligned with Python or the docs are corrected.

## 19. `Client` defaults to `http://localhost:34181`

- **Symptom**: `ECONNREFUSED 127.0.0.1:34181` from a backend or CI job.
- **Cause**: `new Client()` uses `AGNT5_GATEWAY_URL` or the local dev gateway, not `https://gw.agnt5.com`.
- **Workaround**: set `AGNT5_GATEWAY_URL` and `AGNT5_API_KEY`, or pass `gatewayUrl`/`apiKey`.
- **Linear**: none filed.
- **Remove when**: the default matches Python or the docs say localhost.

## 20. `AGNT5_SANDBOX_PROVIDER` is ignored

- **Symptom**: the env var selects nothing; `new Sandbox()` auto-picks the first provider with credentials.
- **Workaround**: `new Sandbox({ provider: 'e2b' })`.
- **Linear**: none filed.
- **Remove when**: `Sandbox` reads the variable.

## 21. `ctx` has no `state`, `session`, `user`, `memory`, `conversation`

- **Symptom**: Python-shaped code (`ctx.state.set`, `ctx.session.state`, `ctx.memory.user.save`) fails to compile.
- **Workaround**: run-scoped `await ctx.set/get/delete`; standalone `ConversationMemory` / `SemanticMemory` (in-memory by default); pass `sessionId` at call time.
- **Linear**: none filed.
- **Remove when**: `Context` gains session/user scopes.

## 22. `npx tsx` never type-checks

- **Symptom**: type errors surface only at runtime or in CI.
- **Workaround**: `npx tsc --noEmit` (add a `typecheck` script).
- **Remove when**: never; keep as guidance.

## 23. Smaller mismatches

- `import { VERSION } from '@agnt5/sdk'` returns `"0.6.0"` on 0.10.5 — use `npm ls @agnt5/sdk`. Remove when the constant is generated from `package.json`.
- `new AskUserTool(ctx)` / `new RequestApprovalTool(ctx)` take `ContextImpl`; cast the workflow `ctx`. Remove when the constructors accept `Context`.
- `import { Prompt } from '@agnt5/sdk'` is the MCP server prompt class; LM prompts are plain `{ id, variables, version }` objects (`LMPrompt`). Remove when a dedicated LM prompt export exists.
- Capture content switch is `AGNT5_LLM_CAPTURE_CONTENT=off`, not `AGNT5_CAPTURE_CONTENT_MODE` / `AGNT5_CAPTURE_MAX_CONTENT_CHARS`. Remove when the variable names are aligned.
- Evaluator presets default `includeInput` to `false` (Python: `true` for most). Remove when aligned.
- `agent.stream()` yields the final `AgentResult` (no `eventType`) mixed in with events; guard with `'eventType' in item`. Remove when the result is delivered separately.
- `ctx.sleep` / `fn().timeout` / `Client` timeouts are milliseconds; Python uses seconds. Keep as guidance.
