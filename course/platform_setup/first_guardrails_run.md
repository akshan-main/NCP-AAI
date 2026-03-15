# First NeMo Guardrails Configuration

> **Deployment tiers**: NeMo Guardrails supports two modes. **Library mode** (used in this guide and all labs): `pip install nemoguardrails langchain-nvidia-ai-endpoints` + `NVIDIA_API_KEY` + NVIDIA-hosted models — runs on a CPU laptop. **Microservice mode** (advanced/optional): NGC container deployment requiring Docker, NGC API key, potentially GPU for Safety NIM models, and Kubernetes for production. This guide uses library mode exclusively.

> **Engine convention**: This guide uses `engine: nim`, which is the convention in current NVIDIA Guardrails documentation. Older NVIDIA docs and examples may show `engine: nvidia_ai_endpoints`. Both refer to the same NVIDIA-hosted NIM inference backend and both require `langchain-nvidia-ai-endpoints` to be installed. If you are on an older Guardrails version and `engine: nim` is not recognized, try `engine: nvidia_ai_endpoints` instead.

## Hardware / Environment Requirements

| Requirement | Details |
|---|---|
| GPU required? | No |
| Works on CPU laptop? | Yes |
| Internet required? | Yes (for NVIDIA-hosted LLM calls) |
| Python version | 3.9+ |
| Platform | macOS, Linux, Windows with WSL |
| All steps use | NVIDIA hosted NIM endpoints via API key |

---

## Prerequisites

- Completed: [NVIDIA Account and API Key Setup](build_nvidia_account_and_api_key.md)
- Completed: [First NIM API Call](first_nim_api_call.md)
- `NVIDIA_API_KEY` environment variable is set and verified
- Python 3.9+ installed

---

## Setup Steps

### Step 1: Install NeMo Guardrails

```bash
pip install nemoguardrails langchain-nvidia-ai-endpoints
```

Both packages are required. `nemoguardrails` is the guardrails framework. `langchain-nvidia-ai-endpoints` is the provider that lets Guardrails talk to NVIDIA-hosted models via your `NVIDIA_API_KEY`. Without the second package, your config will look correct but fail at runtime when it tries to reach the model.

Verify:

```bash
nemoguardrails --version
```

Expected output:

```
NeMo Guardrails version X.Y.Z
```

### Step 2: Create a Guardrails Config Directory

NeMo Guardrails expects a specific directory structure. Create it:

```bash
mkdir -p guardrails-quickstart/config
```

The directory layout will be:

```
guardrails-quickstart/
  config/
    config.yml          # Model and rails configuration
    topics.co           # Colang flows for topic control
    output_rails.co     # Colang flows for output filtering
    actions.py          # Custom Python actions
```

### Step 3: Configure config.yml with an NVIDIA-Hosted Model

Create `guardrails-quickstart/config/config.yml`:

```yaml
# guardrails-quickstart/config/config.yml

colang_version: "2.x"

models:
  - type: main
    engine: nim
    model: meta/llama-3.3-70b-instruct

# Enable input and output rails
rails:
  input:
    flows:
      - check topic
  output:
    flows:
      - check pii in output

# Instruction for the LLM's persona
instructions:
  - type: general
    content: |
      You are a helpful technology assistant. You answer questions about
      technology, programming, AI, and software engineering.
      You are polite and concise.
```

This configuration:
- Sets `colang_version: "2.x"` to use Colang 2 flow syntax in `.co` files.
- Uses the NVIDIA-hosted Llama 3.3 model via the `nim` engine (which reads `NVIDIA_API_KEY` from the environment and requires `langchain-nvidia-ai-endpoints`).
- Enables an input rail (`check topic`) and an output rail (`check pii in output`).
- Sets a persona instruction for the LLM.

> **Version note**: If you get an error about `engine: nim` not being recognized, your Guardrails version may predate this convention. Replace `engine: nim` with `engine: nvidia_ai_endpoints` — they connect to the same backend.

### Step 4: Write a Colang 2 Flow to Block Off-Topic Requests

Create `guardrails-quickstart/config/topics.co`:

