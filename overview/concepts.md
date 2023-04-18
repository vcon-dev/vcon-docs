---
description: Important Ideas for vCons and the Conserver
---

# 💡 Concepts

## vCon

A vCon is the container for data and information relating to a real-time, human conversation. It is analogous to a \[[vCard](https://datatracker.ietf.org/doc/html/draft-petrie-vcon#vCard)] which enables the definition, interchange and storage of an individual's various points of contact. The data contained in a vCon may be derived from any multimedia session, traditional phone call, video conference, SMS or MMS message exchange, webchat or email thread. The data in the container relating to the conversation may include Call Detail Records (CDR), call meta data, participant identity information (e.g. STIR PASSporT), the actual conversational data exchanged (e.g. audio, video, text), realtime or post conversational analysis and attachments of files exchanged during the conversation. A standardized conversation container enables many applications, establishes a common method of storage and interchange, and supports identity, privacy and security efforts&#x20;

## Conserver

The conserver is a data platform designed to extract conversations from business phone systems, transform them into actionable insights, and send that data into common business tools such as spreadsheets, Salesforce and no code toolsets. An open core product, the conserver enables data engineering teams to supply a reliable source of information for AI, ML and operational software in cloud, premise and hybrid contexts. The core for many of the business cases enabled by the conserver is the smart capture, redaction and lifecycle management of recorded customer conversations and customer journeys, recently accelerated by FTC and GDPR regulations and by increasing investments into AI and ML.

From a system perspective, shown above, the Conserver attaches to information systems like Web Chat and call center queues, and extracts information from them after conversations are ended. This information is then cleaned and transformed into actionable data. For instance, a distributed call center might extract conversations from a group of sales agents, convert them into text, then filter those conversations looking for times when customers objected to a sale. These objections are then pushed into database tables and Google Sheets as a data self-service option for any business team. The conserver supports multiple data pipelines, each one extracting data from a number of systems, performing transformations such as translations, transcriptions and redactions, and then pushing the prepared data into applications to be used.

In contrast to other data platforms, the Conserver is dedicated to managing the particular complexities of real time conversational sources. For instance, the amount of bandwidth and storage required to manage an hour long audio recording is an order of magnitude larger than managing a typical business object like a PDF. However, even this is just a start. Video is a few orders of magnitude greater than that, and the data creation for service providers such as Zoom and Skype are magnitudes of order still greater. From a legal perspective, regulatory compliance for customer data protections are particular for recorded conversations, and require support for tracking data’s use by automations, and for tracking deletion from a “Right to be Forgotten” request.

## Data Privacy

Data privacy, also known as information privacy or data protection, refers to the practice of safeguarding individuals' personal information from unauthorized access, use, disclosure, alteration, or destruction. It involves ensuring that individuals have control over their own personal data and that organizations that collect, store, and process personal data do so in a manner that respects individuals' privacy rights.

Key concepts and principles of data privacy include:

1. Consent: Organizations should obtain individuals' informed consent before collecting, using, or sharing their personal data. Consent should be freely given, specific, informed, and unambiguous.
2. Purpose Limitation: Personal data should be collected for specific, explicit, and legitimate purposes, and it should not be used for purposes that are incompatible with those for which it was originally collected.
3. Data Minimization: Organizations should collect only the minimum amount of personal data necessary to achieve the stated purpose. Excessive or irrelevant data should not be collected.
4. Accuracy: Personal data should be accurate, complete, and up-to-date. Organizations should take reasonable steps to correct or delete inaccurate data.
5. Storage Limitation: Personal data should be retained only for as long as necessary to fulfill the purposes for which it was collected. Organizations should establish retention policies and securely dispose of data that is no longer needed.
6. Security: Organizations should implement appropriate technical and organizational measures to protect personal data from unauthorized access, data breaches, theft, and other security risks. This may include encryption, access controls, and regular security assessments.
7. Transparency: Organizations should be transparent about their data collection and processing practices. This includes providing clear privacy policies and notices that inform individuals about how their data is being used and their rights regarding their data.
8. Individual Rights: Individuals have certain rights regarding their personal data, including the right to access their data, the right to request corrections or deletions, the right to object to certain uses of their data, and the right to data portability (the ability to transfer their data from one organization to another).
9. Accountability: Organizations are responsible for complying with data privacy laws and regulations and for demonstrating their compliance. This may involve conducting privacy impact assessments, appointing a data protection officer, and maintaining records of data processing activities.

Data privacy laws and regulations vary by jurisdiction, but many countries and regions have enacted legislation to protect individuals' personal data. Examples of data privacy laws include the General Data Protection Regulation (GDPR) in the European Union, the California Consumer Privacy Act (CCPA) in the United States, and the Personal Data Protection Act (PDPA) in Singapore.

Overall, data privacy is a critical aspect of modern society, as the collection and use of personal data have become pervasive in many aspects of life, including online services, healthcare, finance, and commerce. Respecting data privacy helps build trust between individuals and organizations and protects individuals' rights and freedoms.

## Data Protection

Data protection is closely related to data privacy, and the terms are sometimes used interchangeably. However, data protection is more focused on the technical and organizational measures that organizations take to secure personal data, while data privacy encompasses the broader set of legal, ethical, and social considerations related to the collection and use of personal data.

## Link

A link is the basic unit of processing in the conserver. A link takes a single vCon and processes it. Examples of links include:

* **Redaction**: removes personal information from a transcript or a recording
* **Webhook**: takes a vCon and HTTP Posts it to an external system
* **Summary**: takes a transcript and summarizes the conversation
* **Stitcher**: Takes information from external systems to improve vCon data. For instance. and inbound vCon could have the phone number of an agent. A stitcher will fill in missing details like name and email address.

A set of links forms a chain. A link can be a member of multiple chains.  The conserver is responsible for delivering the vCon UUID to each link via a REDIS key.  Links are described by REDIS keys with a prefix of link:, which are loaded on startup from the configuration file.

## Chain

A chain is a series of links that processes a vCon.  There is no theoretical limit to the number and length of chains.  On a timer, the conserver iterates over each configured chain in the system. Chains are described by REDIS keys with a prefix of chain:, themselves loaded on startup from the configuration file.

The input of the chain is one or more REDIS lists, which are themselves filled by adapters or by external systems, either directly in REDIS, or through the API.

## Storage

A storage is an external data store, the "final resting place" for vCons as they are processed by the conserver, and are specified for each chain.  Examples of storages include S3, Mongo, postgres or a local file system.  For some storages, such as the file system. S3 or mongo, the storage format is the standard vCon format.  For relational storages, such as postgres, a schema is defined in the repo.



