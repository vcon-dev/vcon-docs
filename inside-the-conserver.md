---
description: The Machinery of the Conserver
---

# Inside the Conserver

The Conserver processes vCons, storing them locally, and projecting them into the third party information services.  The building blocks of the Conserver are links, which have an interface to accept a single vCon, and can then forward that vCon, or create new ones, to other links for further processing.  Links are formed into chains, designed to apply a series of analysis and transformation to the vCons.  Chains are executed by the conserver periodically on a timer, or on request from a third party system.&#x20;

<figure><img src=".gitbook/assets/Conserver Pictures (7).jpg" alt=""><figcaption><p>The System view of the Conserver</p></figcaption></figure>

## Conserver Tech Stack&#x20;

The Conserver is built off of two core platforms: a python API framework FASTAPI, and a REDIS real time database. The conserver itself is written in Python, and uses the standard vCon Python library to create and modify vCons.

REDIS is responsible for storing the conversations, while FAST API coordinates the application software that manages them. Each conversation is stored as a REDIS JSON object in the standard vCon format. In practice, each vCon is stored in REDIS by the UUID of the vCon, making them easy to discover and fast to process. Instead of copying the conversation as it’s built and transformed, it stays stored in REDIS, and the ID to the vCon is passed, optimizing processing efficiency even at very large data sizes. REDIS also provides inter task communication using a series of PUB/SUB channels, coordinating the activities of the conserver for both local software (that inside the conserver itself) but also for external software such as Lambdas or exporting onto other systems like Apache Kafka. Also, third party and hardware enabled systems can use REDIS as a data interchange system, loading and unloading large media files in coordination with the data pipeline.

FAST API provides the application infrastructure for the conserver. The transformation steps are developed as Python modules and loaded as tasks managed by FAST API. As each task finishes, it notifies other system elements by publishing UUID of the vCon. Other tasks wait on these notifications, and when they receive the notification, they can act on that same vCon for whatever purpose they may have. In addition, FAST API provides a REST API to the store of vCons, and a simple UI to manage the conserver.

## Configuration of the Conserver

Configuration is provided through a file structure that defines the environment, chains, links and storage elements.  This file is read into REDIS memory on startup, and is read from REDIS through execution, enabling for dynamic configuration without the need for system restarts.&#x20;

## Creating a vCon

The Conserver is a native vCon device.  Generally, all the operations that are internal to the conserver use vCons as their inputs and outputs.  vCons are like PDFs for conversations; the contents of each vCon is decided by the system that makes it.   Some systems are vCon native, and support the creation of vCons as they export conversations. Other systems support API access, where adapters can create the vCons using data collected through the APIs before forwarding them to the conversation.  At the time of this writing, adapters exist for Bria, Quiq, Volie, Mailgun and RingPlan.&#x20;

vCons may be created using the vCon library and recordings and meta data extracted from third party systems.  To add this vCon to the conserver, you could then either POST the vCon to the Conserver API, or you could store it directly in REDIS, then insert the vCon UUID in the desired chain's input list.

## REDIS

Each vcon is stored in REDIS using JSON and named with a regular key: vcon:\{{vcon-uuid\}},  as are chains "chains:\{{name\}}", links "link:\{{name\}}" and storages "storage:\{{name\}}}".   REDIS allows for the addition of dedicated hardware to accelerate long running and high compute use cases such as transcription and video redaction, as these systems can connect directly to REDIS relieving scale issues from general purpose hardware, while managing the overhead of moving large amounts of data.  Links take a vCon ID as inputs, and bear the responsibility of reading vCons if required, or giving them the option to hand off to optimized hardware.
