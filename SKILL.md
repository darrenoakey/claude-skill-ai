---
name: ai
description: daz-agent-sdk — provider-agnostic AI with tier-based routing and automatic fallback. Use when building anything that needs programmatic AI (text, structured output, agentic, image, TTS, STT). NEVER use raw claude_agent_sdk or Claude API directly.
---

# daz-agent-sdk — Programmatic AI Integration

Provider-agnostic AI library with tier-based routing and automatic fallback across Claude, Gemini, Codex, and Ollama. One import, one line per query. No API keys needed for Claude (ambient auth from Claude Code).

## When This Skill Applies

Any time code needs programmatic AI: text generation, structured output, classification, summarization, agentic tool use, image generation, TTS, STT. If you see `claude_agent_sdk`, `anthropic`, `openai`, or `google.genai` used for AI queries — replace with `daz_agent_sdk`.

---

## Installation

```bash
# From the project directory
cd /Volumes/T9/darrenoakey/src/daz-agent-sdk
./run install          # installs system-wide as editable package

# Or via pip
pip install -e /Volumes/T9/darrenoakey/src/daz-agent-sdk
```

Requires Python 3.11+.

### Go

```bash
go get github.com/darrenoakey/daz-agent-sdk/go
```

Full Go port with Ollama provider, image gen (Z-Image-Turbo via Ollama), TTS/STT subprocess wrappers. See `go/README.md` for usage.

---

## Imports

```python
from daz_agent_sdk import agent                          # ready-to-use singleton
from daz_agent_sdk import Agent, Conversation            # classes
from daz_agent_sdk import Tier, Capability, ErrorKind    # enums
from daz_agent_sdk import Response, StructuredResponse   # response types
from daz_agent_sdk import ImageResult, AudioResult       # capability results
from daz_agent_sdk import Message, ModelInfo, AgentError # supporting types
```

---

## Tiers

Tiers control which providers and models are used. The fallback engine cascades through the tier chain on transient errors.

| Tier | Models (in fallback order) | Use Case |
|------|---------------------------|----------|
| `Tier.VERY_HIGH` | claude-opus-4-6 → gpt-5.3-codex → gemini-2.5-pro | Critical, highest quality |
| `Tier.HIGH` | claude-opus-4-6 → gpt-5.3-codex → gemini-2.5-pro | Default — general purpose |
| `Tier.MEDIUM` | claude-sonnet-4-6 → gpt-5.3-codex → gemini-2.5-flash | Balanced speed/quality |
| `Tier.LOW` | claude-haiku-4-5 → gemini-2.5-flash-lite → ollama:qwen3-8b | Fast, cheap |
| `Tier.FREE_FAST` | ollama:qwen3-8b | Local only, zero cost |
| `Tier.FREE_THINKING` | ollama:qwen3-30b-32k → ollama:deepseek-r1:14b | Local reasoning, zero cost |

---

## Simple Ask (One-Shot)

```python
import asyncio
from daz_agent_sdk import agent, Tier

async def main():
    # Default tier (HIGH) — uses Claude, falls back to Gemini/Codex
    answer = await agent.ask("Explain quantum tunnelling in one paragraph")
    print(answer.text)

    # Cheap/fast tier
    answer = await agent.ask("Summarise this text: ...", tier=Tier.LOW)
    print(answer.text)

    # Free local model
    answer = await agent.ask("What is 2+2?", tier=Tier.FREE_FAST)
    print(answer.text)

    # With system prompt
    answer = await agent.ask(
        "Review this function for bugs",
        system="You are a senior code reviewer. Be concise.",
    )
    print(answer.text)

asyncio.run(main())
```

### Response Object

```python
answer = await agent.ask("Hello")
answer.text              # str — the response text
answer.model_used        # ModelInfo — which model actually responded
answer.model_used.provider        # "claude", "gemini", etc.
answer.model_used.qualified_name  # "claude:claude-opus-4-6"
answer.conversation_id   # UUID
answer.turn_id           # UUID
answer.usage             # dict — token counts etc.
```

---

## Structured Output (Pydantic)

Pass a Pydantic model as `schema=` to get validated, typed responses. No JSON parsing, no markdown stripping — it just works.

```python
from pydantic import BaseModel
from daz_agent_sdk import agent, Tier

class Sentiment(BaseModel):
    label: str          # "positive", "neutral", "negative"
    confidence: float   # 0.0 to 1.0
    reasoning: str

result = await agent.ask(
    "Classify: 'I absolutely love this product!'",
    schema=Sentiment,
    tier=Tier.LOW,
)
print(result.parsed.label)       # "positive"
print(result.parsed.confidence)  # 0.97
print(result.parsed.reasoning)   # "Strong positive language..."

# result.parsed is a validated Sentiment instance
# result.text still contains the raw response text
```

```python
class CodeReview(BaseModel):
    issues: list[str]
    severity: str       # "low", "medium", "high", "critical"
    suggestion: str

review = await agent.ask(
    f"Review this code:\n{code}",
    schema=CodeReview,
)
for issue in review.parsed.issues:
    print(f"- {issue}")
```

