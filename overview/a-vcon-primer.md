---
description: Thomas McCarthy-Howe, CTO, Strolid.
---

# 💬 A vCon Primer



## Introduction

Responsible management of customer data was well understood, if not well distributed, before the AI revolution. Since the AI explosion set off by ChatGPT, the environment in which customer data must be protected is distinctly more hostile. Although difficult, you can change your name and your social security number. Changing the actual look of your face, or the actual sound of your voice, is near impossible. In a future filled with deep fakes, this is a problem demanding a solution for ethical system design, sound business operation and both commercial and civil governance.

This primer explores the vCon, a groundbreaking technology designed to revolutionize the storage, analysis, and management of conversational data, of all kinds. This paper will provide a comprehensive overview of vCon, its structure, significance, and the stakeholders who should be interested in its implementation. For an current technical definition of a vCon, please visit the [working group’s page](https://datatracker.ietf.org/group/vcon/about/) at the IETF (1).

#### What is vCon?

A vCon (virtual conversation) is akin to a PDF, but instead of holding a readable document, it holds a recording of a conversation involving a “natural person” (2). It serves as a standardized format to document, store, and analyze conversations. The concept of vCon originated from a casual remark by Brian Galvin, a former CTO at Genesys, who mused about the lack of a vCard equivalent for conversations. Thus, the term vCon was coined: virtual conversations.

Think of a vCon as a document format for conversations, ensuring that data is secure and authentic. The primary goal of a vCon is to enable a system of open tools to empower customer privacy and facilitate the management of personal information in conversations.

#### Open Standard

vCons, like other data formats such as Word and PDF, are open standards. By open, we mean they are publicly available and designed for use and implementation by anyone, facilitating transparency and compatibility across different systems. Like most patent offices, the United States Patent Office does not allow patents on data formats. The vCon is truly without any intellectual property encumbrances.

This matches well with today’s data privacy challenges. Among the insidious threats of large language models is their opaque nature: unless revealed, the training data of an LLM is unknowable, thus the biases and intents of them are as well. vCons promote transparency by supporting an ecosystem of tools, applications and providers that exchange very sensitive data in a well known, and testable, format. vCons enables confident answers to “Is there personal data in this conversation?” and “Who created this vCon?”. This capability enables tools that can redact personal information, but also tools that can validate the same, each provided by otherwise independent developers.

### Why vCon Matters

#### Privacy and Data Management

Companies recording customer conversations inherently capture personal information, such as voices and faces, which are more sensitive than traditional identifiers like names. vCons help in managing and safeguarding this data, ensuring that companies can track, store, and delete data as required by regulations like GDPR.

#### Compliance with Regulations

vCons are designed to assist companies in complying with customer data regulations. For example, under GDPR, customers have the right to request deletion of their data. vCons provide a structured way to know what data was captured and ensure that it can be deleted or anonymized as required.

#### Efficiency in Machine Learning

Companies using customer data for machine learning need to manage this data responsibly. If a customer requests their data to be deleted, the company must retrain models without that data. vCons make it possible to track which models used which data, optimizing the retraining process and minimizing costs.

### Stakeholders Who Should Care

#### Companies Recording Conversations

Any company that records customer conversations needs to manage personal information responsibly. vCons provide a way to capture, store, and analyze this data while ensuring compliance with privacy regulations.

#### Regulators Governing Personal Data Protection

Companies that share customer data with other entities, such as automotive dealerships and manufacturers, need a reliable way to track and manage this information. vCons ensure that data can be traced and managed throughout its lifecycle.

#### Companies Subject to Data Privacy Regulations

Organizations operating in regions with strict data privacy laws, such as the EU under GDPR, must ensure that they can track and manage all captured data. vCons provide a structured way to meet these regulatory requirements.

#### Companies Using Machine Learning

Businesses leveraging machine learning models that use customer data need to be able to track and manage this data efficiently. vCons facilitate this by providing a clear record of what data was used, ensuring that models can be retrained as needed without excessive costs.

#### Companies Managing Customer Consent

Consent is hardly a static, nor boundless, idea. Consent is given for a purpose, with a time limit, and must be withdrawn upon request.

## Core Components of a vCon

<div align="right">

<figure><img src="../.gitbook/assets/Conserver Pictures (8).jpg" alt=""><figcaption><p>The Insides of a vCon</p></figcaption></figure>

</div>

#### Dialogues

Dialogues form the heart of a vCon, encompassing all recorded media types, including text, messaging, video, and audio. vCons can be "packed" or "unpacked." A packed vCon includes media within the JSON package, making it suitable for scenarios where all parts need to be sent together, such as in an email. Unpacked vCons, on the other hand, are useful when large media files need to be managed separately from the core data package. Each dialog identifies a list of the parties in each dialog, directly enabling a customer’s “right to know”, as described in the GDPR and the CCPA.

#### Parties

Parties in a vCon identify conversation participants. It is crucial to note not only who was involved in the conversation but also who verified their identities.

#### Analysis

The analysis component involves commentary and insights derived from the dialogues. This can range from detecting emotions, recognizing significant events like birthdays, to identifying potential deceit in conversations. The analysis is stored as JSON arrays, making it easy to attach and manage various types of analytical data.

#### Attachments

Attachments provide context to the conversation. For example, a sales lead from Ford that prompted a call can be included as an attachment, enriching the context for future reference and analysis. This ensures that all relevant data is captured and can be used effectively by both humans and automated systems.

### Future Outlook

The adoption of vCons is expected to grow as data privacy regulations become more stringent and the need for responsible data management increases. Companies will likely integrate vCons into their data engineering frameworks, ensuring that they can manage customer data effectively and comply with regulatory requirements.

### Conclusion

vCon represents a significant advancement in the management of conversational data, offering a secure, standardized way to capture, store, and analyze conversations. By enabling better data management, vCons help companies comply with privacy regulations and optimize their use of customer data in machine learning applications. As the need for responsible data management grows, vCon is poised to become an essential tool for businesses worldwide.

For more information, you can refer to the draft in the IETF, a white paper co-authored with Dan Petrie, and the working implementation of vCons in Python available on GitHub.

## Foot Notes

1. [https://datatracker.ietf.org/group/vcon/about/](https://datatracker.ietf.org/group/vcon/about/)
2. In the terminology of the GDPR, a natural person is an individual human being as opposed to a legal person such as a corporation.

