# 🔗 Standard Links

The conserver ships with a standard set of links.
Custom links can be installed by adding them into the "links" subdirectory.
The link represents an operation on a vCon, such as redaction, sharing and analysis.
The conserver can have many different configurations of the same kind of link, making them the essential unit of functionality.

<div align="center">

<figure><img src="../.gitbook/assets/Conserver Internals (1).jpg" alt=""><figcaption><p>An example of a link.</p></figcaption></figure>

</div>

Currently, the standard sets of links are:

* **Analyze**:  Calls OpenAI to process a vCon with a LLM prompt, and adding a new item to the analysis array.
* **Deepgram**:  Calls Deepgram to transcribe the dialog and add an new "transcription" item to the analysis array.
* **Expire vCon**:  Sets the expiration time for the vCon in REDIS.
* **Post analysis to Slack**:  Posts a specified analysis item to REDIS. &#x20;
* **SCITT**: Updates the SCITT ledger
* **Tag**: Adds a tag to the "attachments" array
* **Transcribe**: Calls Whisper to transcribe the dialog and add an new "transcription" item to the analysis array.
* **Webhook**: Posts vCon to URL
