---
description: A Streamlit toolkit for conserver developers, testers and operators to move vCons between stores, inspect them and run QA analysis.
---

# 🗂️ vCon Admin

**Repo:** [vcon-dev/vcon-admin](https://github.com/vcon-dev/vcon-admin)

vcon-admin is a Streamlit application for people who build, test and run a [conserver](../conserver/README.md). It uses MongoDB as its own data store and runs in Docker.

## What it does

- Imports and exports vCons between Redis (the conserver's native store), S3, JSONL, JSON and MongoDB.
- Shows the status and configuration of the local system and its Docker containers, with live Docker logs.
- Uploads vCons to Elasticsearch, and to Milvus or an OpenAI vector store, for QA testing and semantic search.
- Inspects one vCon by UUID: summary, analysis, dialog, parties and attachments, with a JSON download.
- Runs a workbench that sends configured OpenAI prompts (full vCon, summary or transcript) over a chosen set of vCons, and a ChatGPT section for file upload and assistant chat.
- Visualizes vCon data and embeddings.

## Install

The README describes a Docker install tested on Ubuntu 24.04:

1. Clone the repo and create `.streamlit/secrets.toml`.
2. Run `docker network create conserver`, then `docker compose up -d`.
3. Reset the Elasticsearch password with `docker exec -it elasticsearch /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic` and put it in `secrets.toml`.
4. Open `http://localhost:8501`.

`secrets.toml` holds sections for AWS (S3 import and export), MongoDB, OpenAI, Elasticsearch and the conserver API URL and token. For Milvus, start `docker-compose-milvus.yml` as well. To run without Docker, use `poetry install` then `poetry run streamlit run admin.py`.

Do not expose port 8501 on a public host without a firewall rule.

## When not to use it

- For LLM-driven queries, use the [vCon MCP Server](../mcp-server/README.md).
- For pipelines (transcribe, analyze, store), use the [conserver](../conserver/README.md).

## See also

- [Conserver](../conserver/README.md)
- [Conserver Storage](../conserver/storage.md)
