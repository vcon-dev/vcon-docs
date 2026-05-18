---
description: Observability adapters for the vCon MCP server — OpenTelemetry tracing for tool calls.
---

# 📊 vCon MCP Adapters

**Repo:** [vcon-dev/vcon-mcp-adapters](https://github.com/vcon-dev/vcon-mcp-adapters) · **v0.2 released:** May 2026

A collection of observability adapters for the [vCon MCP server](../mcp-server/README.md). The flagship is an OpenTelemetry tracing integration that emits spans for every tool call, capturing:

- Tool name, arguments (with optional redaction)
- Latency, cache hits/misses
- Errors and validation failures
- The client identity (model, agent name) when the MCP transport surfaces it

## When to use it

- You're running the MCP server in production and want trace data flowing into your existing observability stack (Datadog, Honeycomb, Grafana Tempo, etc.).
- You're debugging an LLM client that's making a lot of MCP calls and want to see which ones, in what order, with what arguments.
- You're tracking model-by-model usage of vCon tools for capacity planning or cost attribution.

## Install

See the repo README. The typical deployment is a small wrapper around the MCP server's tool dispatch path that emits OTLP spans to whatever collector you're already running.

## What's new in v0.2 (May 2026)

- Spans now include the May 2026 contract-tool family (`vcon_fetch`, `vcon_search`, `vcon_capabilities`, `vcon_taxonomy`, `describe_response_shape`).
- Argument redaction is now configurable per-tool, so you can keep argument payloads out of trace storage for tools that touch personal data.
- Cache attributes (hit, miss, bypass) are now emitted for tools that have caching.

## See also

- [vCon MCP Server overview](../mcp-server/README.md)
- [Tool Reference](../mcp-server/tool-reference.md) — what the spans are tracing
