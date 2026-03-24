---
name: ai
description: daz-agent-sdk — provider-agnostic AI with tier-based routing and automatic fallback. Use when building anything that needs programmatic AI (text, structured output, agentic, image, TTS, STT). NEVER use raw claude_agent_sdk or Claude API directly.
---

# daz-agent-sdk — Programmatic AI Integration

Provider-agnostic AI library with tier-based routing and automatic fallback across Claude, Gemini, Codex, and Ollama. One import, one line per query. No API keys needed for Claude (ambient auth from Claude Code).

**If writing AI-powered code**, read `EXAMPLES.md` in this directory.
**If configuring, debugging errors, or using the CLI**, read `REFERENCE.md` in this directory.
**If hitting unexpected behavior**, read `GOTCHAS.md` in this directory.

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

## Do NOT Use

- **`claude_agent_sdk` directly** — daz-agent-sdk wraps it with fallback, tiers, and a cleaner API
- **Claude API / `anthropic` SDK** — no direct API calls; use daz-agent-sdk
- **`openai` SDK directly** — use daz-agent-sdk with Codex provider
- **`google.genai` directly** — use daz-agent-sdk with Gemini provider
- **Manual JSON parsing from AI responses** — use `schema=` with Pydantic models
- **Manual markdown stripping** — structured output handles this automatically
