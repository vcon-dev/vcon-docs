---
description: >-
  Points you to the current IETF drafts, reference code, recorded talks and
  outside writing about vCon, so you can go to the primary source.
---

# ✨ More Information

## Read the current pitch

These pieces give the clearest current case for vCon.

* [vCon: The Power of a Definition](https://cpaasaa.com/vcon-the-power-of-a-definition/), Thomas Howe, CPaaSAA, August 2025. Covers why a defined file format matters and SCITT as a global digital notary.
* [The vCon Reality Check: Moving Beyond Generative Hype to Actual Conversational Architecture](https://aheadcrm.medium.com/the-vcon-reality-check-moving-beyond-generative-hype-to-actual-conversational-architecture-41197017fb9b), Thomas Wieberneit, Medium, April 2026. An analyst's view of why enterprises should care.
* [The Pulver vCon Report](https://thejeffpulver.substack.com/), Jeff Pulver's ongoing Substack series on industry adoption.

## IETF drafts

The working group's core draft is [`draft-ietf-vcon-vcon-core-04`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/), syntax `"vcon": "0.4.0"`. Track the [VCON working group](https://datatracker.ietf.org/wg/vcon/about/) for its charter, meetings and documents.

Working group drafts:

* [`draft-ietf-vcon-overview-02`](https://datatracker.ietf.org/doc/draft-ietf-vcon-overview/): use cases and architecture
* [`draft-ietf-vcon-privacy-primer-01`](https://datatracker.ietf.org/doc/draft-ietf-vcon-privacy-primer/): privacy background for implementers
* [`draft-ietf-vcon-cc-extension-02`](https://datatracker.ietf.org/doc/draft-ietf-vcon-cc-extension/): contact center extension

Individual extension drafts:

* [`draft-howe-vcon-lawful-basis-02`](https://datatracker.ietf.org/doc/draft-howe-vcon-lawful-basis/): lawful basis for processing
* [`draft-howe-vcon-lifecycle-01`](https://datatracker.ietf.org/doc/draft-howe-vcon-lifecycle/): lifecycle events on a SCITT transparency service
* [`draft-howe-vcon-wtf-extension-02`](https://datatracker.ietf.org/doc/draft-howe-vcon-wtf-extension/): World Transcription Format
* [`draft-howe-vcon-agent-session-00`](https://datatracker.ietf.org/doc/draft-howe-vcon-agent-session/): AI agent sessions
* [`draft-howe-vcon-sip-signaling-00`](https://datatracker.ietf.org/doc/draft-howe-vcon-sip-signaling/): SIP and STIR/SHAKEN signaling data
* [`draft-howe-vcon-provenance-00`](https://datatracker.ietf.org/doc/draft-howe-vcon-provenance/): generation provenance for model output

Related: [`draft-birkholz-verifiable-agent-conversations-01`](https://datatracker.ietf.org/doc/draft-birkholz-verifiable-agent-conversations/) defines the agent records that the agent session extension embeds.

Extensions are declared in the top-level `extensions` array, and the ones a reader must support go in `critical`. The [field reference](field-reference.md) lists every token, and the [Extensions](../extensions/README.md) section covers each one.

Revisions above were current on datatracker on 2026-10-06. The datatracker links always open the newest revision.

**Earlier drafts.** The individual draft [`draft-petrie-vcon-04`](https://datatracker.ietf.org/doc/draft-petrie-vcon/) used syntax `"0.0.1"`. The later working group draft [`draft-ietf-vcon-vcon-container-03`](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-container/) used `"0.0.2"`. Both are replaced by `draft-ietf-vcon-vcon-core`. Treat material citing them, or syntax `0.3.0`, as historical.

## Software

* [vcon-dev/vcon](https://github.com/vcon-dev/vcon) gathers the project's repositories, including the conserver and libraries, as submodules.
* [py-vcon](https://github.com/py-vcon/py-vcon) is Dan Petrie's Python implementation, including `py-vcon-server`.
* This site documents the [Python library](../vcon-library/README.md), the [JavaScript library](../vcon-js-library/README.md) and the [Conserver](../conserver/README.md).

## Videos and presentations from 2022 and 2023

These predate the working group's core draft, so field names in them may be out of date.

* [Keynote at TADSummit](https://youtu.be/TVq7Y1SoGo4?si=Led6pdqP6rmvynkW), Paris, October 2023
* [TADSummit Podcast](https://youtu.be/Ijmvras0DFE?si=5Qs-3VtDK8agW-Ud) with Alan Quayle, September 2023
* [Birds of a Feather session at IETF 116](https://youtu.be/EF2OMbo6Qj4), Yokohama, March 2023
* [Presentation at TADSummit](https://youtu.be/ZBRJ6FcVblc), Portugal, November 2022
* [Presentation at IETF 115](https://youtu.be/dJsPzZITr_g?t=243), London, November 2022
* [Presentation at IIT](https://youtu.be/s-pjgpBOQqc), Chicago, October 2022

See [Talks, Articles & Press](../talks-articles-press/README.md) for the full list, including newer recordings.

## Papers

* [White paper](https://docs.google.com/document/d/1TV8j29knVoOJcZvMHVFDaan0OVfraH_-nrS5gW4-DEA/edit?usp=sharing)
* [Keynote proposal for vCons](https://blog.tadsummit.com/2021/12/08/strolid-keynote-vcons/), TADSummit blog, December 2021

## Ecosystem

A partial list of companies, products and communities working with vCon.

* [Vconic](https://vconic.com): Strolid's commercial vCon platform for real-time vCon processing.
* [Strolid: IETF for vCons](https://strolid.com/ietf-for-vcons/): the BPO that incubated vCon, running roughly a quarter million conversations per month through the format.
* [MindMaking](https://mindmaking.com/): a vCon app store for service providers that turns calls into vCons with transcript, summary, action items and sentiment.
* [Nimble Ape](https://nimblea.pe/): Dan Jenkins and CommCon, the open RTC community that hosted early implementer conversations.
* [CPaaS Acceleration Alliance](https://cpaasaa.com/tag/vcon/): the alliance's vCon coverage, including the CASA Amsterdam events.
* [Cavell](https://www.cavell.com/a-comprehensive-guide-to-vcon-in-communications/): an analyst overview for the CX and CCaaS audience.
* [Telecom Reseller podcasts](https://telecomreseller.com/category/podcasts/): Doug Green's podcast series, including the Pulver, CarrierX and healthcare AI episodes.
