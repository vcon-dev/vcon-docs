---
description: Convert AI agent framework traces into vCons that carry an MCP session attachment, from the command line or a Python SDK.
---

# 📊 vCon MCP Adapters

**Repo:** [vcon-dev/vcon-mcp-adapters](https://github.com/vcon-dev/vcon-mcp-adapters) (default branch `claude/vcon-mcp-adapters-M1Oie`) · **Status:** pre-release, `0.1.0` in `pyproject.toml`

vcon-mcp-adapters turns traces from AI agent frameworks into vCons. It is not an OpenTelemetry tracer and does not instrument the [vCon MCP server](../mcp-server/README.md). It reads a trace after the fact, normalizes it into an `MCPSession` object (turns, tool calls, tool results, usage, artifacts), and attaches that object to a vCon as `attachments[0]` with `purpose: "mcp_session"`.

## Supported inputs

- Anthropic Messages API responses
- OpenAI Responses API calls and OpenAI Agents SDK sessions
- Claude Code session JSONL files
- Exports from Helicone, Langfuse and LangSmith, as files or fetched live from their APIs

## Install

```bash
pip install vcon-mcp-adapters
pip install "vcon-mcp-adapters[anthropic]"   # or [openai], [agents]
```

The CHANGELOG lists the Helicone, Claude Code, Langfuse and LangSmith adapters under `0.2.0` to `0.2.2`, all marked Unreleased. Check the repo before relying on them.

## Convert a trace

```bash
trace-to-vcon anthropic response.json --out session.vcon.json \
  --user-name "Alice" --assistant-name "Claude" --model claude-opus-4-7

trace-to-vcon claude-code ~/.claude/projects/<encoded-cwd>/<session>.jsonl \
  --user-name "Alice" --out cc.vcon.json

trace-to-vcon helicone helicone-export.json --user-name "Alice" --out-dir ./out/
trace-to-vcon fetch langsmith --project my-app --out-dir ./out/

trace-to-vcon validate session.vcon.json
```

Live fetches read `HELICONE_API_KEY`, the Langfuse key variables, or `LANGSMITH_API_KEY` from the environment. Export files produce one vCon per record or trace.

## Python SDK

```python
from vcon_mcp_adapters import to_vcon
from vcon_mcp_adapters.adapters.anthropic import from_trace
from vcon_mcp_adapters.parties import user_party, ai_agent_party

session = from_trace(open("messages_response.json").read())
vcon = to_vcon(session, parties=[
    user_party(name="Alice", email="alice@example.com"),
    ai_agent_party(provider="anthropic", model="claude-opus-4-7"),
])
```

## Lawful basis

The adapters emit vCons with syntax `0.4.0` and no `lawful_basis` attachment. Add one before storing or sharing the output if the conversation involves personal data. See [Lawful Basis](../extensions/lawful-basis.md).

## See also

- [vCon MCP Server overview](../mcp-server/README.md)
- [vCon Anthropic Chats](vcon-anthropic-chats.md), a narrower converter for Claude Code and claude.ai history
