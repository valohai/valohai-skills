---
name: valohai-api-calls
description: Make Valohai REST API calls to fetch execution results, outputs, logs, and metadata, trigger or stop executions, manage pipelines, and automate Valohai workflows from scripts or CI/CD. Use this skill when a user wants to interact with the Valohai API programmatically, asks how to get execution data via API, wants to trigger a run from outside Valohai, needs to download artifacts, or asks any question involving the Valohai REST API. Triggers on mentions of API token, REST API, requests library, curl, fetching results, triggering executions programmatically, or automation.
license: Apache-2.0
metadata:
  author: valohai
  version: "1.0"
compatibility: Requires Python 3.8+ and the requests library (pip install requests). Designed for Claude Code or similar AI coding agents.
---

# Valohai REST API

Interact with Valohai programmatically to fetch execution results, trigger runs, download artifacts, and build automation pipelines. No special SDK required — just HTTP requests.

## Philosophy

The Valohai REST API is a standard JSON REST API. Every action available in the Valohai UI is available via API. Authentication uses a single bearer token. All list endpoints are paginated. All IDs are UUIDs.

The API base is `https://app.valohai.com/api/v0/`. Self-hosted installations use `https://<your-domain>/api/v0/`. The interactive API explorer is at `https://app.valohai.com/api/v0/` and full docs at `https://app.valohai.com/api/docs/`.

## Step-by-Step Instructions

### 1. Generate an API Token

1. Click **"Hi, \<username>!"** in the top-right corner of the Valohai UI
2. Go to **My Profile → Authentication**
3. Click **Manage Tokens** and scroll to the bottom
4. Click **Generate New Token**
5. Copy the token immediately — it is shown only once

Store it as an environment variable. Never hardcode it or commit it to version control:

```shell
# Linux/Mac
export VH_API_TOKEN=your_token_value

# Windows
set VH_API_TOKEN=your_token_value
```

### 2. Set Up Your Auth Headers

Every request needs the `Authorization` header. Set it once and reuse:

```python
import os
import requests

auth_token = os.environ["VH_API_TOKEN"]
headers = {"Authorization": f"Token {auth_token}"}
```

```bash
# cURL equivalent
curl -H "Authorization: Token $VH_API_TOKEN" https://app.valohai.com/api/v0/projects/
```

### 3. Find Your IDs

Most API calls require IDs. Here is where to find them:

| ID | Where to find it |
|---|---|
| **Project ID** | Project Settings → General; or list projects via API |
| **Execution ID** | The UUID in the URL: `.../execution/<uuid>/` |
| **Execution counter** | The `#42` number shown in the UI — use the `counter` field |
| **Step name** | Defined in `valohai.yaml` under `- step: name: <step-name>` |
| **Datum ID** | Output file UUID from listing execution outputs |
| **Pipeline ID** | The UUID in the URL: `.../pipeline/<uuid>/` |

List all projects to find project IDs:

```python
resp = requests.get("https://app.valohai.com/api/v0/projects/", headers=headers)
resp.raise_for_status()
for p in resp.json()["results"]:
    print(p["id"], p["name"])
```

## Common Integration Patterns

### Fetch Execution Results

The most common support request: getting outputs and metrics after an execution finishes.

```python
import os, requests, json

headers = {"Authorization": f"Token {os.environ['VH_API_TOKEN']}"}
execution_id = "YOUR-EXECUTION-UUID"

# 1. Check execution status
ex = requests.get(
    f"https://app.valohai.com/api/v0/executions/{execution_id}/",
    headers=headers
).json()
print("Status:", ex["status"])  # complete | error | stopped | started | queued

# 2. Fetch logged metrics/metadata
meta = requests.get(
    f"https://app.valohai.com/api/v0/executions/{execution_id}/metadata/",
    headers=headers
).json()
print("Metrics:", json.dumps(meta, indent=2))

# 3. List output files
data = requests.get(
    f"https://app.valohai.com/api/v0/data/?execution={execution_id}",
    headers=headers
).json()
for datum in data["results"]:
    print("Output:", datum["name"], "→", datum["id"])
```

### Get a Download URL for an Output File

Output files are stored in cloud storage. The API returns a short-lived pre-signed URL:

```python
datum_id = "YOUR-DATUM-UUID"
resp = requests.get(
    f"https://app.valohai.com/api/v0/data/{datum_id}/download_url/",
    headers=headers
)
resp.raise_for_status()
download_url = resp.json()["url"]
print(download_url)  # pre-signed URL, typically valid for ~1 hour
```

### Fetch Execution Logs

```python
execution_id = "YOUR-EXECUTION-UUID"
resp = requests.get(
    f"https://app.valohai.com/api/v0/executions/{execution_id}/logs/",
    headers=headers
)
resp.raise_for_status()
for line in resp.json().get("logs", []):
    print(line.get("message", ""))
```

### List and Filter Executions

```python
project_id = "YOUR-PROJECT-UUID"
resp = requests.get(
    "https://app.valohai.com/api/v0/executions/",
    headers=headers,
    params={
        "project": project_id,
        # Optional filters — combine as needed:
        # "status": "error",       # error | complete | stopped | started | queued
        # "step": "train-model",   # filter by step name
        # "limit": 50,
    }
)
resp.raise_for_status()
for ex in resp.json()["results"]:
    print(ex["counter"], ex["status"], ex["step"])
```

### Trigger a New Execution

