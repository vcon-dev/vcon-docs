---
description: Convert Claude Code session files and claude.ai conversation exports into vCons from the command line.
---

# 🤖 vCon Anthropic Chats

**Repo:** [VCONIC/vcon-anthropic-chats](https://github.com/VCONIC/vcon-anthropic-chats) · **Version:** `0.1.0` · Python 3.12+

A command-line converter that turns Claude conversations into vCons, one vCon per chat. It reads two sources, auto-detected from the input path:

- Claude Code sessions, the JSONL files under `~/.claude/projects/<encoded-cwd>/`
- The `conversations.json` file from a claude.ai account data export

It does not call the Anthropic API.

## Install and run

```bash
pip install -e .

vcon-anthropic-chats <input-path> \
    [--source auto|claude-code|claude-web] \
    [--out-dir ./out] [--post-url https://example.com/vcons] \
    [--auth-header-env VCON_AUTH] [--user-email you@example.com] \
    [--agent-id sip:claude@anthropic.com] [--overwrite] [--dry-run]
```

The input is a positional path (a file or a directory), not stdin. Give at least one of `--out-dir` or `--post-url` unless you pass `--dry-run`. `--auth-header-env` names an environment variable holding the full header value, for example `Authorization: Bearer ...`, so the token stays off the command line.

```bash
vcon-anthropic-chats ~/.claude/projects/-Users-me-Documents-GitHub-myrepo --out-dir ./vcons
```

## What the output looks like

Syntax `0.4.0`. Messages become text dialog entries between a customer party and an agent party. The rest is preserved so the conversion loses nothing:

- Tool calls and results as attachments of type `tool_use` and `tool_result`
- Thinking blocks as analysis entries of type `thinking`
- Other Claude-specific blocks as `claude.<kind>` attachments
- Session details as a `claude.session_metadata` attachment
- The original input file as a `source` attachment, base64url encoded

## Spec gaps

The converter predates the current drafts, so treat its output as a starting point.

- It does not use the [Agent Session extension](../extensions/agent-session.md): no `extensions[]` array, and the attachments above use a `type` key with the names listed, not the extension's purposes.
- It writes no `lawful_basis` attachment. Add one before storing the vCons. See [Lawful Basis](../extensions/lawful-basis.md).

## See also

- [vCon MCP Adapters](vcon-mcp-adapters.md), which converts other agent framework traces
- [vCon Adapter Development Guide](../vcon-adapters/vcon-adapter-development-guide.md)
