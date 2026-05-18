# 🏫 Concepts

The Conserver has five primitives. Everything you configure in `config.yml` is one of these.

## Link

A link is the basic unit of processing in the Conserver. A link takes a single vCon and does one thing to it — transcribe, summarize, tag, redact, route, notify. The conserver ships **22 standard links** (see [Standard Links](standard-links.md)) and you can write [custom links](creating-custom-links.md) just by implementing a `run(vcon_uuid, link_name, opts) -> str | None` function.

Examples:

- **Transcription** (`deepgram_link`, `openai_transcribe`, `wtf_transcribe`): convert audio to text.
- **Analysis** (`analyze`, `analyze_vcon`, `check_and_tag`): apply an LLM to extract summaries, sentiment, labels.
- **Routing & filtering** (`sampler`, `jq_link`, `tag_router`): decide which vCons proceed.
- **Integration** (`webhook`, `post_analysis_to_slack`): notify external systems.
- **Audit** (`scitt`, `datatrails`): record proof on a transparency ledger.

Links are described by Redis keys with a `link:` prefix, loaded on startup from `config.yml`. A link can be a member of multiple chains.

## Chain

A chain is a sequence of links applied to a vCon. The conserver iterates over each configured chain in the main event loop, popping vCon UUIDs from the chain's **ingress lists** (Redis lists), running them through every link in order, and either writing the result to the chain's **storages**, pushing the UUID onto **egress lists** for downstream consumers, or both.

If any link in a chain returns `None`, processing stops for that vCon — that's how `sampler`, `jq_link`, and `tag_router` (with `forward_original: false`) filter vCons mid-chain.

Chains are described by Redis keys with a `chain:` prefix and configured in the `chains:` section of `config.yml`. There is no practical limit to the number of chains or links per chain.

## Storage

A storage is the durable destination for processed vCons. After a chain's last link runs, every storage configured for that chain receives the vCon (in parallel by default — controlled by `CONSERVER_PARALLEL_STORAGE`).

The conserver ships **14 storage backends** (see [Storage](storage.md)):

- **Document stores:** `mongo`, `redis_storage`, `chatgpt_files`, `dataverse`
- **Relational:** `postgres`
- **Object stores / file systems:** `s3`, `file`, `sftp`
- **Search:** `elasticsearch`, `milvus`
- **Transparency & audit:** `scitt`, `spaceandtime`
- **Pipelines / proxies:** `vcon_mcp`, `webhook`

Different chains can target different storages — that's how you route, say, transcribed customer-service calls to Postgres while sending the redacted public-facing version to S3.

## Tracer

A tracer is the audit-trail counterpart to a link. While links *change* the vCon, tracers *record* what happened to it. The conserver invokes every configured tracer before the first link runs, after each link runs, and at chain completion — so the trail captures who processed what, when, and with what configuration.

Tracers are typically backed by a verifiable ledger (JLINC, DataTrails, or a [SCITT transparency service](../extensions/lifecycle.md)). They're optional but recommended for any deployment that has to satisfy GDPR / CCPA right-to-know or right-to-erasure requirements. See [Conserver Tracers](conserver-tracers.md) for the full lifecycle and per-tracer reference.

## Follower

A follower is a *second* conserver that polls another conserver via the REST API and mirrors its vCons. Use followers for federation, geo-distribution, or low-latency read replicas. See the `followers:` section of [Configuring the Conserver](configuring-the-conserver.md) for the configuration shape.
