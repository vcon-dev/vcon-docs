---
description: >-
  vCon is the open container for conversations. Signed, consent aware, and
  portable, it lets organizations record what was said, by whom, and under what
  authority, in a way that machines and auditors can both trust.
---

# 👋 Welcome to the Home of the vCons

## A trust layer for every conversation, human or agent

Every business runs on conversations. Calls, chats, video meetings, agent dialog. Until now there has been no shared way to record what was said, on whose behalf, and under what authority, in a form that survives moving between systems.

vCon (virtual conversation) is that container. It is an open IETF standard for packaging the parties, the dialog, the recording or transcript, the consent, and the analysis into one signed, portable JSON object. The Conserver is the open source platform that creates and manages vCons at scale.

If you read one page, read this:

{% content-ref url="vcons/a-vcon-primer.md" %}
[a-vcon-primer.md](vcons/a-vcon-primer.md)
{% endcontent-ref %}

## Why this matters now

{% tabs %}
{% tab title="Agentic AI in production" %}
<figure><img src=".gitbook/assets/Conserver Pictures (8).jpg" alt=""><figcaption><p>AI agents are starting to talk to customers and to each other. There is no shared record of what an agent said, on whose behalf it spoke, or under what authority it acted. vCon is that record. It captures the dialog, the parties, and the consent in a single signed object that any downstream tool can verify.</p></figcaption></figure>
{% endtab %}

{% tab title="Authentic vs synthetic" %}
<figure><img src=".gitbook/assets/SCITT and vCon (2).jpg" alt=""><figcaption><p>Deepfakes blur the line between real and synthetic conversation. A spoofed call injected into an AI pipeline is the conversational analog of malware injected into a software supply chain. vCon pairs with SCITT, the IETF effort for Supply Chain Integrity, Transparency and Trust, so every lifecycle event (creation, sharing, analysis, deletion) is recorded in an append only ledger that cannot be altered after the fact.</p></figcaption></figure>
{% endtab %}

{% tab title="Consent that travels" %}
<figure><img src=".gitbook/assets/Conserver Internals.jpg" alt=""><figcaption><p>Consent today lives in policies, contracts and screenshots. It rarely travels with the conversation it covers. vCon carries consent inside the file itself, scoped by purpose and time. That is the difference between reporting on a privacy policy and enforcing one. GDPR style deletion, AI training opt outs, and downstream sharing all become verifiable acts rather than promises.</p></figcaption></figure>
{% endtab %}

{% tab title="Out of the silo" %}
<figure><img src=".gitbook/assets/App Integration (1).jpg" alt=""><figcaption><p>Conversation data sits trapped inside contact center platforms, recording vendors, and AI tools that do not interoperate. The same vCon can be read by a contact center analytics tool, a compliance audit, and an AI agent without any of them agreeing on a private format first. The format is open, free of intellectual property encumbrances, and aimed at the next decade of conversation infrastructure.</p></figcaption></figure>
{% endtab %}
{% endtabs %}

{% hint style="info" %}
**Provenance over policy.** vCon does not ask you to trust a vendor that says the right things. It puts cryptographic provenance, parties, and consent inside the file itself, so the next system in line can verify them on its own.
{% endhint %}

## Find your path

{% tabs %}
{% tab title="Architects and CTOs" %}
You want to know where vCon fits in your stack and whether it is real.

* [vCons are...](vcons/vcons-are....md) — the one paragraph mental model
* [Why vCons need a file](vcons/why-vcons.md) — conversations as durable digital artifacts in the AI era
* [Conserver Introduction](conserver/conserver-introduction.md) — the reference platform that produces and processes vCons
* [Day in the Life of a vCon](conserver/day-in-the-life-of-a-vcon.md) — end to end flow
{% endtab %}

{% tab title="Business and compliance" %}
You care about consent, regulatory exposure, and AI governance.

