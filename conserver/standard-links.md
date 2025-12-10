# Standard Links

Links are the processing units of the Conserver. Each link performs a specific operation on a vCon as it flows through a chain. Links can analyze content, transform data, route vCons, integrate with external services, and more.

## Link Interface

All links implement the same interface:

```python
def run(vcon_uuid: str, link_name: str, opts: dict = default_options) -> str | None:
    """
    Process a vCon through this link.

    Args:
        vcon_uuid: UUID of the vCon to process
        link_name: Name of this link in the configuration
        opts: Configuration options merged with defaults

    Returns:
        vcon_uuid: Continue processing with this vCon UUID
        None: Stop chain processing (filter out this vCon)
    """
```

## Available Links

### Transcription Links

These links convert audio recordings in vCon dialogs to text transcripts.

---

#### deepgram_link

Speech-to-text transcription using the Deepgram API with automatic language detection and confidence scoring.

```yaml
links:
  deepgram:
    module: links.deepgram_link
    options:
      DEEPGRAM_KEY: "your-api-key"
      minimum_duration: 30
      api:
        model: "nova-2"
        smart_format: true
        detect_language: true
```

| Option | Description | Default |
|--------|-------------|---------|
| `DEEPGRAM_KEY` | Deepgram API key | (required) |
| `minimum_duration` | Minimum audio duration in seconds to transcribe | `30` |
| `api.model` | Deepgram model to use | `nova-2` |
| `api.smart_format` | Enable smart formatting | `true` |
| `api.detect_language` | Enable automatic language detection | `true` |

---

#### groq_whisper

Speech-to-text transcription using Groq's implementation of the Whisper ASR model.

```yaml
links:
  groq_whisper:
    module: links.groq_whisper
    options:
      GROQ_API_KEY: "your-api-key"
      model: "whisper-large-v3"
      minimum_duration: 3
```

| Option | Description | Default |
|--------|-------------|---------|
| `GROQ_API_KEY` | Groq API key | (required) |
| `model` | Whisper model to use | `whisper-large-v3` |
| `minimum_duration` | Minimum audio duration in seconds | `3` |

---

#### hugging_face_whisper

Speech-to-text transcription using Hugging Face's Whisper implementation, supporting both API-based and local inference.

```yaml
links:
  hf_whisper:
    module: links.hugging_face_whisper
    options:
      model: "openai/whisper-large-v3"
      minimum_duration: 3
```

| Option | Description | Default |
|--------|-------------|---------|
| `model` | Hugging Face model identifier | `openai/whisper-large-v3` |
| `minimum_duration` | Minimum audio duration in seconds | `3` |

---

#### openai_transcribe

Speech-to-text transcription using OpenAI's Whisper API or Azure OpenAI. Supports automatic chunking for long audio files.

```yaml
links:
  openai_transcribe:
    module: links.openai_transcribe
    options:
      # Public OpenAI
      OPENAI_API_KEY: "sk-..."

      # Or Azure OpenAI
      AZURE_OPENAI_API_KEY: "your-key"
      AZURE_OPENAI_ENDPOINT: "https://your-resource.openai.azure.com"
      AZURE_OPENAI_API_VERSION: "2024-10-21"

      model: "gpt-4o-transcribe"
      language: "en"
      minimum_duration: 3
      max_chunk_duration: 480
      use_silence_chunking: true
      silence_thresh: -40
      silence_len: 2000
```

| Option | Description | Default |
|--------|-------------|---------|
| `OPENAI_API_KEY` | OpenAI API key | (none) |
| `AZURE_OPENAI_API_KEY` | Azure OpenAI API key | (none) |
| `AZURE_OPENAI_ENDPOINT` | Azure OpenAI endpoint URL | (none) |
| `model` | Model to use | `gpt-4o-transcribe` |
| `language` | Language code | `en` |
| `minimum_duration` | Minimum audio duration in seconds | `3` |
| `max_chunk_duration` | Maximum chunk duration for splitting | `480` (8 min) |
| `use_silence_chunking` | Split at silence points | `true` |
| `silence_thresh` | Silence threshold in dBFS | `-40` |
| `silence_len` | Minimum silence length in ms | `2000` |

---

#### transcribe

Local transcription using the vCon library's built-in transcription capabilities.