```python
payload = {
    "project": "YOUR-PROJECT-UUID",
    "commit": "main",              # branch name, tag, or commit SHA
    "step": "train-model",         # step name from valohai.yaml
    "parameters": {                # optional — override default parameters
        "learning_rate": 0.001,
        "epochs": 10,
    },
    "inputs": {                    # optional — override default inputs
        "dataset": ["s3://my-bucket/data.csv"],
    },
    "environment": "ENV-UUID",     # optional — environment UUID
    "tags": ["ci", "experiment"],  # optional
}
resp = requests.post(
    "https://app.valohai.com/api/v0/executions/",
    headers={**headers, "Content-Type": "application/json"},
    json=payload,
)
resp.raise_for_status()
print("Created:", resp.json()["id"], "#" + str(resp.json()["counter"]))
```

> **Tip:** In the Valohai UI, use the **"Show as API call"** button on the execution create page to get a pre-filled JSON payload you can copy directly.

### Stop a Running Execution

```python
execution_id = "YOUR-EXECUTION-UUID"
resp = requests.post(
    f"https://app.valohai.com/api/v0/executions/{execution_id}/stop/",
    headers=headers
)
resp.raise_for_status()
print("Stopped:", resp.json()["status"])
```

### Trigger a Pipeline

```python
payload = {
    "project": "YOUR-PROJECT-UUID",
    "commit": "main",
    "blueprint": "my-pipeline-name",  # pipeline name from valohai.yaml
}
resp = requests.post(
    "https://app.valohai.com/api/v0/pipelines/",
    headers={**headers, "Content-Type": "application/json"},
    json=payload,
)
resp.raise_for_status()
print("Pipeline:", resp.json()["id"])
```

### Poll Until an Execution Finishes

```python
import time

execution_id = "YOUR-EXECUTION-UUID"
terminal_statuses = {"complete", "error", "stopped", "crashed"}

while True:
    ex = requests.get(
        f"https://app.valohai.com/api/v0/executions/{execution_id}/",
        headers=headers,
    ).json()
    print("Status:", ex["status"])
    if ex["status"] in terminal_statuses:
        break
    time.sleep(10)
```

### Paginate Through All Results

All list endpoints are paginated. `next` is `null` on the last page:

```python
url = "https://app.valohai.com/api/v0/executions/?project=YOUR-PROJECT-UUID"
all_executions = []
while url:
    resp = requests.get(url, headers=headers)
    resp.raise_for_status()
    data = resp.json()
    all_executions.extend(data["results"])
    url = data["next"]
print(f"Total: {len(all_executions)} executions")
```

## Best Practices

1. **Always use environment variables** for tokens — never hardcode them or commit to Git
2. **Reuse the headers dict** — build it once at the top of your script, pass everywhere
3. **Call `raise_for_status()`** after every request to catch errors early
4. **Use `json=payload`** (not `data=json.dumps(payload)`) when POSTing — the `requests` library sets the Content-Type header automatically
5. **Paginate when listing** — results are capped per page; always follow `next` links for complete data
6. **Use "Show as API call"** in the Valohai UI when building execution payloads — it generates the exact JSON with correct IDs pre-filled
7. **Download URLs are short-lived** — fetch them just before use, do not cache for longer than ~1 hour
8. **Self-hosted installs** use a different base URL — use `os.environ.get("VH_HOST", "https://app.valohai.com")` to make scripts portable

## Edge Cases

- **401 Unauthorized**: Token is missing, expired, or incorrect — regenerate in My Profile → Authentication
- **404 Not Found**: Check that the UUID is correct and belongs to an accessible project
- **Self-hosted URL**: Replace `https://app.valohai.com` with your installation domain throughout
- **Execution counter vs execution ID**: The counter (`#42`) is human-readable but the UUID is required for API calls — use `GET /api/v0/executions/?project=<id>` to find the UUID from the counter
- **Pipeline node executions**: A pipeline creates one execution per node. To fetch results from a pipeline, list executions filtered by project and tag/step, not by pipeline ID
- **Large output files**: The download URL endpoint returns a pre-signed cloud storage URL — stream large files rather than loading into memory
- **Rate limiting**: The API does not publish hard rate limits, but avoid tight polling loops; use 10-second intervals minimum when polling execution status

## Endpoint Quick Reference

| Goal | Method | Endpoint |
|---|---|---|
| List projects | GET | `/api/v0/projects/` |
| Get one project | GET | `/api/v0/projects/{id}/` |
| Fetch repo changes | POST | `/api/v0/projects/{id}/fetch/` |
| List executions | GET | `/api/v0/executions/?project={id}` |
| Get one execution | GET | `/api/v0/executions/{id}/` |
| Create execution | POST | `/api/v0/executions/` |
| Stop execution | POST | `/api/v0/executions/{id}/stop/` |
| Execution logs | GET | `/api/v0/executions/{id}/logs/` |
| Execution metadata/metrics | GET | `/api/v0/executions/{id}/metadata/` |
| List output files | GET | `/api/v0/data/?execution={id}` |
| Download URL for a file | GET | `/api/v0/data/{datum_id}/download_url/` |
| List pipelines | GET | `/api/v0/pipelines/?project={id}` |
| Create pipeline | POST | `/api/v0/pipelines/` |
| List datasets | GET | `/api/v0/datasets/?project={id}` |
| List environments | GET | `/api/v0/environments/` |
