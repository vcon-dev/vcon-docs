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