```yaml
links:
  transcribe:
    module: links.transcribe
    options:
      transcribe_options:
        model_size: "base"
        output_options: ["vendor"]
```

| Option | Description | Default |
|--------|-------------|---------|
| `transcribe_options.model_size` | Model size | `base` |
| `transcribe_options.output_options` | Output format options | `["vendor"]` |

---

### Analysis Links

These links use AI to analyze and extract insights from vCon content.

---

#### analyze

OpenAI-powered analysis of vCon transcripts with customizable prompts, sampling, and retry mechanisms.

```yaml
links:
  analyze:
    module: links.analyze
    options:
      OPENAI_API_KEY: "sk-..."
      prompt: "Summarize this transcript in 3 bullet points"
      analysis_type: "summary"
      model: "gpt-4-turbo"
      sampling_rate: 1
      temperature: 0.3
      source:
        analysis_type: "transcript"
        text_location: "body.text"
```

| Option | Description | Default |
|--------|-------------|---------|
| `OPENAI_API_KEY` | OpenAI API key | (required) |
| `prompt` | Analysis prompt | (required) |
| `analysis_type` | Type label for the analysis | `summary` |
| `model` | OpenAI model | `gpt-3.5-turbo-16k` |
| `sampling_rate` | Fraction of vCons to analyze (0-1) | `1` |
| `temperature` | Model temperature | `0.3` |
| `source.analysis_type` | Source analysis type to analyze | `transcript` |
| `source.text_location` | Path to text within source | `body.text` |

---

#### analyze_vcon

AI analysis of entire vCon objects, returning structured JSON output.

```yaml
links:
  analyze_vcon:
    module: links.analyze_vcon
    options:
      OPENAI_API_KEY: "sk-..."
      system_prompt: "You are a conversation analyst."
      prompt: "Analyze this vCon and return insights as JSON."
      analysis_type: "vcon_analysis"
      model: "gpt-4-turbo"
```

| Option | Description | Default |
|--------|-------------|---------|
| `OPENAI_API_KEY` | OpenAI API key | (required) |
| `system_prompt` | System prompt for the model | (optional) |
| `prompt` | Analysis prompt | (required) |
| `analysis_type` | Type label for the analysis | `vcon_analysis` |
| `model` | OpenAI model | `gpt-4-turbo` |

---

#### detect_engagement

Detects whether both parties actively engaged in a conversation.

```yaml
links:
  engagement:
    module: links.detect_engagement
    options:
      OPENAI_API_KEY: "sk-..."
      prompt: "Did both the customer and the agent speak? Respond with 'true' or 'false'."
      analysis_type: "engagement_analysis"
      model: "gpt-4.1"
      source:
        analysis_type: "transcript"
        text_location: "body.paragraphs.transcript"
```

| Option | Description | Default |
|--------|-------------|---------|
| `OPENAI_API_KEY` | OpenAI API key | (required) |
| `prompt` | Evaluation prompt | (engagement detection prompt) |
| `analysis_type` | Type label | `engagement_analysis` |
| `model` | OpenAI model | `gpt-4.1` |
| `sampling_rate` | Fraction to process | `1` |

Adds an `engagement` tag with value `true` or `false`.

---

#### analyze_and_label

Combined analysis that extracts labels/categories and applies them as tags.

```yaml
links:
  labeler:
    module: links.analyze_and_label
    options:
      OPENAI_API_KEY: "sk-..."
      prompt: "Analyze this transcript and provide relevant labels."
      analysis_type: "labeled_analysis"
      model: "gpt-4-turbo"
      response_format:
        type: "json_object"
```

| Option | Description | Default |
|--------|-------------|---------|
| `OPENAI_API_KEY` | OpenAI API key | (required) |
| `prompt` | Label extraction prompt | (categorization prompt) |
| `analysis_type` | Type label | `labeled_analysis` |
| `model` | OpenAI model | `gpt-4-turbo` |
| `response_format` | Response format | `{"type": "json_object"}` |

Returns JSON with `labels` array and applies each label as a tag.

---

#### check_and_tag

Evaluates a condition using AI and applies a tag if the condition is met.

```yaml
links:
  check_complaint:
    module: links.check_and_tag
    options:
      OPENAI_API_KEY: "sk-..."
      tag_name: "complaint"
      tag_value: "detected"
      evaluation_question: "Does this conversation contain a customer complaint?"
      model: "gpt-5"
      source:
        analysis_type: "transcript"
        text_location: "body"
```

