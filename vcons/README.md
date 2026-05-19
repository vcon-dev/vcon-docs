---
icon: cassette-tape
---

# vCons

Conversations are the last big category of business content that never got a file format. Pictures got JPEG. Documents got PDF. Spreadsheets got XLSX. Contacts got vCard. Conversations got vCon.

A vCon (Virtualized Conversation) is a portable, verifiable container for a single conversation. It is a JSON object that carries the parties, the dialog (text, audio, video, messaging), the recording or transcript, the consent that was given, and the analysis that has been run on it. It can be signed, encrypted, attached to an email, stored to disk, and sent across a network. It is developed as an open standard under the IETF VCON working group, with reference implementations in Python and JavaScript and a growing ecosystem of adapters, stores, and tools.

Two properties of the format are worth highlighting up front. First, consent travels inside the file. When a customer withdraws consent, every downstream copy of the conversation is governed by that withdrawal, not just the system that originally captured it. This is the difference between a consent management system that reports on policy and one that enforces it. Second, vCons can be data reduced. A derived vCon can be produced that contains only the fields you want to share, and the new file points back to the original so the recipient can verify it is real without seeing everything.

The reason this matters now is that agentic AI is moving from demo to production. Agents are starting to talk to customers and to each other, and there is currently no shared way to record what an agent said, on whose behalf, or under what authority. At the same time the line between authentic and synthetic media is collapsing: a recorded conversation injected into an AI pipeline is the conversational equivalent of malware injected into a software supply chain. vCon plus its companion lifecycle work in SCITT gives the industry a verifiable, signed, consent-aware container for conversations so that the trust questions have a place to live.

vCon is past the pilot stage. Implementations are running at hundreds of thousands of conversations per month at the BPO that incubated the technology, at millions per day at a large financial institution that is now also the first production deployment of real-time vCons, and as a routing prototype at a United Way 211 center. Around thirty to forty companies are actively building with vCons today. Telecom and call-center vendors are leaning in first, which is the usual pattern for an open standard.

## Start here

* [💬 A vCon Primer](a-vcon-primer.md) — the longer-form introduction
* [🧠 Why vCons?](why-vcons.md) — the open-standard and CPaaS case
* [🌎 vCons are...](vcons-are....md) — six short framings
* [💡 Concepts](concepts.md) — the vocabulary you will see in the spec
* [🔒 Privacy primer](privacy-primer.md) — GDPR, CCPA, consent vocabulary
* [✨ More Information](more-information.md) — drafts, talks, articles, podcasts

## Authoritative links

* [IETF VCON working group](https://datatracker.ietf.org/group/vcon/about/)
* [vCon GitHub repository](https://github.com/vcon-dev/vcon)
* [The Pulver vCon Report](https://thejeffpulver.substack.com/) — the ongoing industry-side commentary
* [A Comprehensive Guide to vCon in Communications](https://www.cavell.com/a-comprehensive-guide-to-vcon-in-communications/) — Cavell analyst overview
* [vCon: The Power of a Definition](https://cpaasaa.com/vcon-the-power-of-a-definition/) — CPaaSAA, on why the file format matters
