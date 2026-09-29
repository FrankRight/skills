# Go agents and tools

Verified against `github.com/agnt5dev/sdk-go` **v0.10.3** (SDK source; the product docs
understate Go — `WithAgentSandbox`, skills and `SandboxTools` all exist).

## Concept map: `Agent(...)` → `agnt5.NewAgent(name, opts...)`

| Python parameter | Go |
|---|---|
| `model="openai/gpt-4o-mini"` | `agnt5.WithAgentModel(agnt5.NewOpenAIModel(agnt5.OpenAIConfig{Model: "gpt-4o-mini", APIKey: os.Getenv("OPENAI_API_KEY")}))` — bare name, explicit key |
| `instructions=` | `agnt5.WithAgentInstructions(text)` |
| `tools=[...]` | `agnt5.WithAgentTools(tools...)` (`agnt5.Tool` values; agents are not auto-wrapped — see below) |
| `built_in_tools=` | none |
| `handoffs=` | `agnt5.WithAgentHandoffs(handoffs...)` from `agnt5.NewHandoff` |
| `sandbox=Sandbox()` | `agnt5.WithAgentSandbox(runner)` — adds the four `sandbox_*` tools |
| `max_iterations=10` | `agnt5.WithAgentMaxTurns(n)` (default 10; `0` is ignored) |
| `temperature` / `max_tokens` / `top_p` | none (wrap the model, below) |
| `model_config=ModelConfig(base_url, api_key, timeout, headers)` | fields on the provider config: `BaseURL`, `APIKey`, `Headers`, `HTTPClient` |
| `cache=True` / `PromptCache(...)` | `agnt5.WithAgentPromptCache(agnt5.EnablePromptCache())` — `agnt5-prompts` |
| `callbacks` / `before_*` / `after_*` | none (wrap tools/model, below) |
| `skills` / `skills_dir` / `agents_md` | `WithAgentSkills`, `WithAgentSkillsFromDir`, `WithAgentGuidance` — `agnt5-agent-skills` |

```go
result, err := agent.Run(ctx, agnt5.AgentInput{Message: "...", Messages: priorTurns}) // ctx *agnt5.Context
result.Response        // final text (Python result.output)
result.ToolCallDetails // []agnt5.AgentToolCall{ID, Name, Arguments, Iteration, Result, Error, Handoff}
result.ToolCalls       // count
result.HandoffTo       // name of the agent that took over, if any
result.Messages        // full transcript
```

`NewAgent` returns `agnt5.ErrAgentModelRequired` without a model. `Run` returns
`agnt5.ErrAgentMaxTurnsExceeded` when the turn budget runs out — the result is empty, partial
work is lost (stash it from tools into `ctx.Memory()` if you need it). Inside a workflow wrap
`agent.Run` in `agnt5.Step` so a restart does not re-bill the turn. No `Agent.stream`: emit
`ctx.Output(result.Response)` after the run if the caller streams; built-in providers do not
implement `agnt5.StreamingLanguageModel`. Registering the agent as a component:
`agnt5.RegisterAgent(worker, agent)` (input `{"message": "..."}`).

## Custom tools

```go
weather, err := agnt5.NewTool("get_weather", func(c context.Context, args map[string]any) (any, error) {
    city, _ := args["city"].(string)          // JSON-decoded: strings, float64, bool, []any, map[string]any
    days := 1
    if d, ok := args["days"].(float64); ok { // never args["days"].(int)
        days = int(d)
    }
    if ctx, ok := c.(*agnt5.Context); ok {    // the agent passes the run's *agnt5.Context
        ctx.Logger().Info("weather lookup", "city", city)
    }
    return fmt.Sprintf("Sunny, 22°C in %s for %d day(s)", city, days), nil // any JSON value
},
    agnt5.WithToolDescription("Get the current weather for a city."),
    agnt5.WithToolSchema(map[string]any{ // REQUIRED: without it the model is shown no parameters
        "type": "object",
        "properties": map[string]any{
            "city": map[string]any{"type": "string", "description": "Name of the city to check."},
            "days": map[string]any{"type": "integer", "description": "Forecast days, default 1."},
        },
        "required": []string{"city"},
    }),
    // agnt5.WithToolRecoveryPolicy(agnt5.RecoveryPolicyIdempotentRetry), // default RecoveryPolicyUnknownOutcome
)
```

- Handler type `agnt5.ToolHandler = func(context.Context, map[string]any) (any, error)`.
- A returned `error` is fed back to the model as `Error: ...` and the loop continues (except a
  HITL waiting error, which aborts the run to pause it — `agnt5-human-in-the-loop`).
- Other options: `WithToolMetadata`, `WithoutDurableToolActivation` (skip the durable tool
  activation record). Recovery policies: `IdempotentRetry`, `DurableSteps`, `UnknownOutcome`,
  `Compensate`, `Fail`.