---

## Agentic Use (Tools)

Give the model tools and multiple turns to complete complex tasks autonomously.

```python
result = await agent.ask(
    "Find all TODO comments in this project and summarise them",
    tools=["Read", "Glob", "Grep"],
    max_turns=10,
    cwd="/path/to/project",
)
print(result.text)
```

Available tools: `Read`, `Write`, `Edit`, `Bash`, `Glob`, `Grep`, `WebSearch`, `WebFetch`.

Grant only what's needed. For pure text processing, omit `tools` entirely.

---

## Multi-Turn Conversations

```python
from daz_agent_sdk import agent

async with agent.conversation("code-review", system="You are a code reviewer") as chat:
    # Each call maintains full conversation history
    outline = await chat.say("Here's my PR diff:\n{diff}\nGive me an overview")
    print(outline.text)

    details = await chat.say("Now focus on security concerns")
    print(details.text)

    # Structured output works in conversations too
    summary = await chat.say("Rate the overall quality", schema=ReviewScore)
    print(summary.parsed.score)
```

### Streaming

```python
async with agent.conversation("writer") as chat:
    await chat.say("You are a technical writer")

    async for chunk in chat.stream("Write a guide to Python decorators"):
        print(chunk, end="", flush=True)
    print()  # newline after stream
```

### Forking Conversations

Fork creates an independent copy of the conversation history. Changes to the fork don't affect the original.

```python
async with agent.conversation("brainstorm") as chat:
    await chat.say("We're building a CLI tool. Give me 3 architecture options.")

    # Explore two directions independently
    fork_a = chat.fork("option-a")
    fork_b = chat.fork("option-b")

    detail_a = await fork_a.say("Expand on option 1")
    detail_b = await fork_b.say("Expand on option 2")
```

### MCP Server Integration

Pass MCP servers to give the conversation access to external tools.

```python
async with agent.conversation(
    "data-task",
    mcp_servers={
        "database": {
            "command": "/path/to/db-mcp-server",
            "args": ["--connection", "postgres://..."],
        },
        "slack": {
            "command": "/path/to/slack-mcp-server",
        },
    },
) as chat:
    response = await chat.say("Query the users table and post a summary to #reports")
```

### Conversation Properties

```python
chat.history    # list[Message] — snapshot of conversation history
chat.name       # str | None — conversation name
chat.tier       # Tier — current tier
```

---

## Image Generation

Delegates to local `generate_image` binary. Tier controls step count (quality vs speed).

```python
from daz_agent_sdk import agent, Tier

# Standard image
result = await agent.image(
    "A sunset over mountains",
    width=1024,
    height=1024,
    output="sunset.jpg",
)
print(result.path)  # Path to generated image

# Logo with transparent background (enforces PNG, runs BiRefNet removal automatically)
logo = await agent.image(
    "A minimalist fox logo, cyan on white",
    width=256,
    height=256,
    transparent=True,
    output="logo.png",
)

# Fast draft (2 steps)
draft = await agent.image("A robot", width=512, height=512, tier=Tier.LOW)

# All image() parameters:
#   prompt, width, height       — required
#   output                      — path (temp file if omitted)
#   tier                        — controls step count (HIGH=16, MEDIUM=8, LOW=2)
#   transparent                 — True → PNG + background removal
#   model, steps, image, image_strength, guidance, quantize, seed, timeout
```

Step counts by tier: VERY_HIGH=32, HIGH=16, MEDIUM=8, LOW=2.

---

## Text-to-Speech

```python
audio = await agent.speak(
    "Hello, welcome to the system",
    voice="gary",           # default voice
    output="greeting.wav",
    speed=1.0,
)
print(audio.path)            # Path to WAV file
print(audio.duration_seconds)
```

## Speech-to-Text

```python
text = await agent.transcribe(
    "recording.wav",
    model_size="small",    # "base", "small", "large-v3-turbo"
    language="en",         # optional
)
print(text)
```

---

## List Available Models

```python
from daz_agent_sdk import agent, Tier, Capability

# All models
models = await agent.models()
for m in models:
    print(f"{m.qualified_name} — {m.tier.value}, caps: {m.capabilities}")

# Filter by tier
high_models = await agent.models(tier=Tier.HIGH)

# Filter by capability
text_models = await agent.models(capability=Capability.TEXT)
```

---

## Error Handling

```python
from daz_agent_sdk import agent, AgentError, ErrorKind

try:
    answer = await agent.ask("Hello")
except AgentError as e:
    print(f"Error kind: {e.kind}")       # ErrorKind enum
    print(f"Attempts: {e.attempts}")     # list of provider attempt details

    if e.kind == ErrorKind.AUTH:
        print("Authentication failed — check API keys")
    elif e.kind == ErrorKind.RATE_LIMIT:
        print("All providers rate limited")
    elif e.kind == ErrorKind.TIMEOUT:
        print("All providers timed out")
    elif e.kind == ErrorKind.INVALID_REQUEST:
        print("Bad request — caller bug")
    elif e.kind == ErrorKind.NOT_AVAILABLE:
        print("No providers available")
    elif e.kind == ErrorKind.INTERNAL:
        print("Internal provider error")
```

