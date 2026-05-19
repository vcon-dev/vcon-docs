---
description: Important Ideas for vCons and the Conserver
---

# 💡 Concepts

## vCon

A vCon is the container for data and information relating to a real-time, human conversation. It is analogous to a \[[vCard](https://datatracker.ietf.org/doc/html/rfc6350)] which enables the definition, interchange and storage of an individual's various points of contact. The current specification is [`draft-ietf-vcon-vcon-core-02`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/), with syntax parameter `"vcon": "0.4.0"`. The data contained in a vCon may be derived from any multimedia session, traditional phone call, video conference, SMS or MMS message exchange, webchat or email thread. The data in the container relating to the conversation may include Call Detail Records (CDR), call meta data, participant identity information (e.g. STIR PASSporT), the actual conversational data exchanged (e.g. audio, video, text), realtime or post conversational analysis and attachments of files exchanged during the conversation. A standardized conversation container enables many applications, establishes a common method of storage and interchange, and supports identity, privacy and security efforts&#x20;

## Conserver

The conserver is a data platform designed to extract conversations from business phone systems, transform them into actionable insights, and send that data into common business tools such as spreadsheets, Salesforce and no code toolsets. An open core product, the conserver enables data engineering teams to supply a reliable source of information for AI, ML and operational software in cloud, premise and hybrid contexts. The core for many of the business cases enabled by the conserver is the smart capture, redaction and lifecycle management of recorded customer conversations and customer journeys, recently accelerated by FTC and GDPR regulations and by increasing investments into AI and ML.

From a system perspective, shown above, the Conserver attaches to information systems like Web Chat and call center queues, and extracts information from them after conversations are ended. This information is then cleaned and transformed into actionable data. For instance, a distributed call center might extract conversations from a group of sales agents, convert them into text, then filter those conversations looking for times when customers objected to a sale. These objections are then pushed into database tables and Google Sheets as a data self-service option for any business team. The conserver supports multiple data pipelines, each one extracting data from a number of systems, performing transformations such as translations, transcriptions and redactions, and then pushing the prepared data into applications to be used.

In contrast to other data platforms, the Conserver is dedicated to managing the particular complexities of real time conversational sources. For instance, the amount of bandwidth and storage required to manage an hour long audio recording is an order of magnitude larger than managing a typical business object like a PDF. However, even this is just a start. Video is a few orders of magnitude greater than that, and the data creation for service providers such as Zoom and Skype are magnitudes of order still greater. From a legal perspective, regulatory compliance for customer data protections are particular for recorded conversations, and require support for tracking data’s use by automations, and for tracking deletion from a “Right to be Forgotten” request.

## Parties

The parties section of the vCon is an array that refers to the people or systems in the conversation. Each "dialog" provides an index into the parties array to identify the people.  Each party identifies the

* Network identifier of the party, currently held in the 'tel" field
* The mail adress
* The name of the party
* The role of the party, represented by a string (we use labels like customer, agent)&#x20;
* A validation field, allowing for evidence of third party validation of the identities of the parties
* Other information, such as civic address, timezone or jCard.

## Dialogs

The dialogs section of the vCon is an array of transcripts and recordings that represent the media of the conversation itself.  Each dialog contains an identification of:

* The type of the dialog (recording, transcript)
* The start time
* Duration
* The parties in the conversation
* The originating party of the conversation
* The mimetype of the recording
* Any associated filename

The content of the dialog comes in two flavors: packed and unpacked.  Packed data is included in the vCon itself in the body field.  Unpacked data is not included in the vCon, but is instead referenced by URL, not necessarily publicly accessible.  For both cases, the media of the vCon can be signed to prevent tampering or modification after the vCon is constructed.&#x20;

## Analysis

The analysis section of the vCon is an array of objects that represents third party analysis of the vCon itself. Examples of Analysis includes:

* Sentiment analysis of the conversation itself
* A list of promises made by the people on the call
* A summary of the conversation

Each analysis captures the vendor, schema and details of the analysis itself. In addition to the value that the analysis provides, this also becomes an accounting of the times and places this conversation has been processed by third parties. This list is critical in compliance to data regulations as it allows data controllers to fulfill their obligations to reporting and removing personal data on demand of people and regulators.

## Attachments

The attachments section of the vCon is an array of objects that are documents, traces and other pieces of data that provides the context of a conversation.  For instance, a sales organization may store the lead information in the attachment; a conference call may include the powerpoint that was discussed.  Links in the conserver use attachments to store tracing information, such as the raw responses from external systems or the original source of the vCon.&#x20;

## Encryption and Signing

vCons support encryption of the parent object, and each of the analysis sections can carry encrypted bodies. In addition, external URLs and the vCon itself are signed, allowing for tamper detection of the contents.

## Data Projection

Since the vCon is a nested document, sometimes it is more convenient to format the data in a vCon in other formats, particularly for relational data storage. For instance, a call log link may need to provide a single database row for each vCon into a spreadsheet. A data projection picks and chooses the data inside the vCon to create the spreadsheet row. Normally, data is lost through a projection.

## Privacy and Consent in vCon

Privacy and consent are first-class concerns in vCon: the lawful basis for processing a conversation, and any consent that was given, travel inside the vCon itself, not in a separate system that points at it from the outside. When a customer withdraws consent, every downstream copy of the vCon is governed by that withdrawal.

For the vocabulary behind that work, including GDPR, CCPA, data minimization, sensitive personal data, and the mechanics of consent, see the [Privacy Primer](privacy-primer.md). For how consent and lawful basis are expressed inside a vCon, see the [Lawful Basis extension](../extensions/lawful-basis.md). For the append-only lifecycle record of what has happened to a vCon, see the [Lifecycle extension](../extensions/lifecycle.md).
