---
name: ai
description: Claude Agent SDK integration for programmatic AI. Use when building tools that need AI to make decisions, generate content, summarize, classify, or transform data. ALWAYS use this skill when building anything that uses "claude" or "ai" - NEVER use the raw Claude API.
---

# Claude Agent SDK - Programmatic AI Integration

This skill provides standards and patterns for integrating AI capabilities into tools and scripts using the **Claude Agent SDK**. The SDK uses ambient authentication from Claude Code - no API keys required.

## CRITICAL: When This Skill Applies

**Whenever you see "claude", "ai", "llm", or need programmatic AI capabilities, READ THIS SKILL FIRST.**

This includes:
- Building tools that need AI decisions
- Generating summaries or content
- Classification or categorization
- Data transformation with intelligence
- Natural language to structured output
- Any programmatic Claude integration

## Claude Agent SDK vs Claude API (NEVER USE THE API)

| Aspect | Claude Agent SDK (USE THIS) | Claude API (NEVER USE) |
|--------|----------------------------|------------------------|
| **Authentication** | Ambient - uses Claude Code auth | Requires API key |
| **Tool handling** | Automatic - Claude handles tools | Manual - you implement loop |
| **Setup** | `pip install claude-agent-sdk` | Different package |
| **Use case** | Autonomous agents, programmatic AI | Direct API calls |

**The SDK is the ONLY approved method for programmatic Claude integration.**

---

## Installation

```bash
pip install claude-agent-sdk
```

**Requirements:** Python 3.10+

The Claude Code CLI authentication is used automatically - no API key configuration needed.

---

## Core Pattern: Simple Query

Use this pattern when you need Claude to analyze input and return a text response.

```python
#!/usr/bin/env python3

import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, AssistantMessage, TextBlock


async def ask_claude(prompt: str) -> str:
    """Query Claude and return the text response."""
    response_text = ""

    async for message in query(
        prompt=prompt,
        options=ClaudeAgentOptions(
            allowed_tools=[],
            permission_mode="bypassPermissions"
        )
    ):
        if isinstance(message, AssistantMessage):
            for block in message.content:
                if isinstance(block, TextBlock):
                    response_text += block.text

    return response_text.strip()


async def main():
    result = await ask_claude("What is 2 + 2?")
    print(result)


if __name__ == "__main__":
    asyncio.run(main())
```

---

## Core Pattern: JSON Response

Use this pattern when you need structured output. Claude returns JSON that you parse.

```python
#!/usr/bin/env python3

import asyncio
import json
from claude_agent_sdk import query, ClaudeAgentOptions, AssistantMessage, TextBlock


async def get_structured_response(prompt: str) -> dict:
    """Query Claude for JSON and parse the response."""
    response_text = ""

    async for message in query(
        prompt=prompt,
        options=ClaudeAgentOptions(
            allowed_tools=[],
            permission_mode="bypassPermissions"
        )
    ):
        if isinstance(message, AssistantMessage):
            for block in message.content:
                if isinstance(block, TextBlock):
                    response_text += block.text

    # Handle markdown code blocks in response
    text = response_text.strip()
    if text.startswith("```"):
        lines = text.split("\n")
        lines = lines[1:]  # Remove opening ```json or ```
        if lines and lines[-1].strip() == "```":
            lines = lines[:-1]  # Remove closing ```
        text = "\n".join(lines)

    return json.loads(text)


async def classify_text(text: str) -> dict:
    """Classify text into categories."""
    prompt = f"""Classify the following text.

Return ONLY valid JSON with these fields:
- "category": one of ["question", "statement", "command", "greeting"]
- "sentiment": one of ["positive", "neutral", "negative"]
- "confidence": float between 0 and 1

Text: {text}

Return JSON only, no explanation."""

    return await get_structured_response(prompt)


async def main():
    result = await classify_text("Hello, how are you today?")
    print(json.dumps(result, indent=2))


if __name__ == "__main__":
    asyncio.run(main())
```

---

## Core Pattern: With Tools

Use this pattern when Claude needs to read files, run commands, or interact with the system.

```python
#!/usr/bin/env python3

import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions


async def analyze_codebase(question: str) -> None:
    """Let Claude analyze the codebase to answer a question."""
    async for message in query(
        prompt=question,
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Glob", "Grep"],
            permission_mode="bypassPermissions"
        )
    ):
        if hasattr(message, "result"):
            print(message.result)


async def main():
    await analyze_codebase("Find all TODO comments in this project")


if __name__ == "__main__":
    asyncio.run(main())
