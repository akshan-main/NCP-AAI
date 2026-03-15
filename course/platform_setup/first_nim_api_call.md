# First NIM API Call

## Hardware / Environment Requirements

| Requirement | Details |
|---|---|
| GPU required? | No |
| Works on CPU laptop? | Yes |
| Internet required? | Yes |
| Python version | 3.9+ |
| Platform | Any OS with terminal access (macOS, Linux, Windows with WSL) |
| All steps use | NVIDIA hosted NIM endpoints only |

---

## Prerequisites

- Completed: [NVIDIA Account and API Key Setup](build_nvidia_account_and_api_key.md)
- `NVIDIA_API_KEY` environment variable is set and verified
- Python 3.9+ installed
- `curl` installed (for Step 2)
- Install Python dependencies:

```bash
pip install requests httpx openai langchain-nvidia-ai-endpoints
```

---

## Setup Steps

### Step 1: Verify NVIDIA_API_KEY Is Set

```bash
echo $NVIDIA_API_KEY
```

You should see your key printed (starting with `nvapi-`). If it prints nothing, go back to the account setup guide and fix your environment variable.

Quick Python check:

```python
import os
key = os.environ.get("NVIDIA_API_KEY")
assert key and key.startswith("nvapi-"), f"NVIDIA_API_KEY not set correctly. Got: {key!r}"
print("API key is set.")
```

---

### Step 2: Make a curl Call to the NIM Chat Completions Endpoint

```bash
curl -s -w "\n\nHTTP Status: %{http_code}\nTotal Time: %{time_total}s\n" \
  https://integrate.api.nvidia.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $NVIDIA_API_KEY" \
  -d '{
    "model": "meta/llama-3.3-70b-instruct",
    "messages": [
      {"role": "system", "content": "You are a helpful assistant. Be concise."},
      {"role": "user", "content": "What is NVIDIA NIM in two sentences?"}
    ],
    "max_tokens": 100,
    "temperature": 0.7
  }'
```

Note the `-w` flag, which appends the HTTP status code and total request time. This is your first latency measurement.

---

### Step 3: Make the Same Call Using Python (requests)

```python
import os
import time
import json
import requests

API_KEY = os.environ["NVIDIA_API_KEY"]
URL = "https://integrate.api.nvidia.com/v1/chat/completions"

headers = {
    "Content-Type": "application/json",
    "Authorization": f"Bearer {API_KEY}",
}

payload = {
    "model": "meta/llama-3.3-70b-instruct",
    "messages": [
        {"role": "system", "content": "You are a helpful assistant. Be concise."},
        {"role": "user", "content": "What is NVIDIA NIM in two sentences?"},
    ],
    "max_tokens": 100,
    "temperature": 0.7,
}

start = time.time()
response = requests.post(URL, headers=headers, json=payload)
elapsed = time.time() - start

response.raise_for_status()
data = response.json()

# Extract key fields
message = data["choices"][0]["message"]["content"]
usage = data["usage"]
finish_reason = data["choices"][0]["finish_reason"]

print(f"Model: {data['model']}")
print(f"Response: {message}")
print(f"Finish reason: {finish_reason}")
print(f"Tokens — prompt: {usage['prompt_tokens']}, "
      f"completion: {usage['completion_tokens']}, "
      f"total: {usage['total_tokens']}")
print(f"Latency: {elapsed:.2f}s")
```

**Using httpx (async-capable alternative):**

```python
import os
import time
import httpx

API_KEY = os.environ["NVIDIA_API_KEY"]
URL = "https://integrate.api.nvidia.com/v1/chat/completions"

headers = {
    "Content-Type": "application/json",
    "Authorization": f"Bearer {API_KEY}",
}

payload = {
    "model": "meta/llama-3.3-70b-instruct",
    "messages": [
        {"role": "system", "content": "You are a helpful assistant. Be concise."},
        {"role": "user", "content": "What is NVIDIA NIM in two sentences?"},
    ],
    "max_tokens": 100,
    "temperature": 0.7,
}

start = time.time()
response = httpx.post(URL, headers=headers, json=payload, timeout=30.0)
elapsed = time.time() - start

response.raise_for_status()
data = response.json()

print(f"Response: {data['choices'][0]['message']['content']}")
print(f"Latency: {elapsed:.2f}s")
```

---

### Step 4: Make the Call Using the OpenAI Python Client

NIM endpoints are **OpenAI-compatible**. This means the `openai` Python library works directly with NIM by changing only the `base_url` and `api_key`.

