---
description: >-
  Shows which AI and security rules require a record of what an AI system was
  given, what it did and who was involved, and where a vCon already holds each
  piece, so you can answer an auditor from the conversation itself.
---

# 🏛️ AI Regulation Primer

{% hint style="info" %}
**Not legal advice.** This page maps what regulators ask for onto the vCon format. Whether a given system is in scope, and what it must keep, depends on the law of the place it runs and the sector it serves. Ask counsel for your own case.
{% endhint %}

Put an AI agent on a phone line, in a chat window or behind a support desk and a new question follows it: show me what it did. The rules written for AI over the last two years, and the older security frameworks being stretched to cover it, ask that question in different words. Read side by side, they converge on one record: what the system was given, what it produced, which tools it called, who was involved, when, and proof that nobody changed the record afterwards.

For conversations, that record is a vCon. This page lists the frameworks, the eleven requirements they share, and where each requirement lands in a vCon and the conserver that builds it.

The analysis follows the compliance requirements worked out for Verifiable Agent Conversations (VAC), `draft-birkholz-verifiable-agent-conversations`, which the [Agent Session extension](../extensions/agent-session.md) carries inside a vCon (1).

## Frameworks that ask for a record

| Framework | Where | Applies to | Status, November 2026 |
| --- | --- | --- | --- |
| EU AI Act | EU | AI systems, by risk tier | In force since August 2024, obligations staged (see below) |
| Cyber Resilience Act (CRA) | EU | Products with digital elements | Reporting duties from 11 September 2026, the rest from 11 December 2027 |
| NIS2 Directive | EU | Essential and important entities | Transposition deadline was October 2024; national laws vary |
| ETSI TS 104 223 | EU and international | AI models and systems | Published April 2025 |
| ISO/IEC 42001 | International | AI management systems | Published 2023 |
| ISO/IEC 24970 (4) | International | AI system logging | Final draft, in approval |
| PCI DSS v4.0.1 and PCI SSC AI guidance | International | Payment card environments | Mandatory since March 2025; AI Principles September 2025 |
| SOC 2 Trust Services Criteria | US and international | Service organizations | Active |
| FedRAMP (NIST SP 800-53, OMB M-21-31) | US | Federal cloud services | Active |
| FFIEC IT Handbook, BSI AI finance test criteria | US, Germany | Financial institutions | Active |

### EU AI Act dates after the omnibus

The Digital Omnibus on AI, Regulation (EU) 2026/1744, entered into force on 27 July 2026 and moved the high-risk dates (2):

| Obligation | Applies from |
| --- | --- |
| Article 50 transparency: tell people they are dealing with an AI | 2 August 2026 |
| Article 50(2) marking of generated content, systems already on the market | 2 December 2026 |
| High-risk systems in Annex III (employment, credit, essential services and others), including Article 12 logging | 2 December 2027 |
| High-risk AI inside regulated products (Annex I) | 2 August 2028 |

For a contact center the first row is already live. An AI agent that talks to callers must say it is an AI. The disclosure is a turn in the dialog, with a party and a start time, so the vCon shows that it happened and when.

## Eleven requirements in common

Not every framework asks for every item, but most ask for most of them. Treat the list as the floor a conversation record has to clear.

| # | Requirement | Where the rules say it | Where it lives in a vCon |
| --- | --- | --- | --- |
| 1 | Automatic logging | AI Act Art. 12(1); NIST AU-12; PCI DSS 10.2; ISO 42001 A.6.2.8 | The adapter and conserver build the vCon as the call or session runs. No one types the record up afterwards. |
| 2 | Timestamps | AI Act Art. 12(3)(a); PCI DSS 10.6; NIST AU-8 | `start` on every dialog and attachment, ISO 8601 with a time zone. Trace entries keep their order. |
| 3 | Who acted | AI Act Art. 12(3)(d); PCI DSS 10.2; NIST AC-2 | `parties[]`. People by `tel`, `mailto` or `name`. The agent as its own party with `role: "agent"` and `meta.agent_session` naming `model_id` and `provider`. Every dialog and attachment points at a party index. |
| 4 | What happened | AI Act Art. 12(2); PCI DSS 10.2; NIST AU-3 | Dialog `type`, and the VAC entry types inside the `agent_trace` analysis: user, assistant, tool-call, tool-result, reasoning, event. |
| 5 | Inputs and outputs | AI Act Art. 12(3)(c); PCI SSC AI Principles (3); ETSI TS 104 223 | User-facing turns in `dialog[]`. Prompts, tool inputs, tool outputs and responses in the `agent_trace` body. |
| 6 | Retention | AI Act Arts. 19 and 26(6); OMB M-21-31; PCI DSS 10.5 | A storage policy in the conserver, not a field. Deletion and expiry are recorded as lifecycle events. |
| 7 | Tamper evidence | PCI DSS 10.3; NIST AU-9; SOC 2 | The signed (JWS) form of the vCon. `content_hash` on every external reference. A SCITT receipt for the trace, in an attachment with `purpose: "scitt_receipt"`. |
| 8 | Incident response | NIS2 Art. 23; CRA Art. 14; AI Act Art. 73 | Structured fields let you pull every vCon by time window, party, tool or model when the clock starts. |
| 9 | Anomaly detection | AI Act Art. 12(2)(a); NIST SI-4; ETSI TS 104 223 | Tool calls and results paired by call id, with a `status`, so failed calls and odd tool patterns can be counted across a fleet. |
| 10 | Human oversight | AI Act Arts. 14 and 26(5); ISO 42001; PCI SSC AI guidance | Reasoning entries where the model exposes them. A human who approves an action is a party, and the approval is a recorded turn or event. |
| 11 | Traceability | AI Act Art. 12(1); ISO 42001; CRA; ETSI TS 104 223 | The vCon `uuid`; `redacted` and `amended` pointers back to prior versions; `parent-id` links in the trace; the agent's `environment` (branch, commit); `agent_file_change` attachments for what the agent touched. |

