---
description: Configuring Conserver Links
---

# Configuring Conserver Links

Links are extensibility points to the conserver.
A conserver is configured with one or more links.
A collection of links construct a chain, which is executed in sequence.

Configuration is saved in the `vcon-server/config.yml` file.

## links:

A collection of links, loaded from the folder specified in `module:`, and configured under the `options:` node.

## storage:

Storage drivers for vCons

## chains:

The sequence of links to execute when processing a vCon

### ingress_lists:

The queue to monitor for processing by the link

### egress_lists:

The queue to move the vcon upon successful processing.
Chains can be sequenced by configuring the `egress_lists:` to reference the same name as another `chain`'s `ingress_lists`.

  ```yaml
  links:
    <link-name>:
      module: links.<link-folder>
      options:
        "name": "value-pairs"
        "instructions": "in each link folder"
  expire_vcon:
    module: links.expire_vcon
    options:
      seconds: 604800
  summarize:
    module: links.analyze
    options:
      OPENAI_API_KEY: xxxxx
      prompt: "Summarize this transcript in a few sentences, identify the purpose and the parties of the conversation. Mention if there was a voicemail or if the customer and agent spoke."
      analysis_type: summary
      model: 'gpt-4o-mini'
  webhook_store_call_log:
    module: links.webhook
    options:
      webhook-urls:
        - https://example.com/conserver
  storages:

  chains:
    my-chain:
      links:
        - expire_vcon
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
    notification-chain:
      links:
        - webhook_store_call_log
      ingress_lists:
        - default_egress
      storages:
      egress_lists:
        - end-queue
      enabled: 1
  ```
