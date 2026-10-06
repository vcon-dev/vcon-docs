---
description: Run a SIPREC session recording server that writes each recorded call as a vCon with SIP signaling metadata.
---

# 📞 vCon SIPREC Adapter

**Repo:** [vcon-dev/vcon-siprec-adapter](https://github.com/vcon-dev/vcon-siprec-adapter)

A pure-Python asyncio Session Recording Server for SIPREC (RFC 7866) over UDP, TCP and TLS. It receives the recorded RTP and the SIPREC signaling, and writes one vCon per recording session with syntax `0.4.0`. It needs no PJSIP.

## When to use it

- Your session border controller can fork calls to a SIPREC recorder and you want vCons as the output.
- You need SIP Call-IDs, tags and the SIPREC offer SDP kept with the recording.

## Run it

```bash
pip install -r requirements.txt
python main.py --config config.yaml
```

The repo also ships a Dockerfile. Set `SIPREC_PUBLIC_IP` when the bind address is not the address you advertise in SDP.

## What each vCon contains

- A `recording` dialog per stream, with the audio inline (base64url) or published to a filesystem or S3 and referenced by `url` and a `sha512-` `content_hash`. The converter sets `sip_call_id` on each recording dialog. The helper it uses can also set `sip_from_tag`, `sip_to_tag` and `sip_cseq`, but the converter does not pass them today.
- Attachments with these purposes: `sip-message-trace`, `siprec_wire` (the offer SDP), `session_metadata`, `stream_provenance` and `tags`.
- A `lawful_basis` attachment when configured (below).
- WTF transcripts in `analysis[]` if you plug in a transcription provider. The default is none.

## Signing, delivery and health

- Optional RS256 JWS signing of every vCon with a configured private key.
- Webhook delivery with HMAC-SHA256 body signing (`X-Hub-Signature-256`), an `Idempotency-Key` header set to the vCon UUID, exponential-backoff retries and an optional dead-letter directory.
- `/healthz` and Prometheus `/metrics` on port 8080, and an optional token-protected read-only `/vcons` API.

## Lawful basis

The adapter has no default basis. `lawful_basis.enabled` defaults to true, but if `lawful_basis.lawful_basis` is not set the adapter logs a warning and omits the attachment. Set it in `config.yaml` (or `SIPREC_LAWFUL_BASIS`) to the basis that applies to your deployment. This changed on 2026-09-26 (CON-1091); older releases hardcoded `legitimate_interests`.

## See also

- [SIP Signaling extension](../extensions/sip-signaling.md)
- [Lawful Basis extension](../extensions/lawful-basis.md)
- [Certifying a conversation](../use-cases-studies/patterns.md#certifying-a-conversation)
