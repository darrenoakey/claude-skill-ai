![](banner.jpg)

# Claude Agent SDK - Programmatic AI Integration

A skill for integrating AI capabilities into tools and scripts using the Claude Agent SDK. Build intelligent automation without managing API keys or implementing complex tool loops.

## Purpose

This skill provides standards and patterns for adding AI-powered decision making, content generation, classification, and data transformation to your tools. The SDK uses ambient authentication from Claude Code, eliminating the need for API key management.

**Use this skill whenever you need:**
- Tools that make AI-powered decisions
- Automated content or code generation
- Text classification or categorization
- Intelligent data transformation
- Natural language to structured output conversion

## Installation

```bash
pip install claude-agent-sdk
```

**Requirements:** Python 3.10+

No API key configuration needed - authentication is handled automatically through Claude Code.

## Usage

### Simple Text Query

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, AssistantMessage, TextBlock


async def ask_claude(prompt: str) -> str:
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
    result = await ask_claude("What is the capital of France?")
    print(result)


if __name__ == "__main__":
    asyncio.run(main())
```

### Structured JSON Response

```python
import asyncio
import json
from claude_agent_sdk import query, ClaudeAgentOptions, AssistantMessage, TextBlock


async def classify_sentiment(text: str) -> dict:
    prompt = f"""Classify the sentiment of this text.

Return ONLY valid JSON with:
- "sentiment": one of ["positive", "neutral", "negative"]
- "confidence": float between 0 and 1

Text: {text}

Return JSON only, no explanation."""

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

    # Handle markdown code blocks
    text = response_text.strip()
    if text.startswith("```"):
        lines = text.split("\n")[1:]
        if lines and lines[-1].strip() == "```":
            lines = lines[:-1]
        text = "\n".join(lines)

    return json.loads(text)


async def main():
    result = await classify_sentiment("I love this product!")
    print(json.dumps(result, indent=2))


if __name__ == "__main__":
    asyncio.run(main())
```

### With File System Tools

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions


async def find_todos():
    async for message in query(
        prompt="Find all TODO comments in this project and summarize them",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Glob", "Grep"],
            permission_mode="bypassPermissions"
        )
    ):
        if hasattr(message, "result"):
            print(message.result)


if __name__ == "__main__":
    asyncio.run(find_todos())
```

## Examples

### Generate Git Commit Messages

```python
import asyncio
import subprocess
from claude_agent_sdk import query, ClaudeAgentOptions, AssistantMessage, TextBlock


def get_staged_diff() -> str:
    result = subprocess.run(['git', 'diff', '--cached'], capture_output=True, text=True)
    return result.stdout


async def generate_commit_message() -> str:
    diff = get_staged_diff()
    if not diff:
        return "No staged changes"

    prompt = f"""Write a git commit message for this diff.
Use imperative mood. Be concise. Return ONLY the message.

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


if __name__ == "__main__":
    print(asyncio.run(generate_commit_message()))
```

### Code Review Assistant

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, AssistantMessage, TextBlock


async def review_file(filepath: str) -> str:
    prompt = f"""Review the code in {filepath} for:
- Potential bugs
- Security issues
- Performance concerns

Provide specific, actionable feedback."""

    response = ""
    async for message in query(
        prompt=prompt,
        options=ClaudeAgentOptions(
            allowed_tools=["Read"],
            permission_mode="bypassPermissions"
        )
    ):
        if isinstance(message, AssistantMessage):
            for block in message.content:
                if isinstance(block, TextBlock):
                    response += block.text

    return response.strip()


if __name__ == "__main__":
    import sys
    filepath = sys.argv[1] if len(sys.argv) > 1 else "main.py"
    print(asyncio.run(review_file(filepath)))
```

### Natural Language Calculator

```python
import asyncio
import json
import math
from claude_agent_sdk import query, ClaudeAgentOptions, AssistantMessage, TextBlock


async def calculate(expression: str) -> str:
    prompt = f"""Convert this to a Python math expression.

Return ONLY JSON: {{"python": "expression_here"}}

Input: {expression}"""

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
        text = "\n".join(text.split("\n")[1:-1])

    expr = json.loads(text)["python"]
    result = eval(expr, {"__builtins__": {}, "math": math, "pi": math.pi, "e": math.e})
    return str(result)


if __name__ == "__main__":
    import sys
    expr = " ".join(sys.argv[1:]) or "square root of 256"
    print(asyncio.run(calculate(expr)))
```

## Available Tools

| Tool | Description |
|------|-------------|
| `Read` | Read file contents |
| `Write` | Create new files |
| `Edit` | Modify existing files |
| `Bash` | Run terminal commands |
| `Glob` | Find files by pattern |
| `Grep` | Search file contents |
| `WebSearch` | Search the web |
| `WebFetch` | Fetch web pages |

Grant only the tools your script needs. For pure text processing, use `allowed_tools=[]`.

## Permission Modes

| Mode | Description |
|------|-------------|
| `bypassPermissions` | Auto-approve all tool use (for scripts) |
| `acceptEdits` | Auto-approve file edits only |
| `default` | Prompt for each tool use |
| `plan` | No tool execution (planning only) |

## Quick Reference

Minimal imports:
```python
import asyncio
import json
from claude_agent_sdk import query, ClaudeAgentOptions, AssistantMessage, TextBlock
```

Minimal query function:
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

## License

This project is licensed under [CC BY-NC 4.0](https://darren-static.waft.dev/license) - free to use and modify, but no commercial use without permission.
