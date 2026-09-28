# Model providers

Model strings are `"<provider>/<model-name>"`, e.g. `openai/gpt-4o-mini` (the template default),
`anthropic/claude-sonnet-5`, `google/gemini-2.5-pro`. Model names change often — use the one the
user asks for, and check the provider's docs when unsure instead of guessing.

Providers with first-class Studio credentials: `openai`, `anthropic`, `google` (or `gemini`),
`groq`, `openrouter`, `mistral`, `deepseek`, `xai`. The SDK also accepts `azure`, `bedrock`,
`fireworks`, `together`, `huggingface` (or `hf`), and `ollama`. Current table:
https://agnt5.com/docs/integrations/ai-providers

## `.env.example` key per provider

| Provider | Env var(s) |
|---|---|
| OpenAI | `OPENAI_API_KEY` |
| Anthropic | `ANTHROPIC_API_KEY` |
| Google / Gemini | `GOOGLE_API_KEY` |
| Groq | `GROQ_API_KEY` |
| OpenRouter | `OPENROUTER_API_KEY` |
| Mistral | `MISTRAL_API_KEY` |
| DeepSeek | `DEEPSEEK_API_KEY` |
| xAI | `XAI_API_KEY` |
| Azure OpenAI | `AZURE_OPENAI_API_KEY` + `AZURE_OPENAI_ENDPOINT` |
| Bedrock | `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` |
| Fireworks | `FIREWORKS_API_KEY` |
| Together | `TOGETHER_API_KEY` |
| Hugging Face | `HUGGINGFACE_API_KEY` |
| Ollama | none (local) |

## Wrapping OpenAI / OpenAI Agents SDK / Google ADK code

If the blueprint builds on one of these libraries, add the matching extra
(`agnt5[openai]`, `agnt5[openai-agents]`, `agnt5[google-adk]`). AGNT5 then traces those calls
automatically inside components — no manual instrumentation. Controls: `agnt5-observe`.
