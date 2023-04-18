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

## Data Minimization

Data minimization is a privacy and data protection principle that aims to limit the collection, processing, and retention of personal data to what is strictly necessary for a specific purpose. The principle of data minimization is based on the idea that organizations should only collect, use, and store the minimum amount of personal data required to achieve a legitimate purpose, and should avoid collecting or retaining excessive or irrelevant data.

Data minimization is a key component of many privacy laws and regulations, including the European Union's General Data Protection Regulation (GDPR). It is closely related to other data protection principles, such as purpose limitation and storage limitation.

The principle of data minimization can be broken down into three main components:

1. Collection Minimization: Organizations should only collect personal data that is directly relevant and necessary for the intended purpose. They should avoid collecting data that is not needed for the specific purpose for which it is being collected. For example, if a company is collecting data for the purpose of processing an online order, it should not collect information about the customer's hobbies or political preferences, as this information is not relevant to the transaction.
2. Use Minimization: Organizations should only use personal data for the specific purpose for which it was collected and should avoid using the data for other purposes unless there is a legitimate basis for doing so. For example, if a company collects a customer's email address for the purpose of sending a receipt, it should not use that email address to send marketing materials unless the customer has explicitly consented to receive such communications.
3. Retention Minimization: Organizations should only retain personal data for as long as necessary to fulfill the specific purpose for which it was collected. Once the data is no longer needed for that purpose, it should be securely deleted or anonymized. For example, if a company collects credit card information to process a one-time payment, it should not retain that information indefinitely after the payment has been processed.

Implementing data minimization practices can help organizations reduce the risk of data breaches, protect individuals' privacy rights, and comply with legal and regulatory requirements. Additionally, data minimization can lead to cost savings by reducing the amount of data that needs to be stored and managed, and it can help organizations build trust with their customers and stakeholders by demonstrating a commitment to privacy and data protection.

## Data Protection

Data protection is closely related to data privacy, and the terms are sometimes used interchangeably. However, data protection is more focused on the technical and organizational measures that organizations take to secure personal data, while data privacy encompasses the broader set of legal, ethical, and social considerations related to the collection and use of personal data.

## Consent

Customer consent refers to the process by which a customer or user gives their explicit and informed permission for an organization to collect, use, or disclose their personal data for specific purposes. Consent is a fundamental concept in privacy and data protection laws, and it is one of the legal bases that organizations can rely on to process personal data.

Key aspects of customer consent include the following:

1. Freely Given: Consent must be given voluntarily, without coercion or undue pressure. Customers should have a genuine choice to either give or withhold their consent. If consent is required as a condition for accessing a service, and there is no reasonable alternative, the consent may not be considered freely given.
2. Specific: Consent must be specific to the particular purpose or purposes for which the data will be used. Organizations should clearly explain the purpose of data collection and processing, and consent should not be sought for broad or undefined purposes.
3. Informed: Customers must be provided with clear and understandable information about what data will be collected, how it will be used, and with whom it may be shared. This information should be presented in a transparent and easily accessible manner, so that customers can make an informed decision about whether to give their consent.
4. Unambiguous: Consent must be expressed through a clear, affirmative action. This means that customers must actively indicate their agreement, such as by ticking a box, signing a form, or clicking a button. Passive or implied consent (e.g., pre-ticked boxes or inactivity) is not considered valid consent.
5. Revocable: Customers have the right to withdraw their consent at any time, and the process for doing so should be straightforward and easily accessible. Organizations must inform customers of their right to withdraw consent and provide a mechanism for doing so.
6. Documented: Organizations should keep a record of when and how consent was obtained, as well as the specific information that was provided to the customer at the time of consent. This documentation can help demonstrate compliance with legal and regulatory requirements.

It's important to note that there are certain situations where consent may not be the appropriate legal basis for processing personal data. For example, consent may not be valid if there is a significant power imbalance between the organization and the individual (e.g., in an employer-employee relationship), or if the processing is necessary to fulfill a legal obligation or a contract.

Additionally, some data protection laws, such as the European Union's General Data Protection Regulation (GDPR), provide enhanced protections for certain categories of sensitive personal data (e.g., health information, religious beliefs, sexual orientation), and additional requirements may apply when seeking consent for the processing of such data.

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

## Encryption and Signing

vCons support encryption of the parent object, and each of the analysis sections can carry encrypted bodies.   In addition, external URLs and the vCon itself are signed, allowing for tamper detection of the contents.