Error behavior:
- `RATE_LIMIT`, `TIMEOUT`, `NOT_AVAILABLE`, `INTERNAL` → cascades to next provider in chain
- `AUTH`, `INVALID_REQUEST` → raises immediately (no cascade)
- In conversations: exponential backoff before cascade (1s, 2s, 4s... up to 60s)

---

## Bypass Tier — Use Specific Provider/Model

```python
# Force a specific provider
answer = await agent.ask("Hello", provider="gemini")

# Force a specific model
answer = await agent.ask("Hello", model="claude-sonnet-4-6")

# In conversations
async with agent.conversation("task", provider="ollama", model="qwen3-8b") as chat:
    response = await chat.say("Hello")
```

---

## CLI

```bash
# Single-shot query
daz-agent-sdk ask "What is the capital of France?"
daz-agent-sdk ask --tier low "Summarise this..."

# List models
daz-agent-sdk models
daz-agent-sdk models --tier high
```

---

## Configuration

Config file: `~/.daz-agent-sdk/config.yaml` (optional — sensible defaults work out of the box).

```yaml
tiers:
  high:
    - claude:claude-opus-4-6
    - gemini:gemini-2.5-pro
  low:
    - ollama:qwen3-8b

providers:
  gemini:
    api_key_env: GEMINI_API_KEY
  ollama:
    base_url: http://localhost:11434

image:
  model: z-image-turbo
  tiers:
    high:   { steps: 16 }
    medium: { steps: 8 }
    low:    { steps: 2 }

tts:
  voices:
    gary: { provider: local, voice_id: gary }

logging:
  directory: ~/.daz-agent-sdk/logs
  level: info
  retention_days: 30

fallback:
  single_shot:
    strategy: immediate_cascade
  conversation:
    strategy: backoff_then_cascade
    max_backoff_seconds: 60
```

---

## Quick-Copy Templates

### Minimal text query
```python
from daz_agent_sdk import agent
answer = await agent.ask("Your prompt here")
print(answer.text)
```

### Minimal structured query
```python
from pydantic import BaseModel
from daz_agent_sdk import agent

class Result(BaseModel):
    answer: str
    confidence: float

result = await agent.ask("Your prompt", schema=Result)
print(result.parsed.answer)
```

### Minimal conversation
```python
from daz_agent_sdk import agent

async with agent.conversation("task-name") as chat:
    r = await chat.say("First message")
    r = await chat.say("Follow-up")
```

### Minimal agentic
```python
from daz_agent_sdk import agent

result = await agent.ask(
    "Find and fix the bug",
    tools=["Read", "Edit", "Glob", "Grep", "Bash"],
    max_turns=15,
    cwd="/path/to/project",
)
```

---

## Do NOT Use

- **`claude_agent_sdk` directly** — daz-agent-sdk wraps it with fallback, tiers, and a cleaner API
- **Claude API / `anthropic` SDK** — no direct API calls; use daz-agent-sdk
- **`openai` SDK directly** — use daz-agent-sdk with Codex provider
- **`google.genai` directly** — use daz-agent-sdk with Gemini provider
- **Manual JSON parsing from AI responses** — use `schema=` with Pydantic models
- **Manual markdown stripping** — structured output handles this automatically

## Gotchas & Known Pitfalls

### Anthropic OAuth
- Tokens (sk-ant-oat01-*) require `anthropic-beta: oauth-2025-04-20` header. Refresh via `POST https://platform.claude.com/v1/oauth/token` with `{"grant_type":"refresh_token","refresh_token":"<rt>","client_id":"9d1c250a-e61b-44d9-88ed-5944d1962f5e"}`. Rate limit timestamps are Unix epoch integers.

### claude_agent_sdk Inside Claude Code
- `ClaudeAgentOptions.env` CANNOT unset env vars (SDK merges `{**os.environ, **options.env}`). Pop `CLAUDECODE` from `os.environ` before connecting, restore in finally.
- `claude_agent_sdk` inside Claude Code produces `tool_use ids must be unique` errors — fundamental incompatibility. Use Go MCP binary over stdio JSON-RPC instead.
- Streaming mode: `stream_input()` closes stdin immediately when no hooks/MCP servers configured. Workaround: pass dummy hooks or use `claude --print` subprocess.
- `query()` yields 5 message types. Final answer is in `ResultMessage.result` — NOT in `AssistantMessage` TextBlocks.

### daz_agent_sdk
- `agent.ask()` prompts saying "Visit this URL" trigger tool calls instead of text blocks → `response.text == ""`. Phrase prompts to avoid tool use.
- Pydantic `schema=` parameter: response has `.parsed` attribute. `resp.text` may be empty — always use `resp.parsed`.