```colang
# guardrails-quickstart/config/topics.co
# This flow blocks off-topic requests.

define user ask about technology
  "How does a neural network work?"
  "What programming language should I learn?"
  "Explain cloud computing."
  "What is NVIDIA NIM?"
  "How do transformers work in AI?"

define user ask off topic
  "What's the best recipe for chocolate cake?"
  "Who won the Super Bowl?"
  "Tell me a joke about cats."
  "What's the weather like today?"
  "Can you write me a love poem?"

define flow check topic
  user ask off topic
  bot refuse off topic

define bot refuse off topic
  "I'm sorry, but I can only help with technology-related questions. Please ask me something about technology, programming, AI, or software engineering."
```

This uses Colang's example-based intent matching:
- `define user ask about technology` provides examples of on-topic messages.
- `define user ask off topic` provides examples of off-topic messages.
- The `check topic` flow intercepts off-topic messages and returns a refusal.

### Step 5: Test Using the nemoguardrails Chat CLI

```bash
cd guardrails-quickstart
nemoguardrails chat --config config/
```

This starts an interactive chat session with your guardrails active.

### Step 6: Send an On-Topic Message (Verify It Passes)

In the chat session, type:

```
> How does a neural network learn?
```

**Expected behavior**: The LLM responds with a technology explanation. The input rail does not trigger because this matches the "technology" topic.

**Expected output (approximate):**

```
A neural network learns through a process called training, where it adjusts its
internal weights based on the difference between its predictions and the actual
target values. This is done using an optimization algorithm like gradient descent,
which minimizes a loss function by iteratively updating the weights...
```

### Step 7: Send an Off-Topic Message (Verify It's Blocked)

```
> What's the best recipe for chocolate cake?
```

**Expected behavior**: The input rail triggers. The LLM is NOT called. The bot returns the refusal message.

**Expected output:**

```
I'm sorry, but I can only help with technology-related questions. Please ask me
something about technology, programming, AI, or software engineering.
```

### Step 8: Write an Output Rail That Checks for PII

Create `guardrails-quickstart/config/output_rails.co`:

```colang
# guardrails-quickstart/config/output_rails.co
# This flow checks for PII (personally identifiable information) in LLM responses.

define subflow check pii in output
  $response = event bot_said
  $has_pii = execute CheckPIIAction(text=$response)

  if $has_pii
    bot inform pii redacted
    stop

define bot inform pii redacted
  "I generated a response but it contained personally identifiable information (PII), so I've blocked it to protect privacy. Please rephrase your request without asking for personal data."
```

Now add the PII checking action. Create `guardrails-quickstart/config/actions.py`:

```python
# guardrails-quickstart/config/actions.py
import re
from nemoguardrails.actions import action


@action(name="CheckPIIAction")
async def check_pii_action(text: str) -> bool:
    """Check if the text contains common PII patterns."""
    patterns = [
        r'\b\d{3}-\d{2}-\d{4}\b',              # SSN (XXX-XX-XXXX)
        r'\b\d{3}\s\d{2}\s\d{4}\b',            # SSN with spaces
        r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b',  # Email
        r'\b\d{3}[-.]?\d{3}[-.]?\d{4}\b',      # Phone number
        r'\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b',  # Credit card
    ]
    for pattern in patterns:
        if re.search(pattern, text):
            return True
    return False
```

Key differences from a plain function:
- Uses the `@action` decorator from `nemoguardrails.actions` — this is the documented way to register custom actions.
- Action name ends with `Action` (e.g., `CheckPIIAction`) — this follows the NVIDIA naming convention for Colang-callable actions.
- Function is `async` — Guardrails' action system is async-first.
- The `actions.py` file in the config directory is auto-discovered — no need for an `actions:` block in `config.yml`.

The Colang flow calls this action as `execute CheckPIIAction(text=$response)`, matching the name in the `@action` decorator.

### Step 9: Test PII Redaction

Restart the chat session:

```bash
nemoguardrails chat --config config/
```

Test with a query that might produce PII:

```
> Generate a sample user profile with name, email, phone number, and SSN for testing purposes.
```

**Expected behavior**: The LLM may attempt to generate fake PII data. The output rail checks the response. If PII patterns are detected, the response is blocked.

**Expected output:**

```
I generated a response but it contained personally identifiable information (PII),
so I've blocked it to protect privacy. Please rephrase your request without asking
for personal data.
```

Now test with a safe query:

```
> What are best practices for storing user data securely?
```

**Expected behavior**: The response discusses security practices without containing actual PII patterns, so it passes through the output rail.

### Step 10: Launch the Guardrails Server and Test via HTTP API

> **Install note**: `nemoguardrails server` requires server-specific dependencies (FastAPI, uvicorn, etc.) that are NOT included in the base `nemoguardrails` package. Install the server extra first:
>
> ```bash
> pip install nemoguardrails[server]
> ```
>
> If `nemoguardrails[server]` is not recognized as a valid extra in your version, check the current NeMo Guardrails release notes for the correct extra name, or install server dependencies manually: `pip install fastapi uvicorn`.

Start the server:

```bash
nemoguardrails server --config config/ --port 8090
```

In a separate terminal, test with curl:

```bash
# On-topic request (should pass)
curl -s http://localhost:8090/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [{"role": "user", "content": "What is NVIDIA NIM?"}]
  }' | python -m json.tool

# Off-topic request (should be blocked)
curl -s http://localhost:8090/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [{"role": "user", "content": "What is the best pizza topping?"}]
  }' | python -m json.tool
```

**Expected output for off-topic request:**

```json
{
  "messages": [
    {
      "role": "assistant",
      "content": "I'm sorry, but I can only help with technology-related questions. Please ask me something about technology, programming, AI, or software engineering."
    }
  ]
}
```

To stop the server, press `Ctrl+C` in the terminal where it is running.

---

## Verification Checklist

Confirm each of the following before proceeding:

- [ ] `nemoguardrails --version` prints a version number
- [ ] `python -c "import langchain_nvidia_ai_endpoints"` succeeds (no ImportError)
- [ ] config.yml loads without errors and includes `colang_version: "2.x"`
- [ ] On-topic messages (technology questions) pass through and get LLM responses
- [ ] Off-topic messages are blocked with the refusal message
- [ ] The PII output rail blocks responses containing email, phone, or SSN patterns
- [ ] Safe responses pass through the PII output rail
- [ ] The HTTP server starts and responds to curl requests
- [ ] You can distinguish between input rails (pre-LLM) and output rails (post-LLM)

### Before/After Comparison

| Test | Without Guardrails | With Guardrails |
|---|---|---|
| "What is a neural network?" | LLM responds normally | LLM responds normally (input rail: pass) |
| "Best chocolate cake recipe?" | LLM responds with recipe | Blocked: "I can only help with technology" (input rail: block) |
| "Generate sample SSN data" | LLM generates fake SSNs | Blocked: "PII detected" (output rail: block) |
| "How to store data securely?" | LLM responds normally | LLM responds normally (output rail: pass) |

---

## Common Failure Cases

### 1. Colang Syntax Errors

**Symptom**: `SyntaxError` or `ParsingError` when loading the config, often with a line number in the `.co` file.

**Fix**:
- Colang uses indentation to define flow structure. Use consistent spaces (2 or 4).
- `define user`, `define bot`, and `define flow` keywords must start at column 0.
- User/bot message examples must be indented under their definition.
- Strings must be in double quotes.

### 2. config.yml Engine/Model Mismatch

**Symptom**: Error connecting to model, HTTP 404, "model not found", or "engine not recognized" during chat.

**Fix**:
- Use `engine: nim` (current docs) or `engine: nvidia_ai_endpoints` (older docs). Both work but depend on your Guardrails version.
- The model name must match exactly what's available at [build.nvidia.com](https://build.nvidia.com): `meta/llama-3.3-70b-instruct`.
- Confirm `NVIDIA_API_KEY` is set in the environment where you run nemoguardrails.
- Confirm `langchain-nvidia-ai-endpoints` is installed — without it, neither engine can reach NVIDIA-hosted models.