- `agnt5.RegisterTool(worker, tool)` publishes the schema and makes the tool runnable on its
  own (`agnt5 run get_weather --type tool`); tools attached only via `WithAgentTools` are not
  registered.

## Built-in tools, web search, web fetch: not in Go

No `BuiltInTool.*`, no `agnt5.tools.web_search/web_fetch`, no Brave/Tavily/SearXNG helpers.
Write a tool that calls the API (the `hitl_deep_research` template has `fetch_webpage_tool` and
`wikipedia_search_tool` with SSRF guards) or wrap an MCP server.

## MCP tools

```go
transport, err := agnt5.NewStdioMCPTransport(ctx, "npx", "-y", "@modelcontextprotocol/server-filesystem", "/data")
// remote: agnt5.NewSSEMCPTransport("https://mcp.example.com/sse", map[string]string{"Authorization": "Bearer " + key})
// raw newline JSON-RPC over any io.ReadWriter: agnt5.NewStreamMCPTransport(rw)
mcp, err := agnt5.NewMCPClient(transport) // one transport per client
defer mcp.Close()

listed, err := mcp.ListTools(ctx)          // performs initialize/initialized on first use
tools := make([]agnt5.Tool, 0, len(listed))
for _, t := range listed {
    t := t
    tool, err := agnt5.NewTool(t.Name, func(c context.Context, args map[string]any) (any, error) {
        res, err := mcp.CallTool(c, t.Name, args) // agnt5.MCPCallToolResult{Content []map[string]any, IsError, Raw}
        if err != nil {
            return nil, err
        }
        if res.IsError {
            return nil, fmt.Errorf("%s failed: %v", t.Name, res.Content)
        }
        return res.Content, nil
    }, agnt5.WithToolDescription(t.Description), agnt5.WithToolSchema(t.InputSchema))
    if err != nil { return err }
    tools = append(tools, tool)
}
agent, err := agnt5.NewAgent("repository_researcher", agnt5.WithAgentModel(model),
    agnt5.WithAgentInstructions("..."), agnt5.WithAgentTools(tools...))
```

Not in Go: Streamable-HTTP transport (only SSE and stdio; `NewStreamMCPTransport` is not HTTP),
multi-server `MCPClient` (`add_*_server`, `call_tool_auto`), `MCPServer` (cannot expose AGNT5
components as an MCP server). `agnt5.NewStdioMCPTransportConfig(ctx, agnt5.ServerConfig{Command, Args, Env})`
adds env vars for the subprocess.

## Sandboxed code execution

```go
sandbox := agnt5.NewHTTPSandbox(os.Getenv("SANDBOX_ENDPOINT"), os.Getenv("SANDBOX_API_KEY")) // AGNT5 sandbox HTTP protocol
coder, err := agnt5.NewAgent("coder", agnt5.WithAgentModel(model),
    agnt5.WithAgentInstructions("Use the sandbox workspace to write files and run code before answering."),
    agnt5.WithAgentSandbox(sandbox)) // adds sandbox_execute_code, sandbox_write_file, sandbox_read_file, sandbox_list_files
```

