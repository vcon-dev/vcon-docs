---
icon: arrow-progress
description: The Conserver — a Redis-backed conversation processing engine that ingests vCons, runs them through configurable chains of links, and writes the results to any of fourteen storage backends.
---

# Conserver

The Conserver is the runtime that turns vCons into something useful. It pulls vCons off Redis ingress queues, runs them through a configurable pipeline of **links** (transcribe → analyze → tag → notify → record audit trail → …), and writes the finished result to one or more **storages** (Postgres, S3, MongoDB, Elasticsearch, Milvus, SCITT transparency services, the vCon MCP server, and more).

It's an [open-source project](https://github.com/vcon-dev/vcon-server) — Python 3.12, FastAPI for the API tier, Redis for queuing, Docker Compose for deployment. The current build ships **22 standard links** and **14 storage backends**, plus tracers that emit a verifiable audit trail of every chain execution.

## When to use the Conserver

- You have vCons arriving from one or more adapters (phone systems, SIPREC, email, chat, LLM exports) and need to do something with them at scale.
- You need a transcribe → analyze → store pipeline that runs reliably, scales horizontally, and handles failures via dead-letter queues.
- You want a single integration point for downstream systems (CRM, data warehouse, MCP server, blockchain audit log) so adapter teams don't each build their own.
- You need to track lifecycle events — creation, enhancement, deletion, consent revocation — on a [SCITT transparency ledger](../extensions/lifecycle.md).

## Documentation layout

**Get started:**

- [Conserver Introduction](conserver-introduction.md) — overview and design rationale
- [Quick Start](conserver-quick-start.md) — Docker Compose in fifteen minutes
- [Concepts](concepts.md) — Link, Chain, Storage, Tracer

**Configure:**

- [Configuring the Conserver](configuring-the-conserver.md) — every env var and every YAML section
- [Standard Links](standard-links.md) — reference for all 22 shipped links
- [Storage](storage.md) — reference for all 14 storage backends
- [Conserver Tracers](conserver-tracers.md) — audit and compliance trail

**Build:**

- [Creating Custom Links](creating-custom-links.md) — write your own processing step
- [API](api.md) — REST endpoints for vCon CRUD, ingress / egress, configuration, DLQ
- [Integrating Your App](integrating-your-app.md) — calling the API from your code

**Operate:**

- [Production Deployment](production-deployment.md) — Docker Compose, scaling, secrets, observability
- [Inside the Conserver](inside-the-conserver.md) — architecture and request flow
- [Day in the Life of a vCon](day-in-the-life-of-a-vcon.md) — narrative walkthrough
- [Troubleshooting](troubleshooting.md) — common issues and their fixes
- [Operational Benefits](operational-benefits-of-conservers.md) — federation, multi-tenancy, governance

**Source and support:**

- [GitHub: vcon-dev/vcon-server](https://github.com/vcon-dev/vcon-server) — the canonical repository