### 3. Missing langchain-nvidia-ai-endpoints

**Symptom**: Config loads but fails when trying to call the model. Error mentions missing provider, LLM not found, or `langchain_nvidia_ai_endpoints` import failure.

**Fix**: `pip install langchain-nvidia-ai-endpoints`. This is required for both `engine: nim` and `engine: nvidia_ai_endpoints`.

### 4. Missing colang_version in config.yml

**Symptom**: Colang 2 syntax in `.co` files causes parse errors, or flows don't match the expected behavior.

**Fix**: Add `colang_version: "2.x"` as the first line in `config.yml`. Without this, Guardrails may default to Colang 1.0 parsing, which uses different syntax.

### 5. Guardrails Server Port Conflict

**Symptom**: `Address already in use` when running `nemoguardrails server`.

**Fix**:
- Check what is using the port: `lsof -i :8090`
- Use a different port: `nemoguardrails server --config config/ --port 8091`
- Kill the conflicting process if it is an old guardrails server: `kill <PID>`

### 6. Guardrails Server ImportError (FastAPI/uvicorn missing)

**Symptom**: `ModuleNotFoundError: No module named 'fastapi'` or `No module named 'uvicorn'` when running `nemoguardrails server`.

**Fix**: `pip install nemoguardrails[server]`. If the `[server]` extra is not recognized, install manually: `pip install fastapi uvicorn`.

### 7. Model Not Supporting Chat Format

**Symptom**: Garbled responses, the LLM ignores the system prompt, or rail classification fails.

**Fix**:
- Use an instruction-tuned model (ending in `-instruct` or `-chat`).
- Base models (not instruction-tuned) do not reliably follow the chat format that NeMo Guardrails expects.
- The recommended model for this guide is `meta/llama-3.3-70b-instruct`.

### 8. Rail Not Triggering (Too Permissive)

**Symptom**: Off-topic messages pass through without being blocked.

**Fix**:
- Add more examples to the `define user ask off topic` block. The more diverse examples you provide, the better the intent classifier works.
- Ensure the flow name in `config.yml` (`check topic`) matches exactly the flow name in the `.co` file (`define flow check topic`).
- Test with messages very similar to your examples first, then try more distant phrasing.

### 9. Rail Blocking Everything (Too Strict)

**Symptom**: Even on-topic technology questions are blocked.

**Fix**:
- Add more examples to the `define user ask about technology` block to give the classifier more positive examples.
- The classifier uses semantic similarity. If your off-topic examples are too broad or overlap with technology topics, the classifier may misclassify.
- Try reducing the number of off-topic examples or making them more distinct from technology.

### 10. Custom Action Not Found

**Symptom**: `ActionNotFoundError` or `NameError` when the Colang flow tries to execute `CheckPIIAction`.

**Fix**:
- Confirm `actions.py` is in the `config/` directory (same level as `config.yml`).
- Confirm the function uses the `@action(name="CheckPIIAction")` decorator from `nemoguardrails.actions`.
- Confirm the Colang flow calls `execute CheckPIIAction(...)` — the name must match the decorator's `name` parameter exactly.
- Confirm the function is `async` — Guardrails' action runner expects async functions.
- You do NOT need an `actions:` block in `config.yml` — files named `actions.py` in the config directory are auto-discovered.

---

## Cleanup / Cost-Awareness Notes

- **API costs**: Each guardrailed interaction makes 2-4 LLM calls (intent classification, response generation, output checking). This is more than a direct API call but still uses only a few hundred tokens per interaction.
- **Total cost for this guide**: Approximately 5,000-15,000 tokens across all tests, within free tier limits.
- **Server cleanup**: If you started the guardrails server (Step 10), stop it with `Ctrl+C`. Verify it is stopped: `lsof -i :8090` should return nothing.
- **Project files**: The `guardrails-quickstart/` directory can be safely deleted when no longer needed.
- **No persistent resources**: NeMo Guardrails runs locally. There are no cloud resources, databases, or containers to shut down (beyond the optional HTTP server).