`agnt5.SandboxRunner` interface: `ExecuteCode(ctx, language, code)`, `RunCommand(ctx, []string)`,
`WriteFile`, `ReadFile`, `ListFiles` (+ optional `DeleteFile`). Implement it yourself for E2B,
Daytona etc. — there is no provider registry, `Sandbox(provider=...)`, `template`, `env`,
`cpu_cores`, `memory_mib`, `timeout_secs` or `auto_destroy` (the `coding_agent` template talks to
E2B's HTTP API directly). `agnt5.NewInMemorySandbox()` stores files, executes nothing (tests).
Custom tools reach the workspace through `ctx.Sandbox()` (set by `WithAgentSandbox` during the
run, or `ctx.SetSandbox(runner)`); `agnt5.SandboxTools(nil)` returns the four tools resolving the
sandbox from the context at call time.

## Multi-agent patterns

**Agents as tools** — not automatic; wrap `Run`:

```go
func agentAsTool(specialist *agnt5.Agent, description string) (agnt5.Tool, error) {
    return agnt5.NewTool("ask_"+specialist.Name, func(c context.Context, args map[string]any) (any, error) {
        ctx, ok := c.(*agnt5.Context)
        if !ok {
            return nil, errors.New("agent tools need an AGNT5 context")
        }
        message, _ := args["message"].(string)
        res, err := specialist.Run(ctx, agnt5.AgentInput{Message: message})
        if err != nil {
            return nil, err
        }
        return res.Response, nil
    }, agnt5.WithToolDescription(description), agnt5.WithToolSchema(map[string]any{
        "type":       "object",
        "properties": map[string]any{"message": map[string]any{"type": "string", "description": "The task for the specialist."}},
        "required":   []string{"message"},
    }))
}
```

**Handoffs**:

```go
billing, err := agnt5.NewHandoff(billingAgent,
    agnt5.WithHandoffDescription("Transfer for billing/payment questions"), // default "Transfer to <name>"
    agnt5.WithHandoffToolName("transfer_to_billing"),                       // default transfer_to_<slug of name>
    agnt5.WithHandoffFullHistory(true),                                     // default false (Python: True)
    agnt5.WithHandoffJoinPolicy(agnt5.ChildJoinPolicyRequired),             // or ChildJoinPolicyDetached
)
triage, err := agnt5.NewAgent("triage", agnt5.WithAgentModel(model),
    agnt5.WithAgentInstructions("Route the user to the right specialist."),
    agnt5.WithAgentHandoffs(billing, technical))
res, err := triage.Run(ctx, agnt5.AgentInput{Message: "My payment failed but I was still charged."})
res.HandoffTo // "billing"
```

Same decision table as the SKILL.md. There is no bare `handoffs=[agent]` form; always
`NewHandoff`.

## Callbacks (guardrails): wrap instead

No `before_*`/`after_*` hooks. `agnt5.Tool` is a plain struct, so wrap `Handler`; the model is an
interface, so decorate it:

```go
func guarded(tool agnt5.Tool, blocked map[string]bool) agnt5.Tool { // before/after_tool_callback
    inner := tool.Handler
    tool.Handler = func(c context.Context, args map[string]any) (any, error) {
        if blocked[tool.Name] {
            return map[string]any{"error": tool.Name + " is not allowed"}, nil // tool not executed
        }
        return inner(c, args)
    }
    return tool
}

type capped struct { // before_model_callback: also the only way to set max_tokens/temperature for agents
    agnt5.LanguageModel
    maxTokens int
}

func (m capped) Generate(ctx context.Context, req agnt5.GenerateRequest) (agnt5.GenerateResponse, error) {
    if req.MaxTokens == nil {
        n := m.maxTokens
        req.MaxTokens = &n
    }
    return m.LanguageModel.Generate(ctx, req) // inspect/redact the response here for after_model
}
```

Use `capped{LanguageModel: anthropic, maxTokens: 4096}` with `WithAgentModel` — the Anthropic
provider otherwise sends `max_tokens: 1024` on every agent turn.

## Memory

```go
kv := ctx.Memory().KV(agnt5.MemoryScopeSession)          // Run | Session | User | Global
err := kv.Set(ctx, "theme", "dark"); v, err := kv.Get(ctx, "theme") // (any, error); Delete, List
doc, err := ctx.Memory().Working().Get(ctx)              // one string document per session; Set(ctx, s)
conv := ctx.Memory().Conversation()                      // session chat history
err = conv.Append(ctx, agnt5.MemoryMessage{Role: "user", Content: msg})
history, err := conv.Messages(ctx)                       // []agnt5.MemoryMessage → build agnt5.AgentInput{Messages: ...}
```

Session and user namespaces come from the run's `session_id`/`user_id` metadata
(`X-Session-ID`/`X-User-ID`: `client.Session(id).WithUser(uid)`, `agnt5.WithRunSessionID`).
Without them, session/user memory silently falls back to the run ID — i.e. per-run. The
`weather-agent` template shows replay-safe conversation recording. No semantic memory
(`ctx.memory.user.save/search`) and no `MemoryResult` search API in Go.

## Not available in Go

Built-in provider tools, `web_search`/`web_fetch`, callbacks, `MCPServer`, Streamable-HTTP MCP,
sandbox providers/auto-detect, `Agent.stream`, agent `temperature`/`max_tokens`/`top_p`,
structured output, semantic/user memory search, auto-wrapping agents passed as tools, bare
agents in `handoffs`, `confirmation=`/`durable=` tool flags.

## Go pitfalls for this skill

- Missing `WithToolSchema`: the model sees a parameterless tool and calls it with `{}`.
- `args["n"].(int)` panics/misses — JSON numbers are `float64`.
- Anthropic agents truncate at 1024 output tokens (no agent-level `MaxTokens`); wrap the model.
- gpt-6 tool-using agents fail over Chat Completions (`reasoning_effort` cannot be set), and
  `Temperature`/`MaxTokens` sent to reasoning models are rejected. Use
  `gpt-4o-mini`/`gpt-4.1-mini`, or an `OpenAIConfig.HTTPClient` transport that rewrites the
  request body.
- `ErrAgentMaxTurnsExceeded` discards the transcript; `WithAgentMaxTurns(0)` means 10, not 0.
- Session memory without a session ID is per-run.