| Option | Description | Default |
|--------|-------------|---------|
| `OPENAI_API_KEY` | OpenAI API key | (required) |
| `tag_name` | Tag name to apply | (required) |
| `tag_value` | Tag value to apply | (required) |
| `evaluation_question` | Question to evaluate | (required) |
| `model` | OpenAI model | `gpt-5` |

---

#### hugging_llm_link

AI analysis using Hugging Face language models, supporting both API and local inference.

```yaml
links:
  hf_analysis:
    module: links.hugging_llm_link
    options:
      model: "mistralai/Mistral-7B-Instruct-v0.2"
      prompt: "Summarize this conversation."
      analysis_type: "hf_summary"
```

| Option | Description | Default |
|--------|-------------|---------|
| `model` | Hugging Face model identifier | (required) |
| `prompt` | Analysis prompt | (required) |
| `analysis_type` | Type label | `hf_analysis` |

---

### Routing and Filtering Links

These links control vCon flow through chains.

---

#### sampler

Selectively processes vCons based on various sampling methods.

```yaml
links:
  sampler:
    module: links.sampler
    options:
      method: "percentage"  # percentage, rate, modulo, time
      percentage: 10        # For percentage method
      rate: 100            # For rate method (1 per N)
      modulo: 5            # For modulo method
```

| Option | Description | Default |
|--------|-------------|---------|
| `method` | Sampling method | `percentage` |
| `percentage` | Percentage to process (0-100) | `100` |
| `rate` | Process 1 out of N | `1` |
| `modulo` | Process if UUID modulo equals 0 | `1` |

Returns `None` for filtered vCons, stopping their chain processing.

---

#### jq_link

Filters vCons using jq expressions for complex content-based filtering.

```yaml
links:
  filter_sales:
    module: links.jq_link
    options:
      expression: '.parties[] | select(.role == "agent")'
      forward_on_match: true
      forward_list: "sales_ingress"
```

| Option | Description | Default |
|--------|-------------|---------|
| `expression` | jq expression to evaluate | (required) |
| `forward_on_match` | Continue chain if expression matches | `true` |
| `forward_list` | Alternative ingress list for matches | (none) |

---

#### tag_router

Routes vCons to different Redis lists based on their tags.

```yaml
links:
  router:
    module: links.tag_router
    options:
      routes:
        - tag_name: "priority"
          tag_value: "high"
          destination: "priority_queue"
        - tag_name: "department"
          tag_value: "sales"
          destination: "sales_queue"
      default_destination: "general_queue"
```

| Option | Description | Default |
|--------|-------------|---------|
| `routes` | List of routing rules | `[]` |
| `routes[].tag_name` | Tag name to match | (required) |
| `routes[].tag_value` | Tag value to match | (required) |
| `routes[].destination` | Destination Redis list | (required) |
| `default_destination` | Fallback destination | (none) |

---

### Data Management Links

These links modify vCon content.

---

#### tag

Adds configurable tags to vCons.

```yaml
links:
  add_tags:
    module: links.tag
    options:
      tags:
        - name: "source"
          value: "phone"
        - name: "processed"
          value: "true"
```

| Option | Description | Default |
|--------|-------------|---------|
| `tags` | List of tags to add | `[]` |
| `tags[].name` | Tag name | (required) |
| `tags[].value` | Tag value | (required) |

---

#### diet

Reduces vCon size by removing or redirecting elements. Useful for data minimization and privacy.

```yaml
links:
  slim_down:
    module: links.diet
    options:
      remove_dialog_bodies: true
      remove_attachments: true
      remove_analysis_types:
        - "raw_transcript"
      redirect_media_to_storage: "s3"
      remove_system_prompts: true
```

| Option | Description | Default |
|--------|-------------|---------|
| `remove_dialog_bodies` | Remove dialog body content | `false` |
| `remove_attachments` | Remove all attachments | `false` |
| `remove_analysis_types` | Analysis types to remove | `[]` |
| `redirect_media_to_storage` | Move media to storage | (none) |
| `remove_system_prompts` | Remove system prompts | `false` |

---

#### expire_vcon

Sets an expiration time for vCons in Redis for automatic cleanup.

```yaml
links:
  set_expiry:
    module: links.expire_vcon
    options:
      expire_seconds: 86400  # 24 hours
```

