---
icon: server
description: Where to start with the vCon MCP Server, the open-source Model Context Protocol server that lets AI assistants query and update a store of vCons.
---

# MCP Server

The vCon MCP Server gives an AI assistant or agent access to a store of vCons over the
[Model Context Protocol](https://modelcontextprotocol.io). It runs over stdio or Streamable HTTP,
exposes 46 tools in seven groups plus resources and prompts, and stores vCons in Supabase Postgres
with optional Redis caching and pgvector semantic search.

The code is at [vcon-dev/vcon-mcp](https://github.com/vcon-dev/vcon-mcp), MIT-licensed TypeScript,
published as the npm package `vcon-mcp` and the Docker image
`public.ecr.aws/r4g1k2s3/vcon-dev/vcon-mcp`. These pages describe release 1.9.2.

## Pages in this section

* [What is the vCon MCP Server?](what-is-the-vcon-mcp-server.md): what it does, how MCP works, when to use it, how it relates to the conserver.
* [Tool Reference](tool-reference.md): every tool with its key parameters, plus resources and prompts.
* [Contract Tools](contract-tools.md): the envelope, paging, byte budgets and error codes of the LLM-facing tools.
* [Transport and Deployment](transport-and-deployment.md): transports, keys, tool profiles, tenancy, environment variables, Docker and npm.
* [How the vCon MCP Server is Built](how-the-vcon-mcp-server-is-built.md): validation, plugins, the write and read paths, tables and search.
* [Field-Name Migration](field-name-migration.md): how `critical` and `amended` replace `must_support` and `appended`.

## Elsewhere

* [Source on GitHub](https://github.com/vcon-dev/vcon-mcp)
* [mcp.conserver.io](https://mcp.conserver.io/), the generated technical reference
