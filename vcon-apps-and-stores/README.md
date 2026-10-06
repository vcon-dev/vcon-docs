---
icon: store
description: Explains what a vCon store and a vCon app are, with the talk that introduced the idea and the example code to start from.
---

# vCon Apps and Stores

A vCon store holds vCons. A vCon app reads them. Because a vCon is a JSON document, an app can start from a file or a database query and does not need a vendor API for each conversation source.

## The parts

- Creators produce vCons: phone systems, email, chat, voice automation and AI agents, directly or through [adapters](../vcon-adapters/README.md).
- Hosters store them and handle access and consent. The [conserver](../conserver/README.md) is one implementation, with Redis, S3, Mongo and other [storage backends](../conserver/storage.md).
- Data subjects are the people in the conversations. [Lawful Basis](../extensions/lawful-basis.md) records what each may be used for.
- Apps sit on top: dashboards, analytics, compliance checks and AI assistants. A hoster can also expose the store to AI clients through the [vCon MCP server](../mcp-server/README.md), which uses the Model Context Protocol.

vCons can be signed for integrity. The conserver can also register them with a SCITT (Supply Chain Integrity, Transparency and Trust) transparency log. SCITT is a transparency log, not a blockchain.

## Talk

{% embed url="https://youtu.be/TgAwtYP0pjA" %}

## Start building

- [vCon App Template](vcon-app-template.md) is a small Streamlit example app that reads vCons from MongoDB.
- [TADHack vCon](tadhack-vcon.md) is a set of 43 synthetic vCons for hackathon projects. More datasets are listed in [Public vCon Datasets](../tools/vcon-datasets.md).