| Option | Description | Default |
|--------|-------------|---------|
| `expire_seconds` | TTL in seconds | (required) |

---

### Integration Links

These links connect to external services.

---

#### webhook

Sends vCons to external webhook URLs.

```yaml
links:
  notify:
    module: links.webhook
    options:
      url: "https://api.example.com/vcon-webhook"
      method: "POST"
      headers:
        Authorization: "Bearer token123"
        Content-Type: "application/json"
      include_vcon: true
      timeout: 30
```

| Option | Description | Default |
|--------|-------------|---------|
| `url` | Webhook URL | (required) |
| `method` | HTTP method | `POST` |
| `headers` | HTTP headers | `{}` |
| `include_vcon` | Include full vCon in payload | `true` |
| `timeout` | Request timeout in seconds | `30` |

---

#### post_analysis_to_slack

Posts vCon analysis results to Slack channels.

```yaml
links:
  slack_notify:
    module: links.post_analysis_to_slack
    options:
      webhook_url: "https://hooks.slack.com/services/..."
      channel: "#vcon-alerts"
      analysis_type: "summary"
      template: "New conversation summary: {body}"
      condition:
        tag_name: "priority"
        tag_value: "high"
```

| Option | Description | Default |
|--------|-------------|---------|
| `webhook_url` | Slack webhook URL | (required) |
| `channel` | Slack channel | (required) |
| `analysis_type` | Analysis type to post | `summary` |
| `template` | Message template | `{body}` |
| `condition` | Optional tag condition | (none) |

---

### Audit and Compliance Links

These links provide integrity and audit trail capabilities.

---

#### datatrails

Creates verifiable audit trails using the DataTrails platform, mapping to SCITT envelopes.

```yaml
links:
  audit:
    module: links.datatrails
    options:
      client_id: "your-client-id"
      client_secret: "your-client-secret"
      asset_id: "vcon-asset"
      event_type: "vcon_processed"
      include_hash: true
```

| Option | Description | Default |
|--------|-------------|---------|
| `client_id` | DataTrails client ID | (required) |
| `client_secret` | DataTrails client secret | (required) |
| `asset_id` | Asset identifier | (required) |
| `event_type` | Event type label | `vcon_event` |
| `include_hash` | Include content hash | `true` |

---

#### scitt

Creates and registers signed statements on a SCITT Transparency Service for verifiable integrity.

```yaml
links:
  scitt:
    module: links.scitt
    options:
      service_url: "https://scitt.example.com"
      signing_key_path: "/path/to/key.pem"
      include_receipt: true
```

| Option | Description | Default |
|--------|-------------|---------|
| `service_url` | SCITT service URL | (required) |
| `signing_key_path` | Path to signing key | (required) |
| `include_receipt` | Store receipt in vCon | `true` |

---

## Using Links in Chains

Links are combined into chains in the configuration:

```yaml
chains:
  main_pipeline:
    links:
      - deepgram           # Transcribe audio
      - analyze            # Generate summary
      - check_complaint    # Check for complaints
      - router             # Route based on tags
    storages:
      - postgres
      - s3
    ingress_lists:
      - incoming_calls
    egress_lists:
      - processed_calls
    enabled: 1
    timeout: 300
```

Links execute in order. If any link returns `None`, chain processing stops for that vCon.

## Common Patterns

### Conditional Processing

Use `sampler` or `jq_link` to process only certain vCons:

```yaml
chains:
  sample_analysis:
    links:
      - sampler    # Process 10% of vCons
      - analyze
```

### Multi-stage Analysis

Chain multiple analysis links for comprehensive processing:

```yaml
chains:
  full_analysis:
    links:
      - deepgram           # Step 1: Transcribe
      - analyze            # Step 2: Summarize
      - detect_engagement  # Step 3: Check engagement
      - check_complaint    # Step 4: Detect complaints
      - analyze_and_label  # Step 5: Categorize
```

### Tag-based Routing

Use tags to route vCons to different downstream chains:

```yaml
chains:
  intake:
    links:
      - deepgram
      - analyze
      - tag_router
    ingress_lists: [incoming]
    # No egress_lists - router handles distribution

  priority_handling:
    links:
      - notify_slack
      - priority_storage
    ingress_lists: [priority_queue]
```
