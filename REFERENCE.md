# daz-agent-sdk — Reference

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