## One record, not three

Most systems keep these pieces in different places today. The call recording is in the telephony platform. The transcript is in an analytics tool. The model's prompts and tool calls are in an observability product, if anywhere. An auditor who asks what happened on one call gets three exports with three clocks and no common identifier.

A vCon puts them in one container that shares one set of parties, one timeline and one signature. The conversation is in `dialog[]`. What the agent did to produce its side of it is an `analysis[]` entry of `type: "agent_trace"` holding the VAC record. Files and actions the agent produced are `attachments[]`. The lawful basis for all of it is one more attachment. Sign the vCon and every piece is covered at once.

That is what most of the eleven requirements are really asking for. Rows 3, 4, 5 and 11 are satisfied by the structure itself. Row 7 is satisfied by signing. Rows 8 and 9 follow from the data being structured rather than free text.

## The conserver's part

A vCon is a format. The conserver is where the requirements become behavior.

- **Capture** is automatic because an adapter creates the vCon from the call or the agent session, not a person (row 1).
- **Retention** is a storage setting. Set the floor at twelve months unless your sector says otherwise; that covers PCI DSS and the AI Act's six months, and FedRAMP asks for more (row 6).
- **Integrity** comes from a signing step before storage, and from a link that registers the vCon or its trace with a SCITT transparency service (row 7). The [Lifecycle](../extensions/lifecycle.md) extension records what happened to the vCon afterwards, sent, redacted, deleted, as events on that service.
- **Gating** comes from a [lawful basis](../extensions/lawful-basis.md) check link that runs before any AI link, so a conversation with no grant for analysis never reaches a model.

### Retention floors

| Framework | Minimum |
| --- | --- |
| EU AI Act Art. 19 (providers) and Art. 26(6) (deployers) | 6 months, longer where sector law requires |
| PCI DSS v4.0.1, Req. 10.5.1 | 12 months, the last 3 immediately available |
| FedRAMP, per OMB M-21-31 | 12 months active plus 18 months in cold storage |
| NIS2 | Set by each member state |

## Logs as personal data

Every requirement above pushes toward keeping more, for longer. Privacy law pushes the other way. A conversation record holds what callers said, and a tool result can hold whatever the agent looked up about them. Reasoning entries can hold confidential business logic. The [Privacy Primer](privacy-primer.md) covers the rights involved; three things matter for AI logging in particular.

- **Purpose.** Keeping a record to meet a logging duty is a different purpose from training a model on it. A [lawful basis](../extensions/lawful-basis.md) attachment states which purposes were granted, including `agent_session_recording` and `agent_session_analysis` for agent traces.
- **Minimization.** When an authority or auditor asks for logs, send a derived copy with what they need. The `redacted` pointer ties the copy to the original, and the original stays intact for the next request.
- **Access.** Control who can read the record, and log the reads. The AI Act's authority-access duty implies an audit trail of the audit trail.

## Limits of the format

A vCon records. It does not decide whether a system is high-risk, whether a disclosure was adequate, or whether a retention period is lawful. It can only hold the reasoning a model exposes; many hosted models return a summary or nothing. And a signed record proves the record was not changed after signing. It does not prove the system that wrote it told the truth. Those limits are why the format pairs with transparency logs, consent records and human review rather than replacing them.

## Further reading

- [Agent Session extension](../extensions/agent-session.md): the agent party, the `agent_trace` analysis entry and agent attachments, with examples.
- [Lawful Basis extension](../extensions/lawful-basis.md): the consent and purpose record that gates processing.
- [Lifecycle (SCITT) extension](../extensions/lifecycle.md): recording what happened to a vCon after it was made.
- [Privacy Primer](privacy-primer.md): the privacy rights a conversation record carries.
- [A vCon Primer](a-vcon-primer.md): what a vCon is, for readers arriving here first.

## Notes

1. [draft-birkholz-verifiable-agent-conversations](https://datatracker.ietf.org/doc/draft-birkholz-verifiable-agent-conversations/), H. Birkholz, T. Heldt, O. Steele. Work in progress. The compliance analysis is Section 2 of -01. The vCon profile is [draft-howe-vcon-agent-session](https://datatracker.ietf.org/doc/draft-howe-vcon-agent-session/).
2. [Regulation (EU) 2024/1689](https://eur-lex.europa.eu/eli/reg/2024/1689/oj), the AI Act, as amended by Regulation (EU) 2026/1744. Dates as reported by [Gibson Dunn](https://www.gibsondunn.com/eu-ai-act-omnibus-agreement-postponed-high-risk-deadlines-and-other-key-changes/) and [ActuIA](https://www.actuia.com/en/news/ai-act-obligations-in-force-and-delays-under-omnibus-2026-1744/); check the consolidated text on EUR-Lex before relying on them.
3. PCI SSC, [AI Principles: Securing the Use of AI in Payment Environments](https://blog.pcisecuritystandards.org/ai-principles-securing-the-use-of-ai-in-payment-environments), September 2025: "Where possible, logging should be sufficient to audit the prompt inputs and reasoning process used by the AI system that led to the output provided." The October 2026 *Security Considerations for AI Systems* adds a recommendation of explicit human approval for agent actions on unencrypted cardholder data.
4. [ISO/IEC FDIS 24970](https://www.iso.org/standard/88723.html), Artificial intelligence, AI system logging. In the approval phase as of this writing.