```

---

## Available Tools

| Tool | Description | When to Allow |
|------|-------------|---------------|
| **Read** | Read any file | Analysis, code review |
| **Write** | Create new files | Code generation |
| **Edit** | Modify existing files | Refactoring, fixes |
| **Bash** | Run terminal commands | Build, test, deploy |
| **Glob** | Find files by pattern | Codebase exploration |
| **Grep** | Search file contents | Code search |
| **WebSearch** | Search the web | Current information |
| **WebFetch** | Fetch web pages | Documentation, APIs |

**Principle:** Grant minimum necessary tools. For pure text processing, use `allowed_tools=[]`.

---

## Permission Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| `bypassPermissions` | All tools auto-approved | Scripts, automation |
| `acceptEdits` | File edits auto-approved | Code modification |
| `default` | Prompt for each tool use | Interactive apps |
| `plan` | No tool execution | Planning, dry-run |

**For scripts and tools, always use `bypassPermissions`** to avoid interactive prompts.

---

## Options Reference

```python
ClaudeAgentOptions(
    # Tools
    allowed_tools=["Read", "Glob"],  # Tools Claude can use
    permission_mode="bypassPermissions",  # Auto-approve tools

    # Context
    system_prompt="You are a helpful assistant",  # Custom system prompt
    cwd="/path/to/project",  # Working directory

    # Limits
    max_turns=10,  # Maximum agent turns

    # Session
    resume="session-id",  # Resume previous session
)
```

---

## Real-World Examples

### Commit Message Generator

```python
#!/usr/bin/env python3

import asyncio
import subprocess
from claude_agent_sdk import query, ClaudeAgentOptions, AssistantMessage, TextBlock


def get_git_diff() -> str:
    result = subprocess.run(['git', 'diff', '--cached'], capture_output=True, text=True)
    return result.stdout or subprocess.run(['git', 'diff'], capture_output=True, text=True).stdout


async def generate_commit_message(diff: str) -> str:
    prompt = f"""Write a clear, concise git commit message for this diff.

Requirements:
- Use imperative mood (e.g., "Add feature", "Fix bug")
- Be concise but descriptive
- Focus on what and why, not how
- Return ONLY the commit message, nothing else

Diff:
{diff[:10000]}"""

    response = ""
    async for message in query(
        prompt=prompt,
        options=ClaudeAgentOptions(allowed_tools=[], permission_mode="bypassPermissions")
    ):
        if isinstance(message, AssistantMessage):
            for block in message.content:
                if isinstance(block, TextBlock):
                    response += block.text

    return response.strip()


async def main():
    diff = get_git_diff()
    if not diff:
        print("No changes to commit")
        return

    message = await generate_commit_message(diff)
    print(message)


if __name__ == "__main__":
    asyncio.run(main())
```

### Safe Calculator (Natural Language to Python)

```python
#!/usr/bin/env python3

import asyncio
import ast
import json
from claude_agent_sdk import query, ClaudeAgentOptions, AssistantMessage, TextBlock

ALLOWED_IMPORTS = {"math", "fractions", "decimal", "statistics"}


async def translate_to_python(expression: str) -> dict:
    prompt = f"""Convert this math request into a Python expression.

Return ONLY JSON with:
- "python": string (single expression, no statements)
- "imports": list of modules from {list(ALLOWED_IMPORTS)}

User request: {expression}"""

    response = ""
    async for message in query(
        prompt=prompt,
        options=ClaudeAgentOptions(allowed_tools=[], permission_mode="bypassPermissions")
    ):
        if isinstance(message, AssistantMessage):
            for block in message.content:
                if isinstance(block, TextBlock):
                    response += block.text

    text = response.strip()
    if text.startswith("```"):
        lines = text.split("\n")[1:]
        if lines and lines[-1].strip() == "```":
            lines = lines[:-1]
        text = "\n".join(lines)

    return json.loads(text)


def safe_eval(expr: str, imports: list) -> any:
    # Build safe namespace
    ns = {"__builtins__": {}}
    if "math" in imports:
        import math
        ns["math"] = math
        ns.update({"pi": math.pi, "e": math.e})

    # Validate and evaluate
    ast.parse(expr, mode="eval")
    return eval(expr, ns, {})


async def calculate(expression: str) -> str:
    plan = await translate_to_python(expression)
    result = safe_eval(plan["python"], plan.get("imports", []))
    return str(result)


async def main():
    import sys
    expr = " ".join(sys.argv[1:]) if len(sys.argv) > 1 else "square root of 144"
    print(await calculate(expr))


if __name__ == "__main__":
    asyncio.run(main())
```

### Script Generator

```python
#!/usr/bin/env python3

import asyncio
import json
import os
from claude_agent_sdk import query, ClaudeAgentOptions, AssistantMessage, TextBlock


