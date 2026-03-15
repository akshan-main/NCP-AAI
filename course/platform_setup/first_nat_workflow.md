# First NeMo Agent Toolkit Workflow

## Hardware / Environment Requirements

| Requirement | Details |
|---|---|
| GPU required? | No |
| Works on CPU laptop? | Yes |
| Internet required? | Yes (for NVIDIA-hosted LLM calls) |
| Python version | 3.10+ |
| Platform | macOS, Linux, Windows with WSL |
| All steps use | NVIDIA hosted NIM endpoints via API key |

---

## Prerequisites

- Completed: [NVIDIA Account and API Key Setup](build_nvidia_account_and_api_key.md)
- Completed: [First NIM API Call](first_nim_api_call.md)
- `NVIDIA_API_KEY` environment variable is set and verified
- Python 3.10+ installed (NAT requires 3.10+)
- `pip` available

---

## Setup Steps

### Step 1: Install nvidia-nat

```bash
pip install nvidia-nat
```

> **Install note**: The base `nvidia-nat` package covers core agent types, YAML configuration, functions, and middleware. If your lab also uses evaluation, profiling, LangChain integration, or other advanced features, install the required extras — see `platform_setup/feature_to_install_matrix.md` for the full mapping.

If you encounter dependency conflicts, use a virtual environment:

```bash
python -m venv nat-env
source nat-env/bin/activate   # On Windows: nat-env\Scripts\activate
pip install nvidia-nat
```

### Step 2: Verify Installation

```bash
python -c "import nat; print(f'NAT version: {nat.__version__}')"
```

Expected output:

```
NAT version: X.Y.Z
```

If this fails with `ModuleNotFoundError`, the installation did not succeed. Check the failure cases section below.

### Step 3: Set NVIDIA_API_KEY Environment Variable

Confirm the key is available:

```bash
echo $NVIDIA_API_KEY
```

If blank, set it:

```bash
export NVIDIA_API_KEY="nvapi-your-key-here"
```

### Step 4: Create a Minimal YAML Workflow Config

Create a project directory and the configuration file:

```bash
mkdir -p nat-quickstart
```

Create the file `nat-quickstart/workflow.yaml` with the following content:

```yaml
# nat-quickstart/workflow.yaml
# A minimal NeMo Agent Toolkit workflow:
# - Uses an NVIDIA-hosted LLM via NIM
# - Defines one custom tool (get_current_time)
# - Runs a Tool Calling Agent

functions:
  get_current_time:
    type: python
    source: |
      from datetime import datetime, timezone

      def get_current_time() -> str:
          """Return the current UTC time as a formatted string."""
          now = datetime.now(timezone.utc)
          return now.strftime("%Y-%m-%d %H:%M:%S UTC")

llms:
  nim_llm:
    type: nim
    model: meta/llama-3.3-70b-instruct
    temperature: 0.1
    max_tokens: 256

agents:
  time_agent:
    type: tool_calling
    llm: nim_llm
    tools:
      - get_current_time
    system_prompt: |
      You are a helpful assistant. When the user asks for the current time,
      use the get_current_time tool. Always report the time clearly.

workflow:
  entry_agent: time_agent
```

Create the Python tool file `nat-quickstart/tools.py`:

```python
# nat-quickstart/tools.py
from datetime import datetime, timezone


def get_current_time() -> str:
    """Return the current UTC time as a formatted string."""
    now = datetime.now(timezone.utc)
    return now.strftime("%Y-%m-%d %H:%M:%S UTC")
```

### Step 5: Run the Workflow

**Option A: From the command line**

```bash
cd nat-quickstart
nat run --config workflow.yaml --input "What time is it right now?"
```

**Option B: From Python**

```python
import os
os.chdir("nat-quickstart")

from nat import Workflow

workflow = Workflow.from_config("workflow.yaml")
result = workflow.run("What time is it right now?")

print(f"Final answer: {result.output}")
print(f"\nExecution trace:")
for step in result.trace:
    print(f"  [{step.type}] {step.summary}")
```

### Step 6: Observe the Agent's Reasoning Trace

The trace should show something like this:

```
Execution trace:
  [llm_call] Agent decided to call tool: get_current_time
  [tool_call] get_current_time() -> "2026-03-15 14:30:00 UTC"
  [llm_call] Agent formulated final response using tool result
```

This demonstrates the core agent loop:
1. The LLM receives the user query and decides it needs a tool.
2. The agent framework invokes the tool.
3. The tool result is passed back to the LLM.
4. The LLM generates a natural language response incorporating the tool result.

### Step 7: Modify the Config to ReAct Agent

Edit `workflow.yaml` and change the agent type:

```yaml
agents:
  time_agent:
    type: react        # Changed from tool_calling to react
    llm: nim_llm
    tools:
      - get_current_time
    system_prompt: |
      You are a helpful assistant. When the user asks for the current time,
      use the get_current_time tool. Always report the time clearly.
      Think step by step.
```

Re-run:

```bash
nat run --config workflow.yaml --input "What time is it right now?"
```

**Observe the differences in the ReAct trace:**

```
Execution trace:
  [thought] I need to find the current time. I'll use the get_current_time tool.
  [action] get_current_time
  [observation] "2026-03-15 14:30:00 UTC"
  [thought] I now have the current time. I'll report it to the user.
  [answer] The current UTC time is 2026-03-15 14:30:00.
```

