---
description: >-
  An essay on how vCon's lawful basis and lifecycle drafts can give people
  more practical control over recordings of their conversations, and where
  that control still falls short.
---

# vCons and Increasing End User Agency

{% hint style="info" %}
**Essay, May 2026.** This piece first described the consent model as a standalone "consent extension". That work is now [`draft-howe-vcon-lawful-basis`](https://datatracker.ietf.org/doc/draft-howe-vcon-lawful-basis/). For current field names, see [Lawful Basis](../extensions/lawful-basis.md). The argument draws on material from the Harvard Online course [Data Privacy and Technology](https://www.harvardonline.harvard.edu/course/data-privacy-technology), taught by Michael D. Smith and Jim Waldo, and on the published work cited below.
{% endhint %}

Most people have clicked "Accept All Cookies" and agreed to terms they did not read, then wondered what happens to their conversations and data afterwards. Privacy researchers call the missing piece end user agency: the practical ability to control your own data, beyond a theoretical right to privacy.

## End user agency

Agency, in the framing taught in the Data Privacy and Technology course, comes down to concrete questions. Can a person permanently delete their data from a company's servers? Can they restrict which outside services get access? Can they stop collection for a while, or for good, without losing the service? The course treats these controls as settings that run along a spectrum, which a company can turn up or down, rather than as a single yes or no. It also stresses the power gap between individuals and large platforms: the more of someone's life a technology touches, the less control that person tends to have.

## What the vCon drafts provide

**Purpose-level permissions.** Helen Nissenbaum's theory of contextual integrity holds that privacy is about whether information flows match the norms of the context it came from, not about secrecy ([Nissenbaum, "Privacy as Contextual Integrity", Washington Law Review 79, 2004](https://digitalcommons.law.uw.edu/wlr/vol79/iss1/10/)). The Lawful Basis draft fits that model. Its `purpose_grants` let a person allow recording for quality assurance and refuse sentiment analysis or marketing use of the same call.

**Accountability built into infrastructure.** Julie Cohen argues that "privacy's most enduring institutional failure modes flow from its insistence on placing the individual and individualized control at the center" ([Cohen, "Turning Privacy Inside Out", Theoretical Inquiries in Law 20(1), 2019](https://doi.org/10.1515/til-2019-0002)). Purpose grants alone would put the burden back on the individual. Paired with the [Lifecycle draft](../extensions/lifecycle.md), which records consent changes, sharing and deletion on a [SCITT](scitt-supply-chain-integrity-transparency-and-trust.md) log, they move part of that burden onto the systems that process the data.

**Time limits.** Shoshana Zuboff describes a "right to the future tense": the freedom to decide one's own future actions without being steered by those who profit from predicting them ([Zuboff, _The Age of Surveillance Capitalism_, PublicAffairs, 2019, ch. 11](https://www.hachettebookgroup.com/titles/shoshana-zuboff/the-age-of-surveillance-capitalism/9781610395694/)). The Lawful Basis draft adds an `expiration` and an optional revalidation interval, so a permission can lapse unless it is renewed.

**Verifiable records.** With a SCITT Transparency Service, consent decisions and their withdrawal become Signed Statements with Receipts. A person, or a regulator acting for them, can ask for proof that a withdrawal was recorded instead of taking the company's word for it.

**Rights support.** The Lawful Basis draft requires implementations to support access, rectification, erasure, portability and withdrawal of a lawful basis. These requirements come from that extension; the core vCon draft does not impose them.

**Proof of how consent was given.** The draft's `proof_mechanisms` record how a basis was established, from a verbal confirmation in the recording to a signed document. Ryan Calo's work on privacy and vulnerability describes how power and information gaps leave some people more exposed to privacy harm than others ([Calo, "Privacy, Vulnerability, and Affordance", DePaul Law Review 66, 2017](https://cyberlaw.stanford.edu/publications/privacy-vulnerability-and-affordance/)). A recorded proof gives those people something concrete to dispute.

## Where it falls short

**Adoption.** vCons help only where they are used. A small company that adopts them gives its customers more control; a dominant platform that does not keeps the status quo.

**Literacy.** Purpose-level consent assumes people understand terms like "sentiment analysis" or "biometric processing". Many do not.

**Take it or leave it.** If an essential service requires broad permissions, the ability to refuse is not much of a choice.

**Enforcement.** A standard cannot make anyone honor it. Without regulation and enforcement, organizations can skip consent mechanisms that are inconvenient for them.

**Consent fatigue.** More choices can mean more prompts, and people stop reading them.

## The case for it

Today it is hard for a person to know how their data is combined and by whom. Zuboff describes behavioral data feeding prediction products that are traded in what she calls behavioral futures markets (Zuboff 2019). vCons do not end that. They do make processing of conversation data conditional on a recorded, purpose-scoped, time-limited basis, and they make the record of what happened checkable by someone other than the company that did it. That moves privacy for conversations from a system that runs on trust toward one that can be verified, which is the kind of structural change Cohen argues for. Whether it spreads depends on market pressure, regulation and demand from the people whose conversations are recorded.
