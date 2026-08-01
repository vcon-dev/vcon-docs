---
description: Existing privacy law already applies to conversations. This episode shows how a vCon carries consent, purpose limits, minimization, and redaction inside the conversation itself.
---

# 🔒 Thinking in vCon: Privacy

{% embed url="https://www.youtube.com/watch?v=dw4mMcp7jD0" %}
Thinking in vCon: Privacy — 15:16, a companion to the main talk.
{% endembed %}

## The argument

One example carries the whole talk: a recorded customer support call.

GDPR, CCPA, and comparable rights elsewhere in the world already apply to that call. The person on the other end of it can ask what was collected, object to a new use, or ask to be forgotten. None of that is new law and none of it is seriously disputed. What is missing is machinery. In a typical stack the recording lands in one system, the transcript in another, the summary in a third, and the consent, if it was captured at all, sits in a checkbox somewhere in a fourth. Rights that exist on paper become expensive or impossible to honor in practice.

The claim this episode makes is that privacy here is a container problem before it is a policy problem. When consent, purpose limitation, data minimization, and redaction travel inside the conversation object, permissions can be verified by whoever receives it rather than assumed on the strength of a contract. A downstream tool does not have to trust that the right thing happened upstream. It can check.

## What that looks like in a vCon

Consent travels with the dialog, recorded per purpose in the object itself rather than in a separate system that the recording will eventually be separated from. Purpose limits are explicit, so "recorded for quality assurance" and "usable as model training data" are distinguishable permissions, and an object can carry one without the other. Minimization and redaction become visible, verifiable states of the record instead of undocumented side effects of some pipeline. And because the object is signed and carries a tamper-evident history, a recipient can tell whether the consent they are reading is the consent that was actually given.

The practical consequence is that answering a right-to-know or right-to-erasure request stops being a project and becomes a query against objects that already carry the answer.

## Watch the short version

Five minutes instead of fifteen: [Privacy (short)](https://www.youtube.com/watch?v=XAAaSRiyBe4) makes the same argument without the worked example.

## Go deeper

- [Privacy Primer](../vcons/privacy-primer.md) — a short orientation to data and communications privacy for people building or reviewing vCon-based systems
- [Lawful Basis](../extensions/lawful-basis.md) — the extension that records legal grounds for processing, with cryptographic proof and per-purpose consent
- [Privacy-First Conversation Management](../deep-dives/privacy-first-conversation-management.md) — the technical whitepaper behind this episode
- [Lifecycle (SCITT)](../extensions/lifecycle.md) — append-only audit ledger for consent and deletion events
- [vCons and Increasing End User Agency](../deep-dives/vcons-and-increasing-end-user-agency.md) — the same argument from the data subject's side
