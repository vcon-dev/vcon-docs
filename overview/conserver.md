---
description: A data platform for gathering, creating, storing and sharing vCons
---

# 🚀 Conserver

The conserver is a data platform designed to extract conversations from business phone systems, transform them into actionable insights, and send that data into common business tools such as spreadsheets, Salesforce and no code toolsets. An open core product, the conserver enables data engineering teams to supply a reliable source of information for AI, ML and operational software in cloud, premise and hybrid contexts. The core for many of the business cases enabled by the conserver is the smart capture, redaction and lifecycle management of recorded customer conversations and customer journeys, recently accelerated by FTC and GDPR regulations and by increasing investments into AI and ML.

<figure><img src="../.gitbook/assets/Conserver Pictures (7).jpg" alt=""><figcaption></figcaption></figure>

From a system perspective, shown above, the Conserver attaches to information systems like Web Chat and call center queues, and extracts information from them after conversations are ended. This information is then cleaned and transformed into actionable data. For instance, a distributed call center might extract conversations from a group of sales agents, convert them into text, then filter those conversations looking for times when customers objected to a sale. These objections are then pushed into database tables and Google Sheets as a data self-service option for any business team. The conserver supports multiple data pipelines, each one extracting data from a number of systems, performing transformations such as translations, transcriptions and redactions, and then pushing the prepared data into applications to be used.

In contrast to other data platforms, the Conserver is dedicated to managing the particular complexities of real time conversational sources. For instance, the amount of bandwidth and storage required to manage an hour long audio recording is an order of magnitude larger than managing a typical business object like a PDF. However, even this is just a start. Video is a few orders of magnitude greater than that, and the data creation for service providers such as Zoom and Skype are magnitudes of order still greater. From a legal perspective, regulatory compliance for customer data protections are particular for recorded conversations, and require support for tracking data’s use by automations, and for tracking deletion from a “Right to be Forgotten” request.



