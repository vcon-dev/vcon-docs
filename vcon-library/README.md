---
icon: plug
---

# vCon Library

## vCon Python Library

### About the Library

The vCon (Virtual Conversation) library is a powerful Python tool designed to capture, structure, and manage conversation data in a standardized format. It provides a robust set of features for creating, manipulating, and analyzing digital representations of conversations, making it particularly useful for applications in customer service, call centers, chat systems, and any scenario where structured conversation data is valuable.

At its core, the vCon library allows you to create vCon objects, which serve as containers for all elements of a conversation. These objects can include multiple parties (participants in the conversation), a series of dialogs (individual messages or utterances), metadata (such as tags for easy categorization), attachments (like transcripts or other related files), and even analysis data (such as sentiment analysis results).

A vCon is the container for data and information relating to a real- time, human conversation. It is analogous to a \[vCard] which enables the definition, interchange and storage of an individual's various points of contact. The data contained in a vCon may be derived from any multimedia session, traditional phone call, video conference, SMS or MMS message exchange, webchat or email thread. The data in the container relating to the conversation may include Call Detail Records (CDR), call meta data, participant identity information (e.g. STIR PASSporT), the actual conversational data exchanged (e.g. audio, video, text), realtime or post conversational analysis and attachments of files exchanged during the conversation. A standardized conversation container enables many applications, establishes a common method of storage and interchange, and supports identity, privacy and security efforts.

Key capabilities of the vCon library include:

1. Creating and managing vCon objects with a flexible, extensible structure.
2. Adding and retrieving conversation participants (parties) with various attributes.
3. Recording and organizing dialog entries with timestamps, content, and sender information.
4. Attaching metadata and tags for easy categorization and searching.
5. Including file attachments related to the conversation.
6. Incorporating analysis data from various sources (e.g., sentiment analysis, topic classification).
7. Signing and verifying vCon objects for data integrity and authenticity.
8. Serializing vCon objects to and from JSON for easy storage and transmission.

The library is designed with extensibility in mind, allowing for easy integration with various analysis tools and systems. It also includes built-in support for handling different types of conversation data, including text, audio, and video.

By providing a standardized way to structure and manage conversation data, the vCon library enables powerful applications in areas such as conversation analytics, quality assurance, compliance monitoring, and machine learning model training for natural language processing tasks.

Whether you're building a customer service platform, a conversation analysis tool, or any application that deals with structured dialog data, the vCon library offers a comprehensive solution for capturing, storing, and working with conversation information in a consistent and powerful way.

### Features

* Create and manipulate vCon objects
* Add parties, dialogs, attachments, and analysis to vCons
* Sign and verify vCons using JWS (JSON Web Signature)
* Generate UUID8 identifiers
* Pack and unpack dialogs
* Add and retrieve tags

### IETF vCon Working Group

The vCon (Virtual Conversation) format is being developed as an open standard through the Internet Engineering Task Force (IETF). The vCon Working Group is focused on creating a standardized format for representing digital conversations across various platforms and use cases.

#### Participating in the Working Group

1. **Join the Mailing List**: Subscribe to the vCon working group mailing list at [vcon@ietf.org](mailto:vcon@ietf.org)
2. **Review Documents**:
   * Working group documents and drafts can be found at: https://datatracker.ietf.org/wg/vcon/documents/
   * The current Internet-Draft can be found at: https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-container/
3. **Attend Meetings**:
   * The working group meets virtually during IETF meetings
   * Meeting schedules and connection details are announced on the mailing list
   * Past meeting materials and recordings are available on the IETF datatracker
4. **Contribute**:
   * Submit comments and suggestions on the mailing list
   * Propose changes through GitHub pull requests
   * Participate in working group discussions
   * Help with implementations and interoperability testing

For more information about the IETF standardization process and how to participate, visit: https://www.ietf.org/about/participate/
