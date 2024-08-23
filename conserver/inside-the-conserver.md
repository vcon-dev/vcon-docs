---
description: The Machinery of the Conserver
---

# ❤️ Inside the Conserver

<figure><img src="../.gitbook/assets/Conserver Pictures (7).jpg" alt=""><figcaption><p>The System view of the Conserver</p></figcaption></figure>

The Conserver processes vCons, storing them locally, and projecting them into the third party information services.  The building blocks of the Conserver are links, which have an interface to accept a single vCon, and can then forward that vCon, or create new ones, to other links for further processing.  Links are formed into chains, designed to apply a series of analysis and transformation to the vCons.  Chains are executed by the conserver periodically on a timer, or on request from a third party system.&#x20;

## Links: The Fundamental Building Block

The heart of the conserver functionality is the "link".  A link is a Python module that takes a single vCon and processes it.  Chains are ultimately created by combining links in serial. All links have the same interface.  Using links has multiple advantages:

* **Configurable**: Uses a flexible options system for customization.
* **Retry Mechanism**: Implements exponential backoff for API call retries.
* **Caching**: Avoids redundant analysis by checking existing data.
* **Metrics**: Tracks performance and error metrics.
* **Modular**: Designed to be part of a larger system, likely for processing voice conversations.

As an example, let's look at [the analyze link.](https://github.com/vcon-dev/vcon-server/blob/86f26ecb1ec01877586c712921c564f6241b2d6c/server/links/analyze/\_\_init\_\_.py)  This link takes a vCon, applies a prompt to it, then adds an analysis to the vCon with the result.&#x20;

<figure><img src="../.gitbook/assets/Conserver Internals (1).jpg" alt=""><figcaption><p>A Conserver Link</p></figcaption></figure>

### Main Function: run

