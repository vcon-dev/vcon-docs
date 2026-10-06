---
description: >-
  Introduces vCon, the open IETF format for recording a conversation and what
  was done with it, and gives a newcomer a short path from first read to
  running code.
---

# 👋 Welcome to the Home of the vCons

## A container for every conversation, human or agent

Calls, chats, video meetings and agent dialog have had no shared format for recording who took part, what was said, and what has been done with it since, in a form that survives moving between systems.

[vCon (virtual conversation)](vcons/a-vcon-primer.md) is that format. It is an IETF draft that packages the parties, the dialog, the analysis and related attachments into one portable JSON object. A vCon SHOULD be signed or encrypted when it leaves the system that built it, and extensions add a lawful basis record, a lifecycle log and more. The Conserver is the open source server that creates and processes vCons.

{% columns %}
{% column %}
[![](<.gitbook/assets/Journey Badge 540 (1).png>)](https://the-journey-of-a-vcon.netlify.app/)
{% endcolumn %}

{% column %}
{% embed url="https://www.youtube.com/watch?v=MjhpMnvcNds" %}
{% endcolumn %}
{% endcolumns %}

## Start here

1. **Learn what a vCon is.** Read [A vCon Primer](vcons/a-vcon-primer.md), then keep [Concepts](vcons/concepts.md) and the [Field Reference](vcons/field-reference.md) open.
2. **Run a conserver.** The [Conserver Quick Start](conserver/conserver-quick-start.md) gets a vCon pipeline running on your machine.
3. **Build with a library or adapter.** Use the [Python library](vcon-library/quickstart.md) or the [JavaScript library](vcon-js-library/quickstart.md), or wire your platform in with the [adapter template](vcon-adapters/quick-start-from-template.md).
4. **Add extensions.** [Extensions](extensions/README.md) covers lawful basis, lifecycle, WTF transcription, agent sessions and SIP signaling.
5. **Connect AI agents.** The [MCP Server](mcp-server/README.md) lets agents read and search a vCon store.

Compliance and policy readers can go from the primer to the [Privacy Primer](vcons/privacy-primer.md) and the [Lawful Basis extension](extensions/lawful-basis.md).

## What is in the box

{% tabs %}
{% tab title="The vCon object" %}
<figure><img src=".gitbook/assets/Conserver Pictures (8).jpg" alt=""><figcaption><p>A JSON object that carries parties, dialog (recording or transcript), analysis and attachments. Signed, it is tamper-evident. The same format works for a phone call, a chat session, a video meeting, or a conversation between a person and an AI agent.</p></figcaption></figure>
{% endtab %}

{% tab title="The Conserver" %}
<figure><img src=".gitbook/assets/Conserver Internals (5).jpg" alt=""><figcaption><p>The open source server that creates vCons from business systems, runs them through chains of links (transcribe, redact, analyze, forward) and writes them to storage.</p></figcaption></figure>
{% endtab %}

{% tab title="MCP server and adapters" %}
<figure><img src=".gitbook/assets/App Integration.jpg" alt=""><figcaption><p>Applications consume vCons through the Conserver API, the MCP server, or the language libraries. Adapters convert recordings and call data from existing platforms into vCons.</p></figcaption></figure>
{% endtab %}
{% endtabs %}

## Open standard

vCon is developed in the [IETF VCON working group](https://datatracker.ietf.org/group/vcon/about/), where the drafts, mailing list and meeting materials are public. Anyone can read the drafts, join the list and challenge a design decision in writing. As of 2026-10-06 no IPR disclosures had been filed against the core draft, according to the [datatracker IPR search](https://datatracker.ietf.org/ipr/search/?draft=draft-ietf-vcon-vcon-core&submit=draft). Public talks and articles about vCon are collected in [Talks, Articles and Press](talks-articles-press/).

{% embed url="https://github.com/vcon-dev/vcon" %}
Open source repository for vCon and the Conserver
{% endembed %}
