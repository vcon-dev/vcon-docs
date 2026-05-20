---
description: How vCons restore meaningful user control over personal conversational data
---

# vCons and Increasing End User Agency

> **Spec note:** This post was originally written when the consent model was described as a standalone "consent extension." That work has since been formalized as [`draft-howe-vcon-lawful-basis`](https://datatracker.ietf.org/doc/draft-howe-vcon-lawful-basis/). For the current spec-surface and JSON shapes, see [Lawful Basis](../extensions/lawful-basis.md). The rationale and patterns described here remain valid.

In our data-driven world, most of us have experienced the frustration of feeling powerless over our own information. We click "Accept All Cookies," agree to lengthy terms of service without reading them, and wonder what happens to our conversations, location data, and digital footprints once they disappear into corporate servers. This powerlessness represents a fundamental lack of what privacy researchers call "end user agency" – and it's a problem that new technical standards like vCons (Virtualized Conversations) are beginning to address.

### What is End User Agency and Why Does it Matter?

End user agency refers to an individual's meaningful control over their personal data – not just the theoretical right to privacy, but the practical ability to exercise that right. As privacy researcher Mike Smith explains, true agency means answering critical questions: "Are end users allowed to permanently delete their data from the company's servers? Or restrict which outside services can gain access? Can they stop the collection of their data altogether, either for a set amount of time or indefinitely, without forfeiting their ability to use the product or service?"

The absence of such agency creates what Smith identifies as problematic "power asymmetries" between individuals and technology organizations, particularly "near monopolies, like Google and Amazon," which "present a more lopsided power dynamic in the struggles over privacy than smaller competitors."

This power imbalance is especially concerning because, as Smith notes, "the larger the footprint a technology has in your life and the more types and amounts of data it collects on you, the less control you have over your privacy." Without genuine agency, users become passive subjects of data collection rather than active participants in determining how their information is used.

### How vCons Technically Address User Agency

The vCon (Virtualized Conversation) standards, particularly the [Lawful Basis extension](../extensions/lawful-basis.md), represent a significant technical advancement in restoring user agency over conversational data. These standards draw inspiration from several key insights in privacy theory and build technical solutions around them.

**Contextual Control Beyond Binary Choices:** Privacy theorist Helen Nissenbaum's concept of "contextual integrity" argues that privacy is not about secrecy but about "appropriate flow" of information according to contextual norms. vCons embody this principle by enabling users to grant specific permissions for distinct purposes rather than blanket "yes or no" consent. A user might consent to having their customer service call recorded but deny permission for sentiment analysis or marketing use of that same conversation. This granular approach directly addresses Smith's insight that privacy issues "exist on spectrums" and can be thought of as "knobs on a company's dashboard that can be turned up or down."

**Addressing Systemic Control Problems:** Legal scholar Julie Cohen argues that "privacy's most enduring institutional failure modes flow from its insistence on placing the individual and individualized control at the center." vCons address this by creating systematic, auditable consent mechanisms that don't rely solely on individual burden but build accountability into the technical infrastructure itself.

**Reclaiming the "Right to Future Tense":** Shoshana Zuboff warns that surveillance capitalism threatens what she calls our "right to the future tense, which is the essence of free will." vCons counter this through temporal boundaries – expiration controls and revalidation intervals that prevent indefinite data usage. Users can grant time-limited permissions that require explicit renewal, preserving their ability to change their minds about future data use.

**Cryptographic Transparency:** Through integration with [SCITT](scitt-supply-chain-integrity-transparency-and-trust.md) (Supply Chain Integrity, Transparency, and Trust) transparency services, vCons create verifiable audit trails of all consent decisions. This addresses Smith's concern about data aggregation that "works mostly in the dark, without our explicit consent or opt-in." Instead of trusting that companies honor privacy preferences, users can demand cryptographic proof.

**Technical Infrastructure for Rights:** The vCon specification explicitly requires support for deletion, rectification, and consent withdrawal – providing the technical infrastructure for the user agency Smith advocates. This moves beyond theoretical rights to enforceable technical capabilities.

**Proof Mechanisms and Verification:** vCons support multiple ways to verify consent authenticity, from verbal confirmation within recorded conversations to cryptographic signatures. This addresses what law and technology scholar Ryan Calo identifies as the challenge of "privacy vulnerability" – situations where users are particularly susceptible to privacy harms due to power imbalances or information asymmetries.

### Challenges That Still Exist

Despite their technical sophistication, vCons face several significant challenges in actually increasing user agency:

**Adoption and Market Power:** vCons only work if widely adopted. As Smith notes, organizations with larger networks and more dominance have greater power asymmetries. A small startup implementing vCons may increase user agency, but a dominant platform that refuses to implement them maintains the status quo.

**Technical Literacy Requirements:** Granular consent requires users to understand concepts like "sentiment analysis," "biometric processing," or "data aggregation." Many users lack the technical knowledge to make informed decisions about these processing types.

**Economic Pressure:** Users may still face "take it or leave it" scenarios where essential services require broad permissions. True agency requires not just the technical ability to say "no," but real alternatives when users exercise that choice.

**Regulatory Enforcement:** Technical standards alone cannot address power imbalances. Without strong regulatory requirements and enforcement, organizations may simply choose not to implement user-friendly consent mechanisms.

**Complexity vs. Usability:** While granular control increases agency in theory, it may overwhelm users with decisions. The challenge is providing meaningful control without creating consent fatigue.

### The Strategic Value for User Agency

Despite these challenges, vCons represent a fundamental shift in how we approach user agency over personal data. The current system, as Smith observes, makes "it almost impossible to know how data are being aggregated together and by whom," creating accountability problems for services "which we may not even know exist."

Shoshana Zuboff's analysis of surveillance capitalism provides crucial context here. She describes how tech companies have created "behavioral futures markets," where "predictions about our behavior are bought and sold, and the production of goods and services is subordinated to a new 'means of behavioral modification.'" vCons directly counter this by making behavioral data processing conditional on explicit, granular, and time-bounded consent.

vCons address this by creating a standardized, technically enforceable framework for consent that shifts the burden of proof from users to organizations. Instead of trusting that companies honor privacy preferences, users can demand cryptographic evidence of proper consent handling. This represents what Julie Cohen calls "turning privacy inside out" – focusing on "the conditions that are needed to produce sufficiently private and privacy-valuing subjects" rather than placing all responsibility on individual choice.

Most importantly, vCons recognize that meaningful agency requires more than binary choices. Helen Nissenbaum's contextual integrity theory emphasizes that privacy violations occur when "information flows outside appropriate social contexts." vCons provide infrastructure for nuanced consent decisions that acknowledge this contextual nature of privacy. Users can engage with digital services while maintaining precise control over how their data flows between different contexts and purposes.

The strategic value lies not just in the technical capabilities vCons provide, but in their potential to rebalance what Zuboff calls the fundamental power dynamic where "surveillance capitalists now develop 'economies of action,' as they learn to tune, herd, and condition our behavior with subtle and subliminal cues." By creating auditable, granular, time-bounded consent mechanisms, vCons transform privacy from a trust-based system to a verify-based system. This represents the kind of structural change needed to address the fundamental asymmetries Smith identifies between individuals and technology organizations.

While vCons alone cannot solve the privacy crisis, they provide essential technical infrastructure for a world where user agency is not just a regulatory requirement, but a practical reality built into our digital systems. As Ryan Calo observes about emerging technologies and law, we need frameworks that can adapt to "the promiscuity of data" while maintaining human control. The question now is whether market forces, regulatory pressure, and user demand will drive their adoption widely enough to fulfill their promise.

***

_I recently completed a course at HBS Online on Privacy, and I want to thank the instructors for a thoughtful program and for introducing me to the privacy researchers whose work shapes the arguments in this post._

_This post was inspired by the work of several leading privacy researchers: Mike Smith's analysis of privacy as intersecting power dynamics and user agency; Helen Nissenbaum's theory of contextual integrity and appropriate information flows; Julie Cohen's critique of individual-centered privacy approaches and her concept of "turning privacy inside out"; Shoshana Zuboff's groundbreaking analysis of surveillance capitalism and behavioral futures markets; and Ryan Calo's work on privacy vulnerability in emerging technologies. Their collective insights inform our understanding of how technical standards like vCons might restore meaningful user control in our data-driven world._
