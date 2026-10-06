---
icon: diagram-project
description: Describes vConDiary, the example Streamlit app in vcon-app-template, and how to run it against your own MongoDB.
---

# vCon App Template

**Repo:** [vcon-dev/vcon-app-template](https://github.com/vcon-dev/vcon-app-template)

The repo is an example vCon app, not a framework. Its README calls the app vConDiary: a Streamlit application for managing and processing vCons that uses MongoDB for storage, OpenAI for transcription and summarization, and CarrierX as an optional source. The code is `vcondiary.py` and `suggestionbox.py`. Copy it and change it for your own app.

## Run it

You need Python, a MongoDB instance holding vCons, and an OpenAI key. The README recommends Python 3.13.1 and a virtual environment.

```bash
git clone https://github.com/vcon-dev/vcon-app-template
cd vcon-app-template
python3 -m venv ~/venvs/vcondiary && source ~/venvs/vcondiary/bin/activate
pip install -r requirements.txt
```

Create `.streamlit/secrets.toml`:

```toml
[openai]
api_key = "your-openai-key"
organization = "your-org-id"
project = "your-project-id"

[mongo_db]
url = "mongodb://localhost:27017/"
db = "vcons"
collection = "vcons"

[conserver]
api_url = "http://conserver:8000"
auth_token = "your-auth-token"
```

Then start it:

```bash
streamlit run vcondiary.py
```

## See also

- [vCon Apps and Stores](README.md)
- [Quick Start From Template](../vcon-adapters/quick-start-from-template.md), which is about adapters, not apps
