---
icon: arrow-progress
description: Start here to learn what the conserver does, get one running, configure it, and find the reference page for each part.
---

# Conserver

The conserver is an open source server that takes in vCons, runs each one through a configured chain of processing steps, and writes the result to the storage systems you choose. Steps are called links: transcribe, summarize, tag, route, redact, notify, register on a SCITT ledger. The current build ships 23 links and 15 storage modules. Source: [vcon-dev/vcon-server](https://github.com/vcon-dev/vcon-server) (MIT, Python 3.12, FastAPI, Redis, Docker Compose).

Use a conserver when vCons arrive from adapters (phone systems, SIPREC, chat, AI agent sessions) and something has to happen to every one of them, the same way, with failures captured in dead letter queues.

## Path through this section

**1. Understand it**

- [Conserver Introduction](conserver-introduction.md): what a conserver is for, with public sources
- [Concepts](concepts.md): vCon, link, chain, ingress and egress lists, storage, dead letter queues, tracer, follower. The rules for how a chain runs live here.
- [Day in the Life of a vCon](day-in-the-life-of-a-vcon.md): one phone call followed from adapter to AI assistant
- [The Journey of a vCon](vcon-conveyor-infographic.md): the same trip as an interactive picture

**2. Run it**

- [Quick Start](conserver-quick-start.md): a working conserver on Docker Compose

**3. Configure it**

- [Configuring the Conserver](configuring-the-conserver.md): environment variables and every `config.yml` section
- [Standard Links](standard-links.md): all 23 links and their options
- [Storage](storage.md): all 15 storage modules
- [Conserver Tracers](conserver-tracers.md): the JLINC audit tracer

**4. Build on it**

- [Integrating Your App](integrating-your-app.md): sending vCons in and reading results out
- [API](api.md): every REST endpoint
- [Creating Custom Links](creating-custom-links.md): writing your own processing step

**5. Operate it**

- [Production Deployment](production-deployment.md): scaling, secrets, observability
- [Inside the Conserver](inside-the-conserver.md): workers, Redis keys and the processing loop
- [Troubleshooting](troubleshooting.md): common failures and fixes
- [Operational Benefits](operational-benefits-of-conservers.md): what running one buys you, with public sources
