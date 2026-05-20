---
description: A short orientation to data privacy and communications privacy for people building or reviewing vCon-based systems.
---

# 🔒 Privacy Primer

{% hint style="info" %}
**Read the full primer in the IETF draft.** This page is a short orientation. The authoritative version, with full treatment of data subjects' rights, sensitive data, deidentification, communications privacy, AI considerations, and security considerations, lives in [`draft-ietf-vcon-privacy-primer`](https://datatracker.ietf.org/doc/draft-ietf-vcon-privacy-primer/). If you are designing or reviewing a system that handles vCons, read the draft.
{% endhint %}

## Why a primer

The democratization of technology has produced a wave of new entrants in the market for personal data, driven by motives that range from commerce and regulation to fraud prevention and charitable causes. More and more of them touch conversational data as it crosses network boundaries. vCon is one of the artifacts that makes that crossing possible, by giving conversational data a structure that can be processed and shared ethically.

Many of those entrants do not arrive with a working understanding of data minimization, lawful basis for processing, redaction, the right to know, or the right to erasure. The vCon design decisions are a direct response to those concerns: encryption, signing for change detection, redacted versions that retain a verifiable trail back to the original. None of that does any good if the people building and operating the systems do not share a baseline vocabulary. That is what this primer, and the IETF draft behind it, are for.

## Who this is for

The draft is written for three audiences who tend to share the same room at IETF and around vCon work:

- **Engineers and technologists** who live in implementation detail and benefit from seeing the ethical and legal frame their designs sit inside.
- **Regulators, lawyers, and policy staff** who arrive with a constituency in mind and want to understand which of their concerns the vCon framework addresses and which it does not.
- **NGOs**, particularly those working on privacy, security, and human rights, who sit at the intersection of policy and technology and are reasonably skeptical of both commercial and government framings.

If you recognize yourself in one of those groups, the draft is written for you.

## What it covers

The primer is informational, not normative. Its goals are modest:

- Educate a growing audience on responsible handling of personal data, including biometric content inside audio and video.
- Build a shared understanding of what makes that handling hard.
- Be honest about what the vCon framework addresses and what it leaves to law, policy, and operational practice.
- Encourage thoughtful design and review of systems that process personal data.

It is a primer, not a panacea. Much like the distinction between HTTP and HTTPS, the vCon framework gives well-intentioned actors something to build on, while leaving the question of bad actors to the legal system.

## Privacy and vCon — in general

Privacy is sometimes summarized as "the right to be let alone." It helps to think of it in four aspects:

1. Personal information (data) privacy
2. Bodily privacy
3. Territorial privacy
4. Communications privacy

vCon concentrates on the first and the fourth: **data privacy** and **communications privacy**. A recorded conversation is, by construction, both. A voice recording is biometric data. The content can carry health, religious, or political information. The participants are identifiable natural persons. That is why vCon treats encryption, signing, redaction, and lifecycle tracking as first-class concerns rather than optional add-ons.

IETF standards already address privacy in Internet communications, including data minimization ([RFC 7258](https://www.rfc-editor.org/rfc/rfc7258)). They generally do not address the privacy of individuals' data with respect to the organizations that collect, process, and disclose it. The privacy primer extends to that second question.

## Where to go next

- [`draft-ietf-vcon-privacy-primer`](https://datatracker.ietf.org/doc/draft-ietf-vcon-privacy-primer/) — the full primer, including data subjects' rights, what counts as protected and sensitive data, deidentification and anonymization, communications privacy, AI-specific considerations, and security considerations.
- [Lawful Basis extension](../extensions/lawful-basis.md) — how consent and other legal bases for processing are captured inside a vCon.
- [Lifecycle extension](../extensions/lifecycle.md) — the append-only record of what has happened to a vCon, and how revocation propagates.
