# 🔗 Standard Links

The conserver ships with a standard set of links.  Custom links can be installed by adding them into the "links" subdirectory.   Currently, the standard sets of links are:

* **Analyze**:  Calls OpenAI to process a vCon with a LLM prompt, and adding a new item to the analysis array.
* **Deepgram**:  Calls Deepgram to transcribe the dialog and add an new "transcription" item to the analysis array.
* **Expire vCon**:  Sets the expiration time for the vCon in REDIS.
* **Post analysis to Slack**:  Posts a specified analysis item to REDIS. &#x20;
* **SCITT**: Updates the SCITT ledger
* **Tag**: Adds a tag to the "attachments" array
* **Transcribe**: Calls Whisper to transcribe the dialog and add an new "transcription" item to the analysis array.
* **Webhook**: Posts vCon to URL