The `run` function is the entry point for links. [For the analysis link](https://github.com/vcon-dev/vcon-server/blob/86f26ecb1ec01877586c712921c564f6241b2d6c/server/links/analyze/\_\_init\_\_.py#L62), it performs the following steps:

1. Merges provided options in the "config.yml" file with default options.
2. Retrieves the vCon (voice conversation) object from Redis.
3. Applies inclusion filters and sampling.
4. Iterates through dialog entries in the vCon:
   * Retrieves the source text for analysis.
   * Checks if analysis already exists.
   * Generates new analysis using OpenAI if needed.
   * Adds the generated analysis to the vCon object.
5. Stores the updated vCon back in Redis.

### Default Options

A `default_options` dictionary defines the options for the link, and are overridden by the configuration file. For instance, the analysis link is defined with the following options:

* Prompt for summarization
* The value to set as the analysis type when added to the vCon (default: "summary")
* GPT model (default: "gpt-3.5-turbo-16k")
* Sampling rate and temperature
* Source configuration for transcript analysis, for instance "transcript" or "summary"

### Error Handling and Metrics

The module includes error handling for API calls and retries. It also tracks metrics such as analysis time and failures using custom metric functions.

### Return Values

Links can return one of two kinds of values.  Links can return a vCon UUID, or None.  Typically, it would be the vCon UUID that was passed in.  However, if the link created a new vCon, as would be required for creating a new, redacted vCon, the new UUID would be returned by the link.  To stop chain processing, a link could return None.  This is useful for links that filter vCons out, only allowing certain ones down the chain, and stopping the processing of links downstream of the chain.&#x20;

## Chains: Links for Workflow

The fundamental implementation of workflow is created by a series of links.  These chains take vCon uuids from REDIS lists, runs the chain of links on the vCon, stores it, then places the uuids in egress REDIS links.&#x20;

<figure><img src="../.gitbook/assets/Conserver Internals (4) (2).jpg" alt=""><figcaption><p>A Chain</p></figcaption></figure>

### Chain Processing, Link by Link

The [main loop of the conserver](https://github.com/vcon-dev/vcon-server/blob/86f26ecb1ec01877586c712921c564f6241b2d6c/server/main.py#L139) processes vCons:

* Loads configuration and sets up the ingress chain map.
* Enters a loop that continuously checks for new items in the ingress lists using Redis.
* When an item (vCon ID) is found, it creates a VconChainRequest and processes it.
* Handles exceptions by moving problematic vCons to a Dead Letter Queue.

Step by step:&#x20;

1. Processing starts when vCon UUIDs are placed into a ingress list.  Chains may have several ingress lists, and have to have at least one to kick off processing.  Lists are implemented as REDIS lists, and processing is controlled at the thread layer by blocking until the a new element is placed on the list.  UUIDs can be added to the ingress list by other chains, allowing them to be placed in series, from links that can request processing, or from the API.  A typical pattern is to create the vCon using the API, then inserting the UUID into the desired ingress list.
2.  For each vCon taken from the ingress list, it is processed by each link in the chain.   This `_process_link` function handles the execution of a single link in the processing chain for a vCon. Here's a summary of its functionality:



    This enables flexible and dynamic execution of different processing steps (links) in the vCon processing chain, with built-in logging and timing measurements.

    1. It logs the start of processing for the specific link and vCon.
    2. It retrieves the link configuration from the global `config`.
    3. It dynamically imports the module specified for this link if it hasn't been imported before, caching it for future use.
    4. It retrieves any options specified for the link.
    5. It logs the execution of the link's module.
    6. It measures the execution time of the module's `run` method, which is called with the vCon ID, link name, and options.
    7. After execution, it logs the completion of the link processing, including the time taken.
    8. Finally, it returns the result from the module's `run` method, which determines whether the chain should continue processing or stop.
3. After the links have been processed, assuming that none of the links returned None, the vCon UUID is pushed into the chain's egress lists.  Finally, the vCon is then stored in the storages (S3, Mongo, File, etc.) specified for that link.&#x20;
4. In case of an error in any of these links, the vCon UUID will be pushed into the dead letter queue of the original ingress list.

## Tech Stack&#x20;

The Conserver is built off of two core platforms: a python API framework FASTAPI, and a REDIS real time database. The conserver itself is written in Python, and uses the standard vCon Python library to create and modify vCons.

REDIS is responsible for storing the conversations, while FAST API coordinates the application software that manages them. Each conversation is stored as a REDIS JSON object in the standard vCon format. In practice, each vCon is stored in REDIS by the UUID of the vCon, making them easy to discover and fast to process. Instead of copying the conversation as it’s built and transformed, it stays stored in REDIS, and the ID to the vCon is passed, optimizing processing efficiency even at very large data sizes. REDIS also provides inter task communication using a series of PUB/SUB channels, coordinating the activities of the conserver for both local software (that inside the conserver itself) but also for external software such as Lambdas or exporting onto other systems like Apache Kafka. Also, third party and hardware enabled systems can use REDIS as a data interchange system, loading and unloading large media files in coordination with the data pipeline.

Each vcon is stored in REDIS using JSON and named with a regular key: vcon:\{{vcon-uuid\}},  as are chains "chains:\{{name\}}", links "link:\{{name\}}" and storages "storage:\{{name\}}}".   REDIS allows for the addition of dedicated hardware to accelerate long running and high compute use cases such as transcription and video redaction, as these systems can connect directly to REDIS relieving scale issues from general purpose hardware, while managing the overhead of moving large amounts of data.  Links take a vCon ID as inputs, and bear the responsibility of reading vCons if required, or giving them the option to hand off to optimized hardware.

FAST API provides the application infrastructure for the conserver. The transformation steps are developed as Python modules and loaded as tasks managed by FAST API. As each task finishes, it notifies other system elements by publishing UUID of the vCon. Other tasks wait on these notifications, and when they receive the notification, they can act on that same vCon for whatever purpose they may have. In addition, FAST API provides a REST API to the store of vCons, and a simple UI to manage the conserver.