The ReAct agent explicitly shows `thought -> action -> observation` cycles, making its reasoning transparent. The Tool Calling agent uses the LLM's native function calling capability, which is more implicit.

### Step 8: Launch the NAT Console Frontend

```bash
nat console --config workflow.yaml
```

This starts an interactive chat session in your terminal. You can:
- Type messages and see the agent respond
- Observe tool calls happening in real time
- Type `exit` or `quit` to end the session

Try these test prompts:
1. "What time is it?" (should trigger the tool)
2. "Tell me a joke." (should respond without using the tool)
3. "What is 2 + 2?" (should respond without using the tool)
4. "Can you check the time and tell me if it's afternoon in UTC?" (should trigger the tool and reason about the result)

---

## Verification Checklist

Confirm each of the following before proceeding:

- [ ] `import nat` works without errors
- [ ] `workflow.yaml` is valid and loads without parsing errors
- [ ] The agent calls `get_current_time` when asked for the time
- [ ] The agent returns a natural language response that includes the time
- [ ] The execution trace shows the tool call and result
- [ ] You have run both `tool_calling` and `react` agent types and observed the trace differences
- [ ] The NAT console allows interactive chat

### Expected Successful Output (Step 5)

```
Final answer: The current UTC time is 2026-03-15 14:30:00.

Execution trace:
  [llm_call] Agent decided to call tool: get_current_time
  [tool_call] get_current_time() -> "2026-03-15 14:30:00 UTC"
  [llm_call] Agent formulated final response using tool result
```

---

## Common Failure Cases

### 1. nvidia-nat Version Incompatibility

**Symptom**: Import errors, missing attributes, or `ModuleNotFoundError` for submodules.

**Fix**:
- Check the installed version: `pip show nvidia-nat`
- Ensure you have the latest compatible version: `pip install --upgrade nvidia-nat`
- If you have conflicting packages, use a clean virtual environment.
- NAT requires Python 3.10+. Check: `python --version`

### 2. NVIDIA_API_KEY Not Found

**Symptom**: Error message like `NVIDIA_API_KEY environment variable is not set` or `Authentication failed`.

**Fix**:
- Confirm the variable is exported: `echo $NVIDIA_API_KEY`
- If using a virtual environment, you may need to re-export the variable after activating it.
- Confirm the key is valid by testing with a direct curl call (see the NIM API Call guide).

### 3. YAML Syntax Errors

**Symptom**: `yaml.scanner.ScannerError` or `yaml.parser.ParserError` with a line/column number.

**Fix**:
- YAML is indentation-sensitive. Use spaces, never tabs.
- Use exactly 2 spaces per indentation level (consistent throughout the file).
- The pipe `|` for multi-line strings requires the content to be indented relative to the key.
- Validate your YAML online at `yamllint.com` or run: `python -c "import yaml; yaml.safe_load(open('workflow.yaml'))"`

### 4. Indentation Issues in YAML

**Symptom**: Config loads but agent behaves unexpectedly (e.g., system prompt is empty, tools list is missing).

**Fix**: This is a subset of YAML syntax errors but more insidious because the file parses without error but produces the wrong structure. Check:
- `tools:` must be a list (items start with `-`)
- `system_prompt: |` must have the text block indented under it
- Run `python -c "import yaml, json; print(json.dumps(yaml.safe_load(open('workflow.yaml')), indent=2))"` to see the actual parsed structure.

### 5. Function Not Registered Correctly

**Symptom**: Agent says "I don't have access to any tools" or "I cannot call that function".

**Fix**:
- Verify the function name in `tools:` matches exactly the key under `functions:`.
- Verify the function has a docstring — NAT uses the docstring as the tool description for the LLM.
- Verify the function has a return type annotation.
- Check that the `source:` block under the function definition is correctly indented.

### 6. Model Name Mismatch

**Symptom**: HTTP 404 from the NIM endpoint or "model not found".

**Fix**: The model name in the YAML must exactly match an available NIM model. Use `meta/llama-3.3-70b-instruct` (not `llama-3.3-70b-instruct` without the provider prefix). Check available models at `build.nvidia.com`.

### 7. Port Conflict When Launching Console

**Symptom**: `Address already in use` error when starting the NAT console or server mode.

**Fix**:
- Check what is using the port: `lsof -i :8000` (or whatever port NAT defaults to).
- Kill the conflicting process or specify a different port: `nat console --config workflow.yaml --port 8001`

---

## Cleanup / Cost-Awareness Notes

- **API costs**: Each agent run makes 1-3 LLM calls (depending on agent type and complexity). The Tool Calling agent typically makes 2 calls (decide to use tool, then generate response). The ReAct agent may make more. Each call uses a few hundred tokens.
- **Total cost for this guide**: Approximately 5,000-10,000 tokens across all steps, well within free tier limits.
- **Virtual environment**: If you created a virtual environment, you can deactivate it when done: `deactivate`. To remove it entirely: `rm -rf nat-env`.
- **Project files**: The `nat-quickstart/` directory contains only local config files and can be safely deleted when no longer needed.
- **No running services**: Unless you explicitly started a server (Step 8), there are no background processes to stop. The console exits when you type `exit`.
