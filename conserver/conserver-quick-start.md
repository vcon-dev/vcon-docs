---
description: A quick start to getting the conserver up and running
---

# 🐰 Conserver Quick Start

The quickstart will cover:

- [Installing the conserver with Docker](#ubuntu-install-wdocker)
- [Viewing the default configuration](#conserver-configuration)
- [Staring a conserver for processing](#start-the-conserver)
- [Monitoring realtime logs](#viewing-logs-realtime)
- [Submitting a vCon](#submit-a-vcon)
- [Process a vCon](#process-a-chain-of-vcons)

## Ubuntu Install w/Docker

- Install the [Docker client](https://www.docker.com/get-started/), or equivalent

  ```bash
  snap install docker
  ```

- Clone the vCon Repos

  ```bash
  git clone https://github.com/vcon-dev/vcon.git
  cd vcon/
  git submodule sync
  git submodule update --init --recursive
  ```

## Conserver Configuration

A conserver has two configuration files:

- Service Host & Secret Configuration: `vcon-server/.env`
- Service Processing Configuration: `vcon-server/config.yml`

The conserver repo (`/vcon-server`) can be downloaded directly, but is also included in the `vcon` repo cloned above as a sub-repo in the vcon-server directory.

- Configure secrets in the `vcon_server/.env` directory.  
  For the purposes of getting started in a local container, no changes are needed.

  **Example `vcon-server/.env` file**

  ```bash
  # CORE DEPENDENCIES
  ENV=dev
  HOSTNAME=http://0.0.0.0:8000
  HOST=0.0.0.0
  PORT=8000
  REDIS_URL=redis://redis

  # Overriding these on pairing so they don't conflict with django port etc
  REDIS_EXTERNAL_PORT=8001
  CONSERVER_EXTERNAL_PORT=8000

  CONSERVER_API_TOKEN=1111111
  CONSERVER_CONFIG_FILE=./config.yml
  ```

- View the Conserver config file in the server directory.
  See [Configuring Links](./configuring-links.md) for more details.

  **Example `vcon-server/config.yml`**

  ```yaml
  links:
    deepgram:
      module: links.deepgram
      options:
        DEEPGRAM_KEY: xxxxxxxxxxxx
        minimum_duration: 30
        api:
          model: "nova-2"
          smart_format: true  
          detect_language: true
    diarize:
      module: links.analyze
      options:
        OPENAI_API_KEY: xxxx
        prompt: "Go step by step: 1. Diarize the conversation and also identify the Agent and the Customer and show the names along with it like Agent(Agent Name) and output in markdown and label each speaker in bold. Don't add any extra information except for the speakers. Don't add the word markdown. 2. If it's only one speaker, return the transcript. 3. If you can't diarize the transcript, return an empty string."
        analysis_type: diarized
        model: 'gpt-4o'
    sentiment:
      module: links.analyze
      options:
        OPENAI_API_KEY: xxxx
        prompt: "Based on this transcript - if the customer complained, if the customer said they were angry or disappointed, if the customer threatened or used profanity, respond with only the words 'NEEDS REVIEW', otherwise respond 'NO REVIEW NEEDED'."
        analysis_type: customer_frustration
        model: 'gpt-4o-mini'
    summarize:
      module: links.analyze
      options:
        OPENAI_API_KEY: xxxxx
        prompt: "Summarize this transcript in a few sentences, identify the purpose and the parties of the conversation. Mention if there was a voicemail or if the customer and agent spoke."
        analysis_type: summary
        model: 'gpt-4o-mini'
  storages:

  chains:
    my-chain:
      links:
        - deepgram
        - diarize
        - sentiment
        - summarize
      ingress_lists:
        - default_ingress
      storages:
      egress_lists:
        - default_egress
      enabled: 1

    summarize_only:
      links:
      - summarize
      ingress_lists:
        - summarize_ingress
      egress_lists:
        - summarize_egress
  ```

### Start the Conserver

Using a local Docker configuration, start

```bash
docker network create conserver
docker compose up -d
```

### Troubleshooting and Checking

You can validate that the conserver is running on the command line using `docker ps`.  
In the example below, we can see:

- `vcon-server-conserver-1`: processes vcons
- `vcon-server-api-1`: ingests requests
- `vcon-server-redis-1`: default storage and persistance

```output
# docker ps
CONTAINER ID   IMAGE                      COMMAND                  CREATED         STATUS                   PORTS                                                 NAMES
ffe6f68941c8   vcon-server-conserver      "/app/docker/wait_fo…"   5 minutes ago   Up 5 minutes                                                                   vcon-server-conserver-1
8136e15912c5   vcon-server-api            "/app/docker/wait_fo…"   5 minutes ago   Up 5 minutes             0.0.0.0:8000->8000/tcp, :::8000->8000/tcp             vcon-server-api-1
e3388b5f23be   redis/redis-stack:latest   "/entrypoint.sh"         5 minutes ago   Up 5 minutes (healthy)   6379/tcp, 0.0.0.0:8001->8001/tcp, :::8001->8001/tcp   vcon-server-redis-1
```

## Viewing Logs Realtime

Monitor operational logs using `docker compose logs -f`.  
A typical log:

```output
vcon-server-redis-1      | 9:C 23 Aug 2024 17:27:20.581 # WARNING Memory overcommit must be enabled! Without it, a background save or replication may fail under low memory condition. Being disabled, it can also cause failures without low memory condition, see https://github.com/jemalloc/jemalloc/issues/1328. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
vcon-server-redis-1      | 9:C 23 Aug 2024 17:27:20.582 * oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
vcon-server-redis-1      | 9:C 23 Aug 2024 17:27:20.582 * Redis version=7.4.0, bits=64, commit=00000000, modified=0, pid=9, just started
vcon-server-redis-1      | 9:C 23 Aug 2024 17:27:20.582 * Configuration loaded
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.582 * Increased maximum number of open files to 10032 (it was originally set to 1024).
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.583 * monotonic clock: POSIX clock_gettime
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.584 * Running mode=standalone, port=6379.
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.586 * Module 'RedisCompat' loaded from /opt/redis-stack/lib/rediscompat.so
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.614 * <search> Redis version found by RedisSearch : 7.4.0 - oss
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.616 * <search> RediSearch version 2.10.5 (Git=2.10-e2f28a9)
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.616 * <search> Low level api version 1 initialized successfully
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.617 * <search> gc: ON, prefix min length: 2, min word length to stem: 4, prefix max expansions: 200, query timeout (ms): 500, timeout policy: return, cursor read size: 1000, cursor max idle (ms): 300000, max doctable size: 1000000, max number of search results:  10000, 
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.620 * <search> Initialized thread pools!
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.621 * <search> Enabled role change notification
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.621 * Module 'search' loaded from /opt/redis-stack/lib/redisearch.so
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.630 * <timeseries> RedisTimeSeries version 11202, git_sha=5643fd4d6fcb1e9cf084fb2deb9285b08f4a6672
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.631 * <timeseries> Redis version found by RedisTimeSeries : 7.4.0 - oss
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.631 * <timeseries> loaded default CHUNK_SIZE_BYTES policy: 4096
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.631 * <timeseries> loaded server DUPLICATE_POLICY: block
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.631 * <timeseries> loaded default IGNORE_MAX_TIME_DIFF: 0
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.631 * <timeseries> loaded default IGNORE_MAX_VAL_DIFF: 0.000000
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.631 * <timeseries> Setting default series ENCODING to: compressed
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.631 * <timeseries> Detected redis oss
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.631 * Module 'timeseries' loaded from /opt/redis-stack/lib/redistimeseries.so
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.639 * <ReJSON> Created new data type 'ReJSON-RL'
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.639 * <ReJSON> version: 20803 git sha: unknown branch: unknown
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.639 * <ReJSON> Exported RedisJSON_V1 API
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.639 * <ReJSON> Exported RedisJSON_V2 API
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.639 * <ReJSON> Exported RedisJSON_V3 API
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.639 * <ReJSON> Exported RedisJSON_V4 API
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.639 * <ReJSON> Exported RedisJSON_V5 API
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.639 * <ReJSON> Enabled diskless replication
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.639 * Module 'ReJSON' loaded from /opt/redis-stack/lib/rejson.so
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.639 * <search> Acquired RedisJSON_V5 API
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.641 * <bf> RedisBloom version 2.8.2 (Git=unknown)
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.641 * Module 'bf' loaded from /opt/redis-stack/lib/redisbloom.so
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.648 * <redisgears_2> Created new data type 'GearsType'
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.650 * <redisgears_2> Detected redis oss
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.652 # <redisgears_2> could not initialize RedisAI_InitError
vcon-server-redis-1      | 
vcon-server-redis-1      | 
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.652 * <redisgears_2> Failed loading RedisAI API.
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.652 * <redisgears_2> RedisGears v2.0.20, sha='9b737886bf825fe29ddc2f8da81f73cbe0b4e858', build_type='release', built_for='Linux-ubuntu22.04.x86_64', redis_version:'7.4.0', enterprise:'false'.
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.657 * <redisgears_2> Registered backend: js.
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.657 * Module 'redisgears_2' loaded from /opt/redis-stack/lib/redisgears.so
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.657 * Server initialized
vcon-server-redis-1      | 9:M 23 Aug 2024 17:27:20.657 * Ready to accept connections tcp
vcon-server-conserver-2  | Redis is ready!
vcon-server-conserver-2  | Redis is ready. Starting the dependent service...
vcon-server-conserver-2  | {"asctime": "2024-08-23 17:28:24,696", "levelname": "INFO", "name": "__main__", "message": "Starting main loop", "taskName": null}
vcon-server-conserver-4  | Redis is ready!
vcon-server-conserver-4  | Redis is ready. Starting the dependent service...
vcon-server-conserver-4  | {"asctime": "2024-08-23 17:28:24,545", "levelname": "INFO", "name": "__main__", "message": "Starting main loop", "taskName": null}
vcon-server-conserver-3  | Redis is ready!
vcon-server-conserver-3  | Redis is ready. Starting the dependent service...
vcon-server-conserver-3  | {"asctime": "2024-08-23 17:28:25,041", "levelname": "INFO", "name": "__main__", "message": "Starting main loop", "taskName": null}
vcon-server-api-1        | Redis is ready!
vcon-server-api-1        | Redis is ready. Starting the dependent service...
vcon-server-api-1        | Skipping virtualenv creation, as specified in config file.
vcon-server-api-1        | {"asctime": "2024-08-23 17:27:24,198", "levelname": "INFO", "name": "server.api", "message": "Api starting up", "taskName": "Task-1"}
vcon-server-api-1        | {"asctime": "2024-08-23 17:27:24,226", "levelname": "INFO", "name": "uvicorn.error", "message": "Started server process [1]", "taskName": "Task-1", "color_message": "Started server process [\u001b[36m%d\u001b[0m]"}
vcon-server-api-1        | {"asctime": "2024-08-23 17:27:24,226", "levelname": "INFO", "name": "uvicorn.error", "message": "Waiting for application startup.", "taskName": "Task-1"}
vcon-server-api-1        | {"asctime": "2024-08-23 17:27:24,227", "levelname": "INFO", "name": "uvicorn.error", "message": "Application startup complete.", "taskName": "Task-1"}
vcon-server-api-1        | {"asctime": "2024-08-23 17:27:24,227", "levelname": "INFO", "name": "uvicorn.error", "message": "Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)", "taskName": "Task-1", "color_message": "Uvicorn running on \u001b[1m%s://%s:%d\u001b[0m (Press CTRL+C to quit)"}
vcon-server-conserver-1  | Redis is ready!
vcon-server-conserver-1  | Redis is ready. Starting the dependent service...
vcon-server-conserver-1  | {"asctime": "2024-08-23 17:27:22,240", "levelname": "INFO", "name": "__main__", "message": "Starting main loop", "taskName": null}
```

## Submit a vCon

## Process a Chain of vCons

## Postman Configuration

The [vCon admin program](https://github.com/vcon-dev/vcon-admin) is a nice tool for managing the conserver.
