# NVIDIA Account and API Key Setup

## Hardware / Environment Requirements

| Requirement | Details |
|---|---|
| GPU required? | No |
| Works on CPU laptop? | Yes |
| Internet required? | Yes |
| Platform | Any OS with a modern web browser and terminal (macOS, Linux, Windows with WSL) |
| All steps use | NVIDIA hosted APIs only |

---

## Prerequisites

- A modern web browser (Chrome, Firefox, Edge)
- A valid email address for account registration
- Terminal access with `curl` installed
- Python 3.9+ (for later verification steps)
- No GPU, no Docker, no special hardware

---

## Setup Steps

### Step 1: Navigate to build.nvidia.com

Open your browser and go to:

```
https://build.nvidia.com
```

This is the NVIDIA API Catalog — the central hub for discovering, testing, and integrating NVIDIA-hosted AI models.

### Step 2: Create an Account (or Sign In)

1. Click **Sign In / Register** in the top-right corner.
2. If you have an existing NVIDIA account (from NGC, GeForce, or any NVIDIA service), sign in with those credentials.
3. If you do not have an account:
   - Click **Create Account**.
   - Enter your email, create a password, and fill in required profile fields.
   - You will receive a verification email. Click the verification link.
   - Return to build.nvidia.com and sign in.

### Step 3: Navigate to the API Catalog

Once signed in:

1. Click on **API Catalog** in the top navigation (or browse the homepage cards).
2. You will see a grid of available models organized by category: LLMs, embedding models, vision models, speech models, and more.
3. Use the filters on the left sidebar to narrow by category (e.g., "Chat" or "LLM").

### Step 4: Select a Model

1. Find and click on **meta/llama-3.3-70b-instruct** (or search for "llama-3.3" in the search bar).
2. You will land on the model detail page, which shows:
   - Model description and capabilities
   - An interactive playground (right panel)
   - API endpoint URL
   - Code examples (Python, curl, etc.)

### Step 5: Generate an API Key

1. On the model detail page, look for the **"Get API Key"** button (typically near the code examples or in the header area).
2. Click it. If prompted, agree to terms of service.
3. A new API key will be generated and displayed. **Copy it immediately** — you may not be able to view it again.
4. The key will look like: `nvapi-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`

### Step 6: Store the Key Securely

**Option A: Environment variable (recommended for development)**

Add to your shell profile (`~/.bashrc`, `~/.zshrc`, or `~/.bash_profile`):

```bash
export NVIDIA_API_KEY="nvapi-your-key-here"
```

Then reload:

```bash
source ~/.bashrc   # or source ~/.zshrc
```

**Option B: .env file (for project-specific use)**

Create a `.env` file in your project root:

```
NVIDIA_API_KEY=nvapi-your-key-here
```

Then load it in Python with `python-dotenv`:

```python
from dotenv import load_dotenv
load_dotenv()
```

**Security rules:**
- Never commit API keys to version control.
- Add `.env` to your `.gitignore`.
- Never paste keys into shared documents or chat.
- Rotate keys if you suspect exposure.

### Step 7: Verify the Key Works with a curl Command

Run this in your terminal:

```bash
curl -s https://integrate.api.nvidia.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $NVIDIA_API_KEY" \
  -d '{
    "model": "meta/llama-3.3-70b-instruct",
    "messages": [{"role": "user", "content": "Say hello in one sentence."}],
    "max_tokens": 50,
    "temperature": 0.7
  }'
```

You should receive a JSON response containing the model's reply. If you see an error, refer to the Common Failure Cases section below.

### Step 8: Explore the Web UI

Back on the model page at build.nvidia.com:

1. **Playground**: In the right panel, type a message and click **Generate**. This lets you interact with the model without writing code.
2. **Temperature**: Adjust the temperature slider (0.0 = deterministic, 1.0 = creative). Try the same prompt at 0.0 and 1.0 and compare outputs.
3. **Max Tokens**: Set this to control response length. Try 50 vs. 500 and observe the difference.
4. **System Prompt**: Add a system message like "You are a helpful assistant that responds in bullet points." and observe how it changes behavior.
5. **Model Switching**: Go back to the API Catalog and try a different model (e.g., `mistralai/mixtral-8x7b-instruct-v0.1`). Compare responses to the same prompt.

### Step 9: Note Free Tier Limits and Rate Limits

- **Free tier**: NVIDIA provides a limited number of free API credits to new accounts. As of the time of writing, this typically includes a generous initial allocation sufficient for experimentation and coursework.
- **Rate limits**: Free tier accounts have requests-per-minute (RPM) and tokens-per-minute (TPM) caps. If you exceed them, you will receive HTTP 429 (Too Many Requests) errors.
- **Credit tracking**: Check your remaining credits on the build.nvidia.com dashboard under your account/billing section.
- **Cost management**: All labs in this course are designed to stay within free tier limits when run as directed. Avoid running loops or batch scripts that make hundreds of API calls during experimentation.

