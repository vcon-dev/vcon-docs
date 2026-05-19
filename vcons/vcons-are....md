---
description: A Brief Introduction to vCons and the Conserver
---

# 🌎 vCons are...

## vCons are "PDFs" for Conversations

A vCon describes a conversation that involves a "natural" person.  As a simple example, a vCon could be created from the last conversation you had with a customer service agent.  Just like PDFs allow you to create and share any written document; vCons allow you to create and share any human conversation. A vCon identifies the people in the call, recordings and transcripts, analysis and supporting attachments, like documents and logs.  Implemented in easy to manage JSON, vCons are both encryptable and tamper proof.&#x20;

## vCons are an Essential Tool in the Management of Personal Data

By defining the contents and participants of a conversation, vCons enable software tools that are able assert the presence, the absence and the authenticity of personal and biometric information. By tracking and consolidating the interactions with external systems with personal data, vCons enable responsible management of "Right to be Forgotten" regulations.  vCons are best understood as a functional toolset for fine-grained control of personal information, authenticity, context and ethical treatment of personal data. &#x20;

## vCons are Open

vCons are both an open source project, it is also an open standards effort supported by the IETF.   vCons enable the Internet to work better by defining the interchange standard for our most personal information: the sounds of of voice, and the images of our face.  vCons enable the responsible management of personal data, enabling true compliance to customer data protections such as "Right to Know".&#x20;

## vCons are After the Fact or Real Time

vCons started as an after-the-fact format: an expression of a conversation that already happened, packaged for storage, analysis, and compliance. That is still the most common shape. Increasingly, vCons are also produced and updated in real time, while a conversation is still in progress, so that applications can act on a call as it happens rather than waiting for the recording. A large financial-institution deployment is now running real-time vCons in production at millions per day; the format supports both modes without changing shape.

## vCons Carry Their Own Consent

Consent and lawful basis travel inside the vCon. Sometimes the basis is consent. Sometimes it is a regulatory requirement, such as a stockbroker being required to record a trade call. Either way, the basis is part of the conversation record. When a customer withdraws consent, every downstream copy of the vCon is governed by that withdrawal. This is the difference between a consent management system that reports on policy and one that enforces it.

## vCons are the Trust Layer for AI Conversations

When a conversation feeds an AI pipeline, the pipeline is only as trustworthy as the conversation. A deepfake injected into an AI pipeline is the conversational analog of malware injected into a software supply chain. vCons answer that with signatures and lifecycle:

* vCons can be signed and encrypted, so a recipient can cryptographically verify the sender and detect tampering.
* vCons track external processing, including AI, transcription, redaction, and analysis, supporting an authoritative answer to "what systems have seen this conversation?"
* Combined with [SCITT](../deep-dives/scitt-supply-chain-integrity-transparency-and-trust.md), vCons get an append-only lifecycle record: every creation, share, analysis, and deletion event is recorded in a way that cannot be quietly altered later.
* vCons and the conserver allow conversations to move across security boundaries while tracking shared data and supporting remote deletion, which lines up with "Right to Know" and "Right to be Forgotten" under GDPR and CCPA.
