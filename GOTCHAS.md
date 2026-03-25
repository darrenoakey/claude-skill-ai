# daz-agent-sdk — Gotchas & Known Pitfalls

## Anthropic OAuth
- Tokens (sk-ant-oat01-*) require `anthropic-beta: oauth-2025-04-20` header. Refresh via `POST https://platform.claude.com/v1/oauth/token` with `{"grant_type":"refresh_token","refresh_token":"<rt>","client_id":"9d1c250a-e61b-44d9-88ed-5944d1962f5e"}`. Rate limit timestamps are Unix epoch integers.

## claude_agent_sdk Inside Claude Code
- `ClaudeAgentOptions.env` CANNOT unset env vars (SDK merges `{**os.environ, **options.env}`). Pop `CLAUDECODE` from `os.environ` before connecting, restore in finally.
- `claude_agent_sdk` inside Claude Code produces `tool_use ids must be unique` errors — fundamental incompatibility. Use Go MCP binary over stdio JSON-RPC instead.
- Streaming mode: `stream_input()` closes stdin immediately when no hooks/MCP servers configured. Workaround: pass dummy hooks or use `claude --print` subprocess.
- `query()` yields 5 message types. Final answer is in `ResultMessage.result` — NOT in `AssistantMessage` TextBlocks.

## daz_agent_sdk
- `agent.ask()` prompts saying "Visit this URL" trigger tool calls instead of text blocks → `response.text == ""`. Phrase prompts to avoid tool use.
- Pydantic `schema=` parameter: response has `.parsed` attribute. `resp.text` may be empty — always use `resp.parsed`.
- `schema=` with Claude provider triggers agentic tool use (file writing) which is slow and with `max_turns=1` returns empty text. For autonomous loops or high-throughput structured output: skip `schema=`, append JSON schema instructions to the prompt manually, and parse JSON from `result.text` using `parse_json_from_llm()` from `daz_agent_sdk.types`. This is 5-10x faster and avoids the agentic overhead.
- `transparent=True` with spark provider (the default) submits a `background-remove` job to arbiter on spark:8400 (BiRefNet on GPU). With mflux provider, BiRefNet runs locally on CPU — avoid for batch generation (60%+ CPU). Arbiter returns result in `result.data` not `result.image`.
