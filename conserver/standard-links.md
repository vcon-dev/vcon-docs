---
description: Reference for the 23 links that ship with vcon-server, with the options each reads, the defaults in the code, and what each writes to the vCon.
---

# 🔗 Standard Links

A link does one thing to a vCon as it moves through a chain. The conserver ships 23. The option tables below come from each module's `default_options` at vcon-server `b603b15`. A key you leave out takes the default shown. For what a link's return value does, and the order a chain runs in, see [Concepts](concepts.md).

| Category | Links |
| -------- | ----- |
| Transcription | `transcribe`, `wtf_transcribe`, and the deprecated aliases `deepgram_link`, `groq_whisper`, `hugging_face_whisper`, `openai_transcribe` |
| Analysis | `analyze`, `analyze_vcon`, `analyze_and_label`, `check_and_tag`, `detect_engagement`, `hugging_llm_link` |
| Routing and filtering | `sampler`, `jq_link`, `tag_router` |
| Data management | `tag`, `diet`, `expire_vcon` |
| Integration | `webhook`, `post_analysis_to_slack` |
| Audit | `scitt`, `datatrails` |
| Testing | `delay` |

## Conventions

**Interface.** Every link exposes `run(vcon_uuid, link_name, opts)`. It returns a UUID to continue, a falsy value to stop the chain, or raises to send the vCon to the dead letter queue. [Creating Custom Links](creating-custom-links.md) covers writing one.

**Secrets.** Provider keys go in the link's `options`. The conserver does not expand `${VAR}` in `config.yml` and does not read provider keys from the environment, apart from two defaults noted under `detect_engagement` and `groq_whisper`. See [Configuring the Conserver](configuring-the-conserver.md#secrets). The links that record their options in `vendor_schema` drop any option whose name contains `key`, `token`, `secret`, `password`, `credential` or `proxy_url`.

**OpenAI-backed links.** `analyze`, `analyze_vcon`, `analyze_and_label`, `check_and_tag`, `detect_engagement` and `openai_transcribe` all build their client from the same options. The first group present wins:

| Provider | Options |
| -------- | ------- |
| LiteLLM proxy | `LITELLM_PROXY_URL` and `LITELLM_MASTER_KEY` |
| Azure OpenAI | `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_API_KEY`, and `AZURE_OPENAI_API_VERSION` (default `2024-10-21`) |
| OpenAI | `OPENAI_API_KEY` (`api_key` and `openai_api_key` also work), with optional `organization` and `project` |

With none of them set the link raises, except `detect_engagement`, which logs a warning and passes the vCon on. `analyze` records the provider as the analysis `vendor` (`openai`, `azure`, or for LiteLLM a name inferred from the model). The other links record `openai`. The links also accept `send_ai_usage_data_to_url` and `ai_usage_api_token`, which post token counts to an endpoint you run.

**Shared analysis options.** The analysis links `analyze`, `analyze_vcon`, `analyze_and_label`, `check_and_tag` and `detect_engagement` also accept:

| Option | Description |
| ------ | ----------- |
| `sampling_rate` | Probability from 0 to 1 that the link runs on a given vCon. Default `1`. Skipped vCons continue down the chain. |
| `only_if` | Run only if the vCon matches. `{section: "analysis" or "attachments", type: <name>, includes: <text>}`. `purpose` works as an alias of `type`. The link runs when an element of that section has that type and its body contains the text. For `type: tags` the text must equal one whole tag, such as `priority:high`. |
| `source` | Which analysis to read, as `{analysis_type, text_location}`. The default `body.paragraphs.transcript` suits Deepgram with `smart_format`. For `openai_transcribe`, `groq_whisper` and `hugging_face_whisper` set `text_location: body.text`. For `wtf_transcribe` set `analysis_type: wtf_transcription` and `text_location: body.transcript.text`. |

These links work one dialog at a time. They skip a dialog that has no source analysis, and they skip a dialog that already has an analysis of their `analysis_type`, so a rerun does not duplicate.

