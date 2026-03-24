# daz-agent-sdk — Usage Examples

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