```python
import os
import time
from openai import OpenAI

client = OpenAI(
    base_url="https://integrate.api.nvidia.com/v1",
    api_key=os.environ["NVIDIA_API_KEY"],
)

start = time.time()
response = client.chat.completions.create(
    model="meta/llama-3.3-70b-instruct",
    messages=[
        {"role": "system", "content": "You are a helpful assistant. Be concise."},
        {"role": "user", "content": "What is NVIDIA NIM in two sentences?"},
    ],
    max_tokens=100,
    temperature=0.7,
)
elapsed = time.time() - start

print(f"Model: {response.model}")
print(f"Response: {response.choices[0].message.content}")
print(f"Finish reason: {response.choices[0].finish_reason}")
print(f"Tokens — prompt: {response.usage.prompt_tokens}, "
      f"completion: {response.usage.completion_tokens}, "
      f"total: {response.usage.total_tokens}")
print(f"Latency: {elapsed:.2f}s")
```

This is a critical insight: **any code built for OpenAI can be pointed at NIM with a two-line change** (base_url and api_key). This is the foundation of NIM's drop-in compatibility.

---

### Step 5: Make the Call Using langchain-nvidia-ai-endpoints

```python
import os
import time
from langchain_nvidia_ai_endpoints import ChatNVIDIA

# The library reads NVIDIA_API_KEY from the environment automatically
llm = ChatNVIDIA(
    model="meta/llama-3.3-70b-instruct",
    temperature=0.7,
    max_tokens=100,
)

start = time.time()
response = llm.invoke("What is NVIDIA NIM in two sentences?")
elapsed = time.time() - start

print(f"Response: {response.content}")
print(f"Metadata: {response.response_metadata}")
print(f"Latency: {elapsed:.2f}s")
```

The `langchain-nvidia-ai-endpoints` package provides native LangChain integration, which means you can use NIM models directly inside LangChain chains, agents, and RAG pipelines without any adapter code.

---

### Step 6: Inspect the Response

Using the OpenAI client response from Step 4, examine these fields:

```python
# Full response object inspection
print("--- Full Response Inspection ---")
print(f"Response ID: {response.id}")
print(f"Object type: {response.object}")
print(f"Created timestamp: {response.created}")
print(f"Model: {response.model}")
print()
print(f"Choice index: {response.choices[0].index}")
print(f"Role: {response.choices[0].message.role}")
print(f"Content: {response.choices[0].message.content}")
print(f"Finish reason: {response.choices[0].finish_reason}")
print()
print(f"Prompt tokens: {response.usage.prompt_tokens}")
print(f"Completion tokens: {response.usage.completion_tokens}")
print(f"Total tokens: {response.usage.total_tokens}")
```

**What each field means:**

| Field | Meaning |
|---|---|
| `id` | Unique identifier for this completion request |
| `model` | The model that actually processed the request (confirms routing) |
| `finish_reason` | `stop` = model finished naturally; `length` = hit max_tokens limit |
| `prompt_tokens` | How many tokens your input consumed |
| `completion_tokens` | How many tokens the model generated |
| `total_tokens` | Sum of prompt + completion (this is what you are billed for) |

---

### Step 7: Experiment with Parameters

**Temperature comparison:**

```python
from openai import OpenAI
import os

client = OpenAI(
    base_url="https://integrate.api.nvidia.com/v1",
    api_key=os.environ["NVIDIA_API_KEY"],
)

prompt = "Write a one-sentence description of the color blue."

for temp in [0.0, 0.5, 1.0]:
    print(f"\n--- Temperature: {temp} ---")
    for run in range(3):
        response = client.chat.completions.create(
            model="meta/llama-3.3-70b-instruct",
            messages=[{"role": "user", "content": prompt}],
            max_tokens=60,
            temperature=temp,
        )
        print(f"  Run {run + 1}: {response.choices[0].message.content.strip()}")
```

At temperature 0.0, all three runs should produce identical (or nearly identical) output. At 1.0, you should see variation.

**Max tokens comparison:**

```python
for max_tok in [10, 50, 200]:
    response = client.chat.completions.create(
        model="meta/llama-3.3-70b-instruct",
        messages=[{"role": "user", "content": "Explain how a transformer model works."}],
        max_tokens=max_tok,
        temperature=0.7,
    )
    content = response.choices[0].message.content
    reason = response.choices[0].finish_reason
    print(f"\nmax_tokens={max_tok} | finish_reason={reason} | length={len(content)} chars")
    print(f"  {content[:100]}...")
```

Note how `finish_reason` changes from `length` (truncated) to `stop` (completed) as you increase max_tokens.

---

### Step 8: Compare Two Models on the Same Prompt

```python
from openai import OpenAI
import os
import time

client = OpenAI(
    base_url="https://integrate.api.nvidia.com/v1",
    api_key=os.environ["NVIDIA_API_KEY"],
)

models = [
    "meta/llama-3.3-70b-instruct",
    "mistralai/mixtral-8x7b-instruct-v0.1",
]

prompt = "Explain the difference between supervised and unsupervised learning in 3 bullet points."

for model in models:
    print(f"\n{'='*60}")
    print(f"Model: {model}")
    print('='*60)

    start = time.time()
    response = client.chat.completions.create(
        model=model,
        messages=[{"role": "user", "content": prompt}],
        max_tokens=200,
        temperature=0.3,
    )
    elapsed = time.time() - start

    print(f"Response:\n{response.choices[0].message.content}")
    print(f"\nTokens: {response.usage.total_tokens} | Latency: {elapsed:.2f}s")
```

