---
description: Ingest SIPREC-formatted SIP recording streams and produce vCons with full signaling metadata.
---

# 📞 vCon SIPREC Adapter

**Repo:** [vcon-dev/vcon-siprec-adapter](https://github.com/vcon-dev/vcon-siprec-adapter)

SIPREC (RFC 7245 / RFC 7866) is the IETF standard for SIP-based call recording. This adapter consumes a SIPREC stream — both the recorded media and the signaling metadata — and produces a vCon with the [SIP Signaling extension](../extensions/sip-signaling.md) populated.

## When to use it

- You're running SIPREC in a contact center or carrier environment and want vCons as the durable output.
- You need STIR/SHAKEN attestation data, SIP Call-IDs, and SDP preserved alongside the recording for fraud investigation or TRACED Act compliance.
- You're correlating vCons with carrier-side CDRs.

## What you get

For each SIPREC session, the adapter produces a vCon with:

- A `dialog[]` entry of type `recording`, with the media as external (URL + SHA-512 hash) and the SIPREC-specific dialog fields populated (`sip_call_id`, `sip_from_tag`, `sip_to_tag`, `sip_cseq`).
- A `parties[]` array with `sip`, `sip_contact`, `sip_user_agent`, and `sip_display_name` for each participant.
- `attachments[]` containing the raw INVITE, the SDP, any STIR PASSporTs, and STIR verification reports — each tagged with the appropriate `purpose:` (`sip-invite`, `sip-sdp`, `stir-passport-extended`, etc.).
- `extensions: ["sip-signaling"]`.

## Spec status

The adapter is part of the May 2026 post-speckit-re-audit batch (commit history shows the field-name compliance pass on 2026-05-10). Output matches the current [SIP Signaling extension](../extensions/sip-signaling.md) draft.

## See also

- [SIP Signaling extension](../extensions/sip-signaling.md) — the spec the adapter produces against
- [Authenticating and Certifying Conversations](../use-cases-studies/authenticating-and-certifying-conversations.md) — the STIR/SHAKEN integration story
