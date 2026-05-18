---
description: Generate synthetic vCons with LLM-written dialog, TTS audio, and S3 upload — the standard way to create test data.
---

# 🎭 vCon Faker

**Repo:** [vcon-dev/vcon_faker](https://github.com/vcon-dev/vcon_faker) · **Streamlit app:** [vcon-faker.streamlit.app](https://vcon-faker.streamlit.app/)

`vcon_faker` generates realistic synthetic vCons. It pairs an LLM (OpenAI by default) to write the dialog with a TTS service to render audio, then assembles a vCon with the audio referenced as external media and uploaded to S3.

## When to use it

- Filling a fresh database with believable test vCons during development.
- Building a benchmark corpus for ASR providers.
- Demoing vCon-based workflows without exposing real customer data.
- Stress-testing pipelines.

## Use the synthetic-data consent pattern

Synthetic vCons should be auditably synthetic. The pattern:

1. **Mark every party as synthetic.** Set `validation: "synthetic"` on each party.
2. **Attach a `purpose: "synthetic_data_consent"` attachment** so downstream consumers can identify the vCon as synthetic in one tag-or-purpose lookup. The attachment body can be minimal — the existence of the purpose is the signal.

Example:

```json
{
  "parties": [
    { "name": "Alice", "role": "customer", "validation": "synthetic" },
    { "name": "Bob",   "role": "agent",    "validation": "synthetic" }
  ],
  "attachments": [
    {
      "purpose": "synthetic_data_consent",
      "party": 0,
      "dialog": 0,
      "encoding": "json",
      "body": "{\"origin\":\"vcon_faker\",\"prompt_version\":\"2026-05-18\"}"
    }
  ]
}
```

For full GDPR-style audit, *also* attach a [Lawful Basis](../extensions/lawful-basis.md) entry using `legitimate_interests` and `expiration: null`, documenting the synthetic origin in a `proof_mechanism` of type `external_system`. Both attachments together give you the strongest possible "this is not real personal data" signal.

## Running locally

The Streamlit app is the easiest entry point. Clone the repo, set `OPENAI_API_KEY` and S3 credentials in `.env`, and `streamlit run app.py`.

For programmatic generation, the underlying module is callable from Python — useful for batch generation in a notebook or pipeline.

## See also

- [Lawful Basis extension](../extensions/lawful-basis.md) — the spec-correct way to declare synthetic origin
- [Speech Recognition Test Set use case](../use-cases-studies/speech-recognition-test-set.md) — one of vcon_faker's most common consumers