**Telemetry.** Each link runs inside a `link.<name>` span and the conserver counts it in `conserver.link.count` and times it in `conserver.link.execution_time`. Many links add their own counters under `conserver.link.<family>.*`. The containers export through `opentelemetry-instrument` and the `OTEL_EXPORTER_OTLP_*` variables. See [Production Deployment](production-deployment.md#metrics).

## Transcription links

These turn recordings in `dialog[]` into `analysis[]` entries of type `transcript`. They process dialogs of type `recording`, skip dialogs shorter than `minimum_duration`, and skip dialogs that already have a transcript. `openai_transcribe` and `deepgram_link` need a dialog `url` and still transcribe a recording that has no `duration`. `groq_whisper` and `hugging_face_whisper` read the dialog `duration` without a default, so a recording without one raises, and they also accept an inline `body`.

***

#### transcribe

The canonical transcription link. It sends the vCon to one vendor, chosen by `vendor`.

```yaml
links:
  transcribe_dg:
    module: links.transcribe
    options:
      vendor: deepgram
      vendor_options:
        DEEPGRAM_KEY: "your-deepgram-key"
        minimum_duration: 30
        api:
          model: nova-2
          smart_format: true
```

| Option | Description | Default |
| ------ | ----------- | ------- |
| `vendor` | `openai`, `groq`, `hugging_face`, `deepgram` or `whisper_builtin`. Any other value raises. | `whisper_builtin` |
| `vendor_options` | Options passed unchanged to the vendor. Each vendor's table is below. Your value replaces the default as a whole. | `{model_size: base, output_options: [vendor]}` |
| `transcribe_options` | Legacy name for `vendor_options`, used when `vendor_options` is absent. | (none) |

`whisper_builtin` calls `Vcon.transcribe(**vendor_options)` from the vcon library and runs Whisper inside the worker. The other vendors call the modules documented next.

***

#### openai\_transcribe

Deprecated alias. The conserver rewrites `module: links.openai_transcribe` to `links.transcribe` with `vendor: openai` and logs one warning. New configs should use `transcribe`. Transcribes with an OpenAI-compatible speech API and splits long audio into chunks.

| Option | Description | Default |
| ------ | ----------- | ------- |
| `OPENAI_API_KEY`, Azure and LiteLLM options | See [Conventions](#conventions) | (none) |
| `model` | Transcription model | `gpt-4o-transcribe` |
| `minimum_duration` | Seconds of audio below which a dialog is skipped | `3` |
| `max_chunk_duration` | Longest chunk in seconds | `480` |
| `use_silence_chunking` | Cut chunks at silence | `true` |
| `silence_thresh` | Silence threshold in dBFS | `-40` |
| `silence_len` | Shortest silence to cut at, in ms | `2000` |

`language` appears in the defaults, but the code never reads it. The analysis has `vendor: openai` and a body with `text`. The chunk results are kept under `chunked_transcription` when the audio was split.

***

#### groq\_whisper

Deprecated alias for `vendor: groq`. Transcribes with Groq's hosted Whisper.

| Option | Description | Default |
| ------ | ----------- | ------- |
| `API_KEY` | Groq key. If you omit it, the module uses `GROQ_API_KEY` from the environment as read at import, else a placeholder string that fails. | `$GROQ_API_KEY` |
| `minimum_duration` | Seconds of audio below which a dialog is skipped | `30` |
| `Content-Type` | In the defaults, not used by the Groq call | `audio/flac` |

The model is fixed in the code as `whisper-large-v3-turbo`. There is no `model` option. At import the module removes `HTTP_PROXY`, `HTTPS_PROXY` and `NO_PROXY` from the process environment, which affects every other link in the same worker. The analysis has `vendor: groq_whisper` and a body with `text`.

***

#### hugging\_face\_whisper

Deprecated alias for `vendor: hugging_face`. Posts audio to a Hugging Face Inference Endpoint that runs Whisper.

| Option | Description | Default |
| ------ | ----------- | ------- |
| `API_URL` | Your endpoint URL | a placeholder, required |
| `API_KEY` | The token only. The link adds `Bearer ` itself, so do not include it | a placeholder, required |
| `Content-Type` | Audio type sent to the endpoint | `audio/flac` |
| `minimum_duration` | Seconds of audio below which a dialog is skipped | `30` |

***

#### deepgram\_link

Deprecated alias for `vendor: deepgram`. Transcribes with Deepgram, directly or through a LiteLLM proxy.

```yaml
links:
  deepgram:
    module: links.transcribe
    options:
      vendor: deepgram
      vendor_options:
        DEEPGRAM_KEY: "your-deepgram-key"
        api:
          model: nova-2
          smart_format: true
          detect_language: true
```

| Option | Description | Default |
| ------ | ----------- | ------- |
| `DEEPGRAM_KEY` | Deepgram key. Required unless the LiteLLM options are set | (none) |
| `api` | Keyword arguments for Deepgram's `PrerecordedOptions`. Include it on the direct path, even as `{}`, because the code reads the key | (none) |
| `minimum_duration` | Seconds of audio below which a dialog is skipped. A missing duration on a `.wav` URL is read from the file | `60` |
| `minimum_confidence` | A transcript below this confidence is discarded | `0.5` |
| `LITELLM_PROXY_URL`, `LITELLM_MASTER_KEY`, `model` | Send audio through a LiteLLM proxy instead. `model` defaults to `nova-3`. No confidence is returned, so the threshold is skipped | (none) |

The analysis has `vendor: deepgram` and the first Deepgram alternative as its body, with `detected_language` added.

***

#### wtf\_transcribe

Sends each recording to a `vfun` transcription server and stores the result as a [WTF](../extensions/wtf-transcription.md) analysis.

```yaml
links:
  wtf:
    module: links.wtf_transcribe
    options:
      vfun-server-url: "https://vfun.example.com/transcribe"
      api-key: "your-vfun-key"
      language: en
      diarize: true
```

| Option | Description | Default |
| ------ | ----------- | ------- |
| `vfun-server-url` | Transcription endpoint. If it is missing the link logs an error and returns `None`, which halts the chain | (required) |
| `api-key` | Sent as a bearer token | `None` |
| `language` | Language hint sent to the server. It is also written to `transcript.language` | `None` |
| `diarize` | Ask for speaker labels | `false` |
| `vfun-timeout` | Seconds to wait for the server | `300` |
| `url-timeout` | Seconds to wait when fetching the audio | `60` |

The analysis has `type: wtf_transcription`, `vendor: vfun`, `schema: wtf-1.0`, `mediatype: application/json` and the server's WTF document as `body`. A dialog that fails to transcribe is logged and skipped, and the chain continues. See the extension page for the body shape.

***

## Analysis links

These send transcript text, or the whole vCon, to a language model and store the answer.

***

#### analyze

Runs a prompt over each dialog's transcript and stores the reply as text.

```yaml
links:
  summarize:
    module: links.analyze
    options:
      OPENAI_API_KEY: "sk-..."
      prompt: "Summarize this transcript in three bullet points."
      analysis_type: summary
      model: gpt-4o-mini
      source:
        analysis_type: transcript
        text_location: body.text
```

| Option | Description | Default |
| ------ | ----------- | ------- |
| `prompt` | Instruction placed before the transcript | `""` |
| `analysis_type` | `type` of the stored analysis | `summary` |
| `model` | Chat model | `gpt-3.5-turbo-16k` |
| `temperature` | Sampling temperature | `0` |
| `system_prompt` | System message | `You are a helpful assistant.` |
| `sampling_rate`, `source`, `only_if`, provider options | See [Conventions](#conventions) | |

The analysis body is the reply as a string, with `encoding: none`, and `vendor_schema` records the model and prompt.

***

#### analyze\_vcon

Sends the whole vCon to the model and stores the JSON it returns.

| Option | Description | Default |
| ------ | ----------- | ------- |
| `prompt` | Instruction placed before the vCon JSON | `Analyze this vCon and return a JSON object with your analysis.` |
| `analysis_type` | `type` of the stored analysis | `json_analysis` |
| `model` | Chat model. The request asks for a JSON object | `gpt-3.5-turbo-16k` |
| `temperature` | Sampling temperature | `0` |
| `system_prompt` | System message | `You are a helpful assistant that analyzes conversation data and returns structured JSON output.` |
| `remove_body_properties` | Drop `body` from every dialog before sending, to save tokens | `true` |
| `sampling_rate`, `only_if`, provider options | See [Conventions](#conventions) | |

It stores one analysis on dialog `0`, with the parsed JSON as the body. If the vCon already has an analysis of that type, the link does nothing. A reply that is not valid JSON raises.

***

#### analyze\_and\_label

Asks the model for labels and applies each one as a tag.

| Option | Description | Default |
| ------ | ----------- | ------- |
| `prompt` | Instruction. It must ask for a JSON object with a `labels` array | `Analyze this transcript and provide a list of relevant labels for categorization. Return your response as a JSON object with a single key 'labels' containing an array of strings.` |
| `analysis_type` | `type` of the stored analysis | `labeled_analysis` |
| `model` | Chat model | `gpt-4-turbo` |
| `temperature` | Sampling temperature | `0.2` |
| `response_format` | Passed to the API | `{type: json_object}` |
| `sampling_rate`, `source`, `only_if`, provider options | See [Conventions](#conventions) | |

Each label `x` becomes the tag `x:x`. The system message is fixed in the code. If the reply is not valid JSON, the raw text is stored as the analysis and no tags are added.

***

#### check\_and\_tag

Asks the model a yes or no question about the transcript and adds a tag when the answer is yes.

```yaml
links:
  check_complaint:
    module: links.check_and_tag
    options:
      OPENAI_API_KEY: "sk-..."
      tag_name: complaint
      tag_value: detected
      evaluation_question: "Does the customer make a complaint?"
      source:
        analysis_type: transcript
        text_location: body.text
```

| Option | Description | Default |
| ------ | ----------- | ------- |
| `tag_name`, `tag_value`, `evaluation_question` | All three are required. The link raises if one is missing | (none) |
| `analysis_type` | `type` of the stored record of the decision | `tag_evaluation` |
| `model` | Chat model | `gpt-5` |
| `response_format` | Passed to the API | `{type: json_object}` |
| `source` | Default `text_location` is `body`, which expects a plain string. Set it for your transcript | `{analysis_type: transcript, text_location: body}` |
| `verbosity`, `minimal_reasoning` | In the defaults. The current code does not send them to the model | `low`, `true` |
| `sampling_rate`, `only_if`, provider options | See [Conventions](#conventions) | |

The stored analysis body is `{link_name, tag: "<name>:<value>", applies: true|false}`, whether or not the tag was added.

***

#### detect\_engagement

Decides whether both sides of the conversation spoke, and tags the result.

| Option | Description | Default |
| ------ | ----------- | ------- |
| `prompt` | Question. The model must answer `true` or `false` | `Did both the customer and the agent speak? Respond with 'true' if yes, 'false' if not. Respond with only 'true' or 'false'.` |
| `analysis_type` | `type` of the stored analysis | `engagement_analysis` |
| `model` | Model, called through the OpenAI Responses API | `gpt-4.1` |
| `temperature` | Sampling temperature | `0.2` |
| `OPENAI_API_KEY` | Defaults to the `OPENAI_API_KEY` environment variable as read at import, else an empty string | `$OPENAI_API_KEY` |
| `sampling_rate`, `source`, `only_if`, other provider options | See [Conventions](#conventions) | |

The analysis body is the string `true` or `false`, and the vCon gets the tag `engagement:true` or `engagement:false`. This is the one link that returns the vCon unchanged when no credentials are set.

***

#### hugging\_llm\_link

Summarizes the transcripts with a Hugging Face model. The prompt asks for a summary, the overall sentiment and key points, and is fixed.

| Option | Description | Default |
| ------ | ----------- | ------- |
| `HUGGINGFACE_API_KEY` | Token for the hosted Inference API | `None` |
| `model` | Model id | `meta-llama/Llama-2-70b-chat-hf` |
| `use_local_model` | Run the model in the worker with `transformers` instead of calling the API. `transformers` and a backend such as `torch` are not in the base image | `false` |
| `max_length` | Output length cap | `1000` |
| `temperature` | Sampling temperature | `0.7` |

It stores one analysis with `type: llm_analysis` and `vendor: huggingface`, and does nothing if one exists. A model failure is logged and the vCon continues unchanged. As read at `b603b15`, the API path defines `analyze` as `async` and the processor calls it without `await`, so test this link on your own data before you rely on `use_local_model: false`.

***

## Routing and filtering links

These decide which vCons continue. Returning `None` stops the chain for that vCon.

***

#### sampler

Keeps a share of vCons.

```yaml
links:
  sample_ten_percent:
    module: links.sampler
    options:
      method: percentage
      value: 10
```

| Option | Description | Default |
| ------ | ----------- | ------- |
| `method` | `percentage`, `rate`, `modulo` or `time_based`. Any other value raises | `percentage` |
| `value` | See below | `50` |
| `seed` | Seeds Python's global random generator when set | `None` |

| Method | What `value` means |
| ------ | ------------------ |
| `percentage` | Keep this percent of vCons, 0 to 100 |
| `rate` | Keep each vCon with probability `1 - e^(-1/value)`, about one in `value` for large values |
| `modulo` | Keep a vCon when the hash of its UUID is divisible by `value`. Stable per UUID |
| `time_based` | Keep a vCon that arrives when the current Unix second is divisible by `value` |

***

#### jq\_link

Filters on a [jq](https://jqlang.github.io/jq/) expression evaluated against the whole vCon.

```yaml
links:
  only_agent_calls:
    module: links.jq_link
    options:
      filter: '[.parties[] | select(.role == "agent")] | length > 0'
      forward_matches: true
```

| Option | Description | Default |
| ------ | ----------- | ------- |
| `filter` | jq program. The vCon matches when the first output is truthy, and does not match when there is no output | `.` |
| `forward_matches` | `true` continues matching vCons. `false` continues the ones that do not match | `true` |

A missing vCon or a jq error returns `None`, so an error drops the vCon from the chain and increments `conserver.link.jq.filter_errors`. A type error from string functions on mixed-type `body` arrays is retried once with the non-string items removed.

***

#### tag\_router

Pushes the UUID onto other Redis lists according to the vCon's tags.

```yaml
links:
  router:
    module: links.tag_router
    options:
      tag_routes:
        complaint: complaint_review_in
        urgent: urgent_in
      forward_original: true
```

| Option | Description | Default |
| ------ | ----------- | ------- |
| `tag_routes` | Map from tag name to target list. It matches the name before the colon in `name:value` | `{}` |
| `forward_original` | `true` lets the vCon continue in this chain. `false` stops it here | `true` |

The link reads the attachment with `purpose: tags`, as a list of `name:value` strings or a dict. A vCon with several matching tags is pushed to each of their lists. The target should be the ingress list of another chain.

***

## Data management links

***

#### tag

Adds tags to the vCon. A tag is a `name:value` string stored in the `tags` attachment.

```yaml
links:
  add_tags:
    module: links.tag
    options:
      tags:
        - "source:phone"
        - "reviewed"
```

| Option | Description | Default |
| ------ | ----------- | ------- |
| `tags` | A list of `"name:value"` strings, where a bare `name` becomes `name:name`, or a dict `{name: value}` | `["iron", "maiden"]` |

The default adds two tags, so always set `tags`. A list of `{name, value}` objects is not supported.

***

#### diet

Shrinks or scrubs the vCon in Redis before it is stored.

| Option | Description | Default |
| ------ | ----------- | ------- |
| `remove_dialog_body` | Replace each dialog `body` with an empty string, or with a URL if one of the two options below is set | `false` |
| `s3_bucket` | Upload the dialog body to this bucket and put a presigned URL in the body | `""` |
| `s3_path` | Key prefix inside the bucket | `""` |
| `aws_access_key_id`, `aws_secret_access_key` | S3 credentials | `""` |
| `aws_region` | S3 region | `us-east-1` |
| `presigned_url_expiration` | Seconds the URL is valid | `3600` when unset |
| `post_media_to_url` | If no bucket is set, POST `{content, vcon_uuid, dialog_id}` here and store the `url` from a `200` reply | `""` |
| `remove_analysis` | Delete every analysis | `false` |
| `remove_attachment_types` | Delete attachments whose `mime_type` is in this list | `[]` |
| `remove_system_prompts` | Remove every `system_prompt` key anywhere in the vCon | `false` |

If an upload or POST fails, the body is emptied. A replaced body gets `body_type: url`. `remove_attachment_types` compares the `mime_type` field, which is not a core vCon attachment field, so it does not match attachments by `purpose` or `mediatype`.

***

#### expire\_vcon

Sets a TTL on the working copy in Redis. Storages are not touched.

| Option | Description | Default |
| ------ | ----------- | ------- |
| `seconds` | TTL applied with `EXPIRE vcon:<uuid>` | `86400` |

The chain still has to store the vCon, so keep `seconds` longer than the rest of the chain takes.

***

## Integration links

***

#### webhook

POSTs the vCon as JSON to each URL. A `webhook` storage with the same options sends after the chain instead. See [Storage](storage.md#webhook).

```yaml
links:
  notify:
    module: links.webhook
    options:
      webhook-urls:
        - "https://api.example.com/vcon-webhook"
      headers:
        Authorization: "Bearer example-token"
```

| Option | Description | Default |
| ------ | ----------- | ------- |
| `webhook-urls` | URLs to POST to, in order. If empty the link logs a warning and continues | `[]` |
| `headers` | Headers for every request | `{}` |

The body honors `EGRESS_FORMAT_VERSION`. The request has no timeout, and a non-2xx reply is logged and ignored. Only a connection error raises. If you need a delivery guarantee, use the storage form, which times out after 30 seconds and treats an HTTP error as a failed write.

***

#### post\_analysis\_to\_slack

Posts an analysis, with a link to the vCon, to Slack when an analysis contains a phrase. It was written for one deployment's conventions, so check that your vCons fit before you use it.

```yaml
links:
  slack_alert:
    module: links.post_analysis_to_slack
    options:
      token: "xoxb-..."
      default_channel_name: "#vcon-alerts"
      url: "https://app.example.com/vcons/{vcon_id}"
      only_if:
        analysis_type: customer_frustration
        includes: NEEDS REVIEW
```

| Option | Description | Default |
| ------ | ----------- | ------- |
| `token` | Slack bot token | `None` |
| `default_channel_name` | Channel for every post. It has no default and the link raises without it | (required) |
| `url` | Details link. `{vcon_id}` is replaced with the UUID. Without it, `?_vcon_id="<uuid>"` is appended | placeholder text |
| `only_if` | `{analysis_type, includes}`. Posts for each analysis of that type whose body contains the text. This is a different shape from the shared `only_if` | `{analysis_type: customer_frustration, includes: NEEDS REVIEW}` |

The post body is the `summary` analysis of the same dialog, so a `summary` analysis must exist. If the vCon has a `strolid_dealer` attachment whose body names a team other than `strolid`, the link also posts to `team-<team>-alerts`. It sets `was_posted_to_slack` on the analysis so it posts once. `channel_name` and `analysis_to_post` are in the defaults, and the code does not read them.

***

## Audit links

***

#### scitt

Registers a signed statement about the vCon on a [SCRAPI](https://datatracker.ietf.org/doc/draft-ietf-scitt-scrapi/) transparency service such as [scittles](https://github.com/vcon-dev/scittles), verifies the receipt, and stores it on the vCon. Use two instances to record `vcon_created` before transcription and `vcon_enhanced` after it. For a copy that does not touch the vCon, use the [`scitt` storage](storage.md#scitt-transparency).

```yaml
links:
  scitt_created:
    module: links.scitt
    options:
      scrapi_url: "http://scittles:8000"
      signing_key_pem: "<base64 of the PEM text>"
      issuer: conserver
      key_id: conserver-key-1
      vcon_operation: vcon_created
```

| Option | Description | Default |
| ------ | ----------- | ------- |
| `scrapi_url` | Service base URL | `http://scittles:8000` |
| `signing_key_pem` | Base64 of the PEM private key text. Preferred in containers | `None` |
| `signing_key_path` | PEM file path, used when `signing_key_pem` is empty | `/etc/scitt/signing-key.pem` |
| `issuer` | COSE issuer | `conserver` |
| `key_id` | COSE key id | `conserver-key-1` |
| `vcon_operation` | Lifecycle event recorded, for example `vcon_created` or `vcon_enhanced` | `vcon_created` |
| `store_receipt` | Append the receipts as an analysis | `true` |

There is no `jwks_uri` option. To verify a receipt, the link reads `jwks_uri` from the service's `/.well-known/transparency-configuration` and falls back to `<scrapi_url>/jwks`. A receipt that fails verification raises.

It registers one statement for each party that has a `tel`, with subject `tel:<number>`, or one with subject `vcon://<uuid>` if no party does. With `store_receipt`, it adds an analysis `type: scitt_receipt`, `vendor: scittles`, on dialog `0`, whose body is one receipt object or a list of them. Each has `entry_id`, `cose_receipt` (base64), `vcon_operation`, `subject`, `vcon_hash` and `scrapi_url`. See the [Lifecycle extension](../extensions/lifecycle.md).

***

#### datatrails

Records an event for each vCon in [DataTrails](https://www.datatrails.ai/). The vCon is not changed.

```yaml
links:
  audit:
    module: links.datatrails
    options:
      partner_id: "your-partner-id"
      vcon_operation: vcon_created
      auth:
        type: oidc-client-credentials
        token_endpoint: "https://app.datatrails.ai/archivist/iam/v1/appidp/token"
        client_id: "your-client-id"
        client_secret: "your-client-secret"
```

| Option | Description | Default |
| ------ | ----------- | ------- |
| `auth` | `{type, token_endpoint, client_id, client_secret}`. The only supported `type` is `oidc-client-credentials`. Anything else raises an HTTP 501 error | (required) |
| `vcon_operation` | Event name, prefixed with `vcon_`. The code reads this key with no fallback, so set it | (required) |
| `api_url` | DataTrails API root | `https://app.datatrails.ai/archivist` |
| `partner_id` | Sent as `DataTrails-Partner-ID` | `not-set` |
| `asset_attributes` | Attributes that find or create the daily asset the events attach to | `{arc_display_type: vcon_droid, conserver_link_version: 0.3.0}` |
| `auth_url` | In the defaults. The code uses `auth.token_endpoint` | the DataTrails token URL |

The event carries the SHA-256 hash of the vCon, subject `vcon://<uuid>` and the operation. A failure to create the asset event raises. A failure to create the asset-free event is counted and ignored. DataTrails statements map onto SCITT, so prefer [`scitt`](#scitt) when you want a vendor-neutral service.

***

## Testing link

#### delay

Sleeps, then passes the vCon on unchanged. Use it to hold a chain open and exercise `CONSERVER_VCON_CONCURRENCY` and shutdown behavior.

| Option | Description | Default |
| ------ | ----------- | ------- |
| `seconds` | Time to sleep. A negative value becomes `0` | `5` |
