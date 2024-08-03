# 🏫 Concepts

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