* [A vCon Primer](vcons/a-vcon-primer.md) — the full story, written for non specialists
* [Privacy Primer](vcons/privacy-primer.md) — how vCon handles PII, deletion and lawful basis
* [Lawful Basis extension](extensions/lawful-basis.md) — consent that travels with the data
* [PII Compliance use case](use-cases-studies/pii-compliance.md) — what this looks like in practice
{% endtab %}

{% tab title="Developers and engineers" %}
You want to read code and get something running.

* [Conserver Quick Start](conserver/conserver-quick-start.md) — a vCon pipeline on your machine
* [vCon Library (Python) Quickstart](vcon-library/quickstart.md)
* [vCon JS Library Quickstart](vcon-js-library/quickstart.md)
* [vCon Adapters Quick Start](vcon-adapters/quick-start-from-template.md) — wire your platform into the ecosystem
* [MCP Server](mcp-server/README.md) — expose vCons to AI agents safely
{% endtab %}

{% tab title="Standards and policy" %}
You are tracking the working group and the surrounding standards.

* [IETF VCON working group](https://datatracker.ietf.org/group/vcon/about/)
* [Core draft `draft-ietf-vcon-vcon-core`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/)
* [SCITT and vCon together](deep-dives/scitt-supply-chain-integrity-transparency-and-trust.md)
* [IETF sessions, talks and press](talks-articles-press/README.md)
{% endtab %}
{% endtabs %}

## What is in the box

{% tabs %}
{% tab title="The vCon object" %}
<figure><img src=".gitbook/assets/Conserver Pictures (8).jpg" alt=""><figcaption><p>A signed JSON object that carries parties, dialog (recording or transcript), attachments, analyses, consent, and a tamper evident history. The same format works for a phone call, a chat session, a video meeting, or a human to agent conversation.</p></figcaption></figure>
{% endtab %}

{% tab title="The Conserver" %}
<figure><img src=".gitbook/assets/Conserver Internals (5).jpg" alt=""><figcaption><p>The open source platform that creates vCons from business systems, runs them through pipelines of links (transcribe, redact, analyze, store, forward), and emits archive copies plus projections into the tools the business already uses.</p></figcaption></figure>
{% endtab %}

{% tab title="MCP server and adapters" %}
<figure><img src=".gitbook/assets/App Integration.jpg" alt=""><figcaption><p>Applications consume vCons through the Conserver API, the MCP server, or the language libraries. Adapters let CPaaS providers, contact centers, and recording vendors plug their platforms into the ecosystem without inventing a new format.</p></figcaption></figure>
{% endtab %}
{% endtabs %}

## Open, by design

vCon is developed in the open at the IETF, the same standards body that produced the protocols the internet runs on, from TCP/IP and DNS to HTTP, TLS, and SIP. IETF work is rough consensus, running code, and a public record. Anyone can read the drafts, join the list, and challenge a design decision in writing.

The format itself carries no patent encumbrance, by design, the same way PDF and vCard do not. The working group brings together telecom regulators, carriers, hyperscale platforms, and human rights organizations, and public statements from those constituencies appear in [Talks, Articles and Press](talks-articles-press/README.md).

{% embed url="https://github.com/vcon-dev/vcon" %}
Open source repository for vCon and the Conserver
{% endembed %}

{% embed url="https://datatracker.ietf.org/group/vcon/about/" %}
IETF VCON working group
{% endembed %}

## Keep reading

{% content-ref url="vcons/a-vcon-primer.md" %}
[a-vcon-primer.md](vcons/a-vcon-primer.md)
{% endcontent-ref %}

{% content-ref url="vcons/why-vcons.md" %}
[why-vcons.md](vcons/why-vcons.md)
{% endcontent-ref %}

{% content-ref url="vcons/concepts.md" %}
[concepts.md](vcons/concepts.md)
{% endcontent-ref %}

{% content-ref url="conserver/conserver-quick-start.md" %}
[conserver-quick-start.md](conserver/conserver-quick-start.md)
{% endcontent-ref %}

{% content-ref url="use-cases-studies/overview.md" %}
[overview.md](use-cases-studies/overview.md)
{% endcontent-ref %}
