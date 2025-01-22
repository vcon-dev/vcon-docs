---
description: A Complete Guide
icon: wrench
---

# Configuring the Conserver

The conserver's configuration file is the heart of how the system operates, defining how conversations flow through the system and how they are processed and stored. Let's break down each major component and how to configure them.

### Configuration File Location

The configuration file location is specified in the environment (I use the .env file), typically at config.yml in the vcon-server root.

### Configuration File Structure

The configuration file is a YAML document with several main sections:

* **links**: Defines the processing modules available to the system
* **storages**: Specifies where vCons can be stored
* **chains**: Defines the workflow pipelines
* **followers**: Configures how the system can follow other conservers

Let's explore each section in detail:

#### Configuring Links

Links are the processing units of the conserver. Each link is a module that performs a specific operation on a vCon. Here's how to configure a link:

```yaml
links:
  deepgram:
    module: links.deepgram
    options:
      DEEPGRAM_KEY: your_key_here
      minimum_duration: 30
      api:
        model: "nova-2"
        smart_format: true
        detect_language: true

  analyze:
    module: links.analyze
    options:
      OPENAI_API_KEY: your_key_here
      prompt: "Summarize this transcript"
      analysis_type: summary
      model: 'gpt-4'
```

Each link configuration needs:

* A unique name (e.g., 'deepgram', 'analyze')
* The module path that implements the link functionality
* An options dictionary containing the link's specific configuration

#### Configuring Storages

Storages define where vCons are saved after processing. The conserver supports multiple storage backends:

```yaml
storages:
  postgres:
    module: storage.postgres
    options:
      user: postgres
      password: your_password
      host: your_host
      port: "5432"
      database: postgres

  s3:
    module: storage.s3
    options:
      aws_access_key_id: your_key_id
      aws_secret_access_key: your_secret
      aws_bucket: your_bucket
```

Each storage needs:

* A unique name
* The storage module implementation
* Connection and authentication options specific to the storage type

#### Configuring Chains

Chains are where you define your processing workflows. They connect links together and specify where the results should be stored:

```yaml
chains:
  transcription_chain:
    links:
      - deepgram
      - analyze
      - webhook_store
    ingress_lists:
      - transcription_input
    storages:
      - postgres
      - s3
    egress_lists:
      - transcription_output
    enabled: 1
    timeout: 300
```

A chain configuration includes:

* The links to execute, in order
* Input lists (ingress\_lists) where new vCons arrive
* Storage locations for the processed vCons
* Output lists (egress\_lists) for downstream processing
* An enabled flag and optional timeout

#### Configuring Followers

Followers allow one conserver to monitor and process vCons from another conserver:

```yaml
followers:
  remote_conserver:
    url: "https://remote-conserver.example.com"
    auth_token: "your_auth_token"
    egress_list: "remote_output"
    follower_ingress_list: "local_input"
    pulling_interval: 60
    fetch_vcon_limit: 10
```

Each follower needs:

* The URL of the remote conserver
* Authentication credentials
* The remote list to monitor (egress\_list)
* The local list to populate (follower\_ingress\_list)
* Polling configuration (interval and batch size)

### Configuration Best Practices

When configuring your conserver:

1. Use meaningful names for your chains, links, and storage configurations to make the system easier to understand and maintain.
2. Consider your processing pipeline carefully - organize links in a logical order where each step builds on the previous ones.
3. Use multiple storage backends when needed - for example, storing in both S3 for long-term storage and Postgres for quick querying.
4. Configure appropriate timeouts for your chains based on the expected processing time of your links.
5. Use the follower configuration when you need to process vCons across multiple conserver instances, creating distributed processing pipelines.

The configuration file is loaded by the system at startup and can be updated via the API endpoint `/config`. The system will use the new configuration immediately after updating.

Remember that the conserver uses Redis as its working storage, so all the lists referenced in ingress\_lists and egress\_lists are Redis lists. The actual vCons are stored in Redis using JSON data types, making them quickly accessible during processing.

This configuration system provides a flexible way to define complex processing pipelines for your vCons while keeping the configuration clear and maintainable.
