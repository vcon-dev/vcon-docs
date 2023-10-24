---
description: A quick start to getting the conserver up and running
---

# 🛠 Quick Start

## Ubuntu Install&#x20;

Based on a digital ocean install, to keep it vanilla. Created a 4 GB Memory / 2 Intel vCPUs / 120 GB Disk / NYC3 - Ubuntu 23.04 x64 droplet, logged in.

```
snap install docker
git clone https://github.com/vcon-dev/vcon.git
cd vcon/
```

Create an \~/vcon/.env file for some of the global environmental stuff.  See example .env below.

## Example .env file

```
AWS_BUCKET=vcon-storage
AWS_KEY_ID=AKxxxxxxxxxxx
AWS_SECRET_KEY=xxxxxxx/xxxxxxx
DEEPGRAM_KEY=xxxxxxxx
ENV=dev

# CORE DEPENDENCIES
ENV=dev
HOSTNAME=http://0.0.0.0:8000
HOST=0.0.0.0
PORT=8000
REDIS_URL=redis://redis

# Overriding these on pairing so they don't conflict with django port etc
REDIS_EXTERNAL_PORT=8001
CONSERVER_EXTERNAL_PORT=8000

CONSERVER_API_TOKEN=xxxxxxxxxx
CONSERVER_CONFIG_FILE=./config.yml

```

```
cd vcon
```

Create a new config file in the server directory, then docker compose up.

## Example ./config.yml

```
environment:
  HOSTNAME: https://localhost:8000
  STITCHER_DATABASE_URL: ''
  SLACK_TOKEN: ''
  OPENAI_API_KEY: ''
links:
  transcribe:
    module: links.transcribe
    ingress-lists: []
    egress-lists: []
    options:
      transcribe_options:
        model_size: base
        output_options:
        - vendor
  script:
    module: links.script
    ingress-lists: []
    egress-lists: []
  summary:
    module: links.summary
    ingress-lists: []
    egress-lists: []
  tag:
    module: links.tag
    ingress-lists: []
    egress-lists: []
    options:
      tags:
      - Geddy
      - Alex
      - The Professor
  webhook:
    module: links.webhook
    options:
      webhook-urls:
      - notreal.com
storages:
  mongo:
    module: storage.mongo
    options:
      MONGO_URL: mongodb://localhost:27017/
      database: conserver
      collection: vcons
  postgres:
    module: storage.postgres
    options:
      user: thomashowe
      password: postgres
      host: localhost
      port: '5432'
      database: postgres
  s3:
    module: storage.s3
    options:
      aws_access_key_id: some_key
      aws_secret_access_key: some_secret
      aws_bucket: some_bucket
chains:
  sample_chain:
    links:
    - transcribe
    - script
    - summary
    - tag
    ingress_lists:
    - sample_list
    storages:
    - mongo
    - postgres
    - s3
    egress_lists:
    - test_output
    enabled: 1
    
```

```
docker compose up
```