---

## Verification Checklist

Confirm each of the following before proceeding:

- [ ] You can sign in to build.nvidia.com
- [ ] You have generated an API key starting with `nvapi-`
- [ ] `echo $NVIDIA_API_KEY` in your terminal prints your key (not blank)
- [ ] The curl command in Step 7 returns a valid JSON response with a model-generated message
- [ ] You have tested at least one model in the web playground
- [ ] You understand your free tier limits

### Expected Successful Output (Step 7 curl)

```json
{
  "id": "chatcmpl-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "object": "chat.completion",
  "created": 1700000000,
  "model": "meta/llama-3.3-70b-instruct",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Hello! It's great to meet you, and I'm here to help with whatever you need."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 12,
    "completion_tokens": 18,
    "total_tokens": 30
  }
}
```

Key indicators of success:
- `"finish_reason": "stop"` means the model completed naturally.
- `usage` shows token counts — this is what you are billed for.
- The response `id` and `created` timestamp confirm the request was processed.

---

## Common Failure Cases

### 1. Account Verification Email Not Arriving

**Symptom**: You registered but never received the verification email.

**Fix**:
- Check your spam/junk folder.
- Try a different email provider (some corporate email filters block automated messages).
- Wait 10-15 minutes — NVIDIA's email system can have delays during peak times.
- Try resending the verification email from the registration page.

### 2. API Key Not Propagating Immediately

**Symptom**: You generated a key but API calls return `401 Unauthorized`.

**Fix**:
- Wait 2-5 minutes. New keys can take a short time to propagate across NVIDIA's infrastructure.
- Verify the key is correctly copied (no leading/trailing whitespace).
- Try generating a new key if the issue persists after 10 minutes.

### 3. Firewall or Network Blocking build.nvidia.com

**Symptom**: Browser cannot reach `build.nvidia.com` or `integrate.api.nvidia.com`. curl hangs or returns a connection error.

**Fix**:
- If on a corporate or university network, check if outbound HTTPS to `*.nvidia.com` is blocked.
- Try on a personal network or hotspot to confirm.
- If you must use the restricted network, contact your IT department to allowlist `build.nvidia.com` and `integrate.api.nvidia.com`.

### 4. Rate Limit Hit on Free Tier

**Symptom**: HTTP 429 response: `Too Many Requests`.

**Fix**:
- Wait 60 seconds and retry.
- Reduce the frequency of your requests.
- Check your remaining credits on the dashboard.
- If you have exhausted your free credits, you will need to add a payment method or wait for a credit refresh.

### 5. Key Stored Incorrectly in Environment

**Symptom**: `echo $NVIDIA_API_KEY` prints nothing, or prints the literal string `nvapi-your-key-here`.

**Fix**:
- Confirm you added the export to the correct shell profile file (`~/.zshrc` for macOS default, `~/.bashrc` for most Linux).
- Confirm you ran `source ~/.zshrc` (or equivalent) after editing.
- Confirm there are no extra quotes or spaces: it should be `export NVIDIA_API_KEY="nvapi-abc123..."`.
- Open a new terminal window and test again.

### 6. Using the Wrong Endpoint URL

**Symptom**: HTTP 404 or unexpected HTML response.

**Fix**:
- The correct base URL for API calls is `https://integrate.api.nvidia.com/v1/`.
- Do not use `https://build.nvidia.com` as the API endpoint — that is the web UI.
- Confirm the full path: `https://integrate.api.nvidia.com/v1/chat/completions`.

### 7. curl Not Installed or Outdated

**Symptom**: `command not found: curl` or TLS errors.

**Fix**:
- macOS: curl is pre-installed. If missing, install via `brew install curl`.
- Linux: `sudo apt install curl` (Debian/Ubuntu) or `sudo yum install curl` (RHEL/CentOS).
- Windows: use WSL or install curl via `winget install curl`.

---

## Cleanup / Cost-Awareness Notes

- **API keys**: If you are done experimenting and want to secure your account, you can revoke keys from the build.nvidia.com dashboard. You can always generate new ones.
- **Free credits**: Monitor your usage. The course labs are designed to consume minimal credits, but uncontrolled experimentation (running loops, large max_tokens values, batch scripts) can burn through credits quickly.
- **No infrastructure to clean up**: This setup involves only a web account and an API key. There are no containers, VMs, or cloud resources to shut down.
- **Budget rule of thumb**: A single chat completion call with a few hundred tokens costs a fraction of a cent. Staying within free tier for this course is achievable if you follow the lab instructions and avoid unnecessary repeated calls.
