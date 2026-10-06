---
icon: database
description: Public vCon datasets on GitHub, with counts, and how to load one into a vcon-mcp environment.
---

# 📚 Public vCon Datasets

vcon-dev publishes several public datasets as GitHub repos, each vCons in IETF vCon syntax 0.4.0. Some also run as a hosted, read-only MCP server for querying the data directly.

## Datasets

| Name | What it is | Count | Spec | Repo | Hosted endpoint |
| --- | --- | --- | --- | --- | --- |
| vCon Supreme Court Arguments | US Supreme Court oral arguments, terms 1955-2025 | 8,503 | 0.4.0 | [vcon-dev/vcon-supreme-court-arguments](https://github.com/vcon-dev/vcon-supreme-court-arguments) | `mcp-scotus-open.vconic.com` |
| IETF Meeting vCons | IETF working-group sessions, meetings 66-126 plus interims | 8,181 | 0.4.0 | [vcon-dev/ietf-meeting-vcons](https://github.com/vcon-dev/ietf-meeting-vcons) | - |
| vCon Dataset: City of Newport, RI | City of Newport, RI public meetings | 115 | 0.4.0 | [vcon-dev/vcon-dataset-city-of-newport-ri](https://github.com/vcon-dev/vcon-dataset-city-of-newport-ri) | - |
| Fake vCons | Synthetic customer-service vCons from vcon_faker | 601 | 0.4.0 | [vcon-dev/fake-vcons](https://github.com/vcon-dev/fake-vcons) | - |
| TADHack 2025 | Synthetic TADHack 2025 demo calls with audio | 43 | 0.4.0 | [vcon-dev/tadhack-2025](https://github.com/vcon-dev/tadhack-2025) | - |

The SCOTUS and IETF datasets link to external audio; those external links do not carry a `content_hash` yet (work in progress). The Newport dataset's external audio links don't either, since the host blocks automated fetch.

## Load one

`vcon-data` (package `@vconic/vcon-data` 0.1.0, repo [VCONIC/vconic-datasets](https://github.com/VCONIC/vconic-datasets)) is the CLI for deploying a dataset into a vcon-mcp Supabase environment. Install it:

```
npm install -g github:VCONIC/vconic-datasets
```

Set up an environment in `~/.config/vcon-data/envs.json` (see the vcon-data README), then deploy a dataset straight from its GitHub repo:

```
vcon-data deploy github.com/vcon-dev/fake-vcons --to sandbox
```

`deploy` lints the dataset first and fails on any lint error. To check a local checkout on its own:

```
vcon-data lint ./my-dataset --json
```
