---
description: Shows how three public-record collections (Supreme Court arguments, IETF sessions, a city's public meetings) were converted to vCon, which fields carry what, and where to get them.
---

# Public Corpora as vCons

Three public-record collections have been converted to vCon and published as open datasets on GitHub. Each one shows vCon holding conversations that are not phone calls: court arguments, standards meetings and municipal meetings. All three carry a `lawful_basis` attachment stating why the recording may be held, and a transcript in the [WTF format](../extensions/wtf-transcription.md).

## US Supreme Court oral arguments

[vcon-dev/vcon-supreme-court-arguments](https://github.com/vcon-dev/vcon-supreme-court-arguments), first published 2026-09.

Every Supreme Court oral argument with public audio, terms 1955 through 2025: 8,503 vCons covering about 8,600 hours of audio. Audio and speaker-attributed transcripts come from the [Oyez API](https://api.oyez.org). Where Oyez has audio but no transcript (196 recordings), the audio was transcribed locally with Whisper.

| vCon part | What it holds |
| --- | --- |
| [`parties`](../vcons/field-reference.md#party-object) | The justices (`role: justice`) and advocates (`role: advocate`) |
| [`dialog`](../vcons/field-reference.md#dialog-object) | One `recording` per argument session, an external link to the Court's MP3 |
| [`analysis`](../vcons/field-reference.md#analysis-object) | The transcript as `wtf_transcription`, one segment per speaker turn, each pointing at a party |
| [`attachments`](../vcons/field-reference.md#attachment-object) | `case_metadata` (docket, question presented, outcome, citation), `tags`, and [`lawful_basis`](../extensions/lawful-basis.md) (legitimate interests, Supreme Court public record) |

The corpus is also served read-only by a [vCon MCP server](../mcp-server/what-is-the-vcon-mcp-server.md). The repository README gives the endpoints and public read-only tokens; a token-free mirror for clients that cannot send headers runs at `https://mcp-scotus-open.vconic.com/mcp`.

## IETF meeting sessions

[vcon-dev/ietf-meeting-vcons](https://github.com/vcon-dev/ietf-meeting-vcons), first published 2026-08, built with [ietf2vcon](https://github.com/vcon-dev/ietf2vcon).

One vCon per working-group session for IETF 66 through 126 (July 2006 to July 2026) plus the VCON working group's 2026 interims: 8,181 vCons. Sessions from IETF 66 to 89 have no recordings, because the IETF did not publish them then, so those vCons carry only the agenda, minutes and slides as attachments. A vCon does not need a dialog to be a useful record.

* Recordings are external `dialog` links to IETF audio or YouTube video.
* 4,061 of 4,079 recorded sessions have a `wtf_transcription`. The `vendor` field records which engine produced each one: YouTube captions, local Whisper, or Apple's on-device speech framework for sessions with no captions.
* For most meetings the transcript body is published separately as a GitHub release asset and referenced from the vCon by `url` and [`content_hash`](../vcons/field-reference.md), which keeps the repository small while the vCon still pins the exact transcript.
* Each vCon's `lawful_basis` attachment cites the [IETF Note Well](https://www.ietf.org/about/note-well/), which permits recording, transcription and publication.
* The vCons are also in a public S3 bucket for anonymous read.

## City of Newport, RI public meetings

[vcon-dev/vcon-dataset-city-of-newport-ri](https://github.com/vcon-dev/vcon-dataset-city-of-newport-ri), first published 2026-08.

115 vCons built from the City of Newport's archived public meeting videos on its Granicus portal: City Council, Planning Board, Zoning Board, Historic District Commission and others. Each vCon has the city as convener plus a public-attendees party, the meeting video as an external `dialog`, a `meeting_metadata` attachment, a Deepgram transcript as `wtf_transcription`, and a `lawful_basis` attachment citing the [Rhode Island Open Meetings Act](http://webserver.rilegislature.gov/Statutes/TITLE42/42-46/INDEX.htm).

## Using them

Counts and load instructions are on [Public vCon Datasets](../tools/vcon-datasets.md). Because all three follow the same structure, the same tool (a vCon MCP server, a search index, a script) reads any of them without per-source code.