Observe differences in:
- Response quality and style
- Token efficiency (same content, fewer tokens = more efficient)
- Latency (different models have different inference speeds)

---

## Verification Checklist

Confirm each of the following before proceeding:

- [ ] curl call returns valid JSON with a model response (Step 2)
- [ ] Python requests call prints model, response, tokens, and latency (Step 3)
- [ ] OpenAI client call works with NIM base_url (Step 4)
- [ ] LangChain NVIDIA call returns a response (Step 5)
- [ ] You can identify `finish_reason`, token counts, and latency in every response
- [ ] Temperature 0.0 produces consistent outputs across runs (Step 7)
- [ ] You have compared at least two different models on the same prompt (Step 8)

### Expected Successful Output (Step 4)

```
Model: meta/llama-3.3-70b-instruct
Response: NVIDIA NIM (NVIDIA Inference Microservice) is a set of optimized inference
microservices that allow developers to deploy AI models with high performance and
low latency. It provides pre-built, GPU-accelerated containers for popular AI models,
accessible through industry-standard APIs.
Finish reason: stop
Tokens — prompt: 28, completion: 52, total: 80
Latency: 1.34s
```

---

## Common Failure Cases

### 1. Wrong Endpoint URL

**Symptom**: HTTP 404, HTML response, or "page not found".

**Fix**: The correct endpoint is `https://integrate.api.nvidia.com/v1/chat/completions`. Common mistakes:
- Using `build.nvidia.com` (that is the web UI, not the API)
- Missing `/v1/` in the path
- Using `http://` instead of `https://`

### 2. Missing Content-Type Header

**Symptom**: HTTP 415 (Unsupported Media Type) or HTTP 400 (Bad Request).

**Fix**: Ensure your request includes `Content-Type: application/json`. With `requests` and `httpx`, using the `json=` parameter sets this automatically. With curl, use `-H "Content-Type: application/json"`.

### 3. API Key in Wrong Format

**Symptom**: HTTP 401 (Unauthorized).

**Fix**:
- The Authorization header must be `Bearer <key>` (with a space after Bearer).
- Confirm the key starts with `nvapi-`.
- Check for trailing whitespace or newline characters in the key.
- If using an environment variable, confirm it is exported (not just set).

### 4. Model Name Typo

**Symptom**: HTTP 404 or error message like "model not found".

**Fix**: Model names are exact and case-sensitive. Common mistakes:
- `meta/llama-3.3-70b` (missing `-instruct`)
- `llama-3.3-70b-instruct` (missing `meta/` prefix)
- Using an old model version that has been deprecated

Check available models at: `https://build.nvidia.com`

### 5. Timeout on Large max_tokens

**Symptom**: Request hangs or returns a timeout error (HTTP 504 or client-side timeout).

**Fix**:
- Reduce `max_tokens` for testing (use 100-200 instead of 4096).
- Increase client timeout: `requests.post(..., timeout=60)` or `httpx.post(..., timeout=60.0)`.
- Large generations take longer. For the hosted API, responses over 2000 tokens may take 10-30 seconds depending on load.

### 6. JSON Parsing Errors

**Symptom**: `json.JSONDecodeError` or garbled output.

**Fix**:
- If using curl, ensure the `-d` payload is valid JSON (no trailing commas, proper quoting).
- Use `curl ... | python -m json.tool` to pretty-print and validate the response.
- If the response is HTML instead of JSON, you are hitting the wrong URL (see failure case 1).

### 7. openai Library Version Incompatibility

**Symptom**: `TypeError` or `AttributeError` when using the OpenAI client.

**Fix**: This code requires `openai>=1.0.0` (the new client API). Check your version:

```bash
pip show openai
```

If you have version 0.x, upgrade:

```bash
pip install --upgrade openai
```

---

## Cleanup / Cost-Awareness Notes

- **Token costs**: Each API call consumes tokens from your free tier allocation. The experiments in this guide use approximately 2,000-3,000 total tokens, which is negligible.
- **Avoid loops**: The temperature comparison in Step 7 makes 9 API calls (3 temperatures x 3 runs). This is fine for a one-time exercise. Do not run it in a loop.
- **Model comparison**: Step 8 makes 2 API calls. Again, negligible cost.
- **No resources to clean up**: All calls are stateless API requests. There are no containers, files, or cloud resources to shut down.
- **Rate limit awareness**: If you re-run the experiments many times in quick succession, you may hit the rate limit. Wait 60 seconds and retry.