async def generate_script(name: str, description: str) -> dict:
    prompt = f"""Generate a script named {name}.

Description: {description}

Requirements:
- Use Bash unless Python is significantly better
- Always start with a shebang
- Keep it well-commented but concise

Return ONLY valid JSON with:
- "script": the complete script text
- "documentation": markdown documentation
- "reply": message to show the user

Return JSON only."""

    response = ""
    async for message in query(
        prompt=prompt,
        options=ClaudeAgentOptions(allowed_tools=[], permission_mode="bypassPermissions")
    ):
        if isinstance(message, AssistantMessage):
            for block in message.content:
                if isinstance(block, TextBlock):
                    response += block.text

    text = response.strip()
    if text.startswith("```"):
        lines = text.split("\n")[1:]
        if lines and lines[-1].strip() == "```":
            lines = lines[:-1]
        text = "\n".join(lines)

    return json.loads(text)


async def main():
    import sys
    if len(sys.argv) < 3:
        print("Usage: create_script <name> <description>")
        return

    name = sys.argv[1]
    description = " ".join(sys.argv[2:])

    result = await generate_script(name, description)

    path = os.path.expanduser(f"~/bin/{name}")
    with open(path, "w") as f:
        f.write(result["script"])
    os.chmod(path, 0o755)

    print(f"Created: {path}")
    print(result["reply"])


if __name__ == "__main__":
    asyncio.run(main())
```

---

## Error Handling

```python
from claude_agent_sdk import (
    ClaudeSDKError,      # Base error
    CLINotFoundError,    # Claude Code not installed
    CLIConnectionError,  # Connection issues
    ProcessError,        # Process failed
    CLIJSONDecodeError,  # JSON parsing issues
)

try:
    async for message in query(prompt="Hello"):
        pass
except CLINotFoundError:
    print("Error: Claude Code CLI not found. Install with: curl -fsSL https://claude.ai/install.sh | bash")
except ProcessError as e:
    print(f"Process failed with exit code: {e.exit_code}")
except CLIJSONDecodeError as e:
    print(f"Failed to parse response: {e}")
```

---

## Best Practices

### 1. Always Strip and Validate JSON Responses

Claude may wrap JSON in markdown code blocks. Always handle this:

```python
def parse_json_response(text: str) -> dict:
    text = text.strip()
    if text.startswith("```"):
        lines = text.split("\n")[1:]
        if lines and lines[-1].strip() == "```":
            lines = lines[:-1]
        text = "\n".join(lines)
    return json.loads(text)
```

### 2. Use Specific Prompts for Structured Output

Be explicit about the exact JSON structure you expect:

```python
prompt = """Return ONLY valid JSON with these exact fields:
- "answer": string
- "confidence": float between 0 and 1
- "reasoning": string

No other text, just JSON."""
```

### 3. Limit Input Size

Truncate large inputs to avoid token limits:

```python
prompt = f"Analyze this code:\n{code[:10000]}"
```

### 4. Use Minimal Tools

Only grant tools that are actually needed:

```python
# Good: minimal permissions
options = ClaudeAgentOptions(allowed_tools=[], permission_mode="bypassPermissions")

# Only when file access is needed
options = ClaudeAgentOptions(allowed_tools=["Read", "Glob"], permission_mode="bypassPermissions")
```

### 5. Handle Empty Responses

Always check for empty responses:

```python
if not response.strip():
    raise ValueError("Claude returned empty response")
```

---

## Imports Reference

Standard imports for Claude Agent SDK usage:

```python
import asyncio
import json
from claude_agent_sdk import query, ClaudeAgentOptions, AssistantMessage, TextBlock
```

For error handling:

```python
from claude_agent_sdk import (
    ClaudeSDKError,
    CLINotFoundError,
    CLIConnectionError,
    ProcessError,
    CLIJSONDecodeError,
)
```

---

## Quick Copy Templates

### Minimal Text Query

```python
async def ask(prompt: str) -> str:
    r = ""
    async for m in query(prompt=prompt, options=ClaudeAgentOptions(allowed_tools=[], permission_mode="bypassPermissions")):
        if isinstance(m, AssistantMessage):
            for b in m.content:
                if isinstance(b, TextBlock):
                    r += b.text
    return r.strip()
```

### Minimal JSON Query

```python
async def ask_json(prompt: str) -> dict:
    r = await ask(prompt)
    if r.startswith("```"):
        r = "\n".join(r.split("\n")[1:-1])
    return json.loads(r)
```

---

## Existing Tools Using This Pattern

Reference these in `~/bin` for real-world examples:
- `commit` - Commit message generation
- `create_script` - Script generation with JSON output
- `calc` - Natural language calculator
- `convert` - File format conversion planning
- `fill-out-mr` - MR description generation
- `publish-skill` - README and banner generation

---

## Do NOT Use

- **Claude API directly** - Always use the Agent SDK
- **API keys** - The SDK uses ambient authentication
- **Manual tool loops** - The SDK handles this automatically
- **Other LLM libraries** - Use the Agent SDK for Claude integration
