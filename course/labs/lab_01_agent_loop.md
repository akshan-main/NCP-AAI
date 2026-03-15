# Lab 1: The Agent Loop — From Chatbot to Agent

## Interaction Classification
- **Type**: Hosted + Local NVIDIA interaction
- **NVIDIA Services Used**: NIM API (build.nvidia.com), NAT Tool Calling Agent
- **Credentials Required**: NVIDIA_API_KEY from build.nvidia.com
- **Hardware**: CPU laptop sufficient
- **Milestone**: Milestone 1 (First NIM call) + Milestone 2 (First NAT workflow)

## Objective

Build a minimal agent from scratch (no frameworks) that implements the **Perceive -> Reason -> Act -> Observe** loop in pure Python, calling the NVIDIA NIM API directly. Then rebuild the identical agent using NVIDIA Agent Toolkit (NAT) to understand what the framework abstracts away. By the end of this lab you will be able to explain every component of an agent loop and articulate the value NAT provides over a hand-rolled implementation.

## Prerequisites

| Requirement | Details |
|---|---|
| Module | Module 1 — Foundations of Agentic AI |
| Software | Python 3.10+, `pip` |
| Accounts | NVIDIA NGC account with an API key for build.nvidia.com NIM endpoints |
| Hardware | CPU-only is fine; no GPU required |
| Packages | `httpx`, `python-dotenv`, `nvidia-nat` |

Generate your NVIDIA NIM API key at <https://build.nvidia.com> under your account settings. Store it in a `.env` file — never commit it.

## Deliverables

1. `bare_agent.py` — a hand-rolled agent loop (no frameworks) that calls at least one tool.
2. `nat_agent.py` — the same agent rebuilt with NAT's Tool Calling Agent.
3. `nat_agent.yaml` — NAT YAML configuration for the NAT agent.
4. `comparison.md` — a short write-up (300-500 words) comparing the two implementations on lines of code, error handling, extensibility, and readability.
5. Terminal output logs showing both agents completing the same 3 test queries.

## Recommended Repo Structure

```
lab_01_agent_loop/
├── .env                   # NVIDIA_API_KEY=nvapi-...
├── bare_agent.py          # Hand-rolled agent
├── nat_agent.py           # NAT-based agent
├── nat_agent.yaml         # NAT YAML config
├── tools.py               # Shared tool definitions
├── comparison.md          # Write-up
├── requirements.txt       # httpx, python-dotenv, nvidia-nat
└── logs/
    ├── bare_agent_run.txt
    └── nat_agent_run.txt
```

## Implementation Steps

### Part A — Direct NIM API Call (Steps 1-3)

#### Step 1: Set up the project and verify NIM API access

```bash
mkdir lab_01_agent_loop && cd lab_01_agent_loop
python -m venv .venv && source .venv/bin/activate
pip install httpx python-dotenv
```

Create `.env`:

```
NVIDIA_API_KEY=nvapi-XXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

Create `test_nim.py` and make a raw HTTP call to verify your key works:

```python
import httpx
import os
from dotenv import load_dotenv

load_dotenv()

API_KEY = os.environ["NVIDIA_API_KEY"]
BASE_URL = "https://integrate.api.nvidia.com/v1"

response = httpx.post(
    f"{BASE_URL}/chat/completions",
    headers={
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
    },
    json={
        "model": "meta/llama-3.1-70b-instruct",
        "messages": [{"role": "user", "content": "Say hello in one sentence."}],
        "max_tokens": 64,
    },
    timeout=30.0,
)

print(response.status_code)
print(response.json()["choices"][0]["message"]["content"])
```

Run it: `python test_nim.py`. You should see a 200 status and a greeting. If you get 401, your API key is wrong. If you get 429, you have hit rate limits — wait 60 seconds.

#### Step 2: Define tools the agent can call

Create `tools.py` with two simple tools:

```python
import math

TOOL_REGISTRY = {}

def register_tool(name, description, parameters):
    """Decorator to register a tool with its schema."""
    def decorator(func):
        TOOL_REGISTRY[name] = {
            "function": func,
            "schema": {
                "type": "function",
                "function": {
                    "name": name,
                    "description": description,
                    "parameters": parameters,
                },
            },
        }
        return func
    return decorator

@register_tool(
    name="calculator",
    description="Evaluate a mathematical expression. Supports +, -, *, /, sqrt, pow.",
    parameters={
        "type": "object",
        "properties": {
            "expression": {
                "type": "string",
                "description": "A Python math expression, e.g. '2 + 3 * 4' or 'math.sqrt(16)'"
            }
        },
        "required": ["expression"],
    },
)
def calculator(expression: str) -> str:
    """Safely evaluate a math expression."""
    allowed_names = {"math": math, "sqrt": math.sqrt, "pow": pow, "abs": abs}
    try:
        result = eval(expression, {"__builtins__": {}}, allowed_names)
        return str(result)
    except Exception as e:
        return f"Error: {e}"

@register_tool(
    name="string_length",
    description="Return the character count of a given string.",
    parameters={
        "type": "object",
        "properties": {
            "text": {"type": "string", "description": "The string to measure"}
        },
        "required": ["text"],
    },
)
def string_length(text: str) -> str:
    return str(len(text))
```

#### Step 3: Build the bare agent loop

Create `bare_agent.py`. This is the core of Part A — a `while` loop that repeatedly: sends messages to NIM, checks if the model wants to call a tool, executes the tool, feeds the result back, and loops until the model produces a final answer.

```python
import httpx
import json
import os
from dotenv import load_dotenv
from tools import TOOL_REGISTRY

load_dotenv()

API_KEY = os.environ["NVIDIA_API_KEY"]
BASE_URL = "https://integrate.api.nvidia.com/v1"
MODEL = "meta/llama-3.1-70b-instruct"
MAX_TURNS = 10

def call_nim(messages: list, tools: list) -> dict:
    """Send a chat completion request with tool definitions."""
    response = httpx.post(
        f"{BASE_URL}/chat/completions",
        headers={
            "Authorization": f"Bearer {API_KEY}",
            "Content-Type": "application/json",
        },
        json={
            "model": MODEL,
            "messages": messages,
            "tools": tools,
            "tool_choice": "auto",
            "max_tokens": 1024,
        },
        timeout=60.0,
    )
    response.raise_for_status()
    return response.json()

def dispatch_tool(name: str, arguments: dict) -> str:
    """Look up a tool by name and call it."""
    if name not in TOOL_REGISTRY:
        return f"Error: unknown tool '{name}'"
    func = TOOL_REGISTRY[name]["function"]
    try:
        return func(**arguments)
    except TypeError as e:
        return f"Error calling {name}: {e}"

def agent_loop(user_query: str) -> str:
    """Run the Perceive -> Reason -> Act -> Observe loop."""
    tools = [entry["schema"] for entry in TOOL_REGISTRY.values()]
    messages = [
        {"role": "system", "content": "You are a helpful assistant. Use the provided tools when needed. Always show your reasoning."},
        {"role": "user", "content": user_query},
    ]

    for turn in range(MAX_TURNS):
        print(f"\n--- Turn {turn + 1} ---")

        # REASON: call the LLM
        result = call_nim(messages, tools)
        choice = result["choices"][0]
        message = choice["message"]

        # Check for finish
        if choice["finish_reason"] == "stop":
            final = message.get("content", "")
            print(f"FINAL ANSWER: {final}")
            return final

        # ACT: if the model requested tool calls, execute them
        if "tool_calls" in message and message["tool_calls"]:
            messages.append(message)  # append assistant message with tool_calls
            for tool_call in message["tool_calls"]:
                fn_name = tool_call["function"]["name"]
                fn_args = json.loads(tool_call["function"]["arguments"])
                print(f"  TOOL CALL: {fn_name}({fn_args})")

                # OBSERVE: get the tool result
                observation = dispatch_tool(fn_name, fn_args)
                print(f"  OBSERVATION: {observation}")

                messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call["id"],
                    "content": observation,
                })
        else:
            # Model returned content without tool calls — treat as final answer
            final = message.get("content", "")
            print(f"FINAL ANSWER: {final}")
            return final

    return "Error: agent exceeded maximum turns."

if __name__ == "__main__":
    test_queries = [
        "What is the square root of 144 plus 13?",
        "How many characters are in the word 'supercalifragilisticexpialidocious'?",
        "What is 2^10 divided by the length of the string 'hello world'?",
    ]
    for q in test_queries:
        print(f"\n{'='*60}")
        print(f"QUERY: {q}")
        answer = agent_loop(q)
        print(f"ANSWER: {answer}")
```

Run it and save logs:

```bash
python bare_agent.py 2>&1 | tee logs/bare_agent_run.txt
```

### Part B — Rebuild with NAT (Steps 4-5)

#### Step 4: Install NAT and rebuild the agent

```bash
pip install nvidia-nat
```

> **Install note**: The base `nvidia-nat` package covers core agent types, YAML configuration, functions, and middleware. If your lab also uses evaluation, profiling, LangChain integration, or other advanced features, install the required extras — see `platform_setup/feature_to_install_matrix.md` for the full mapping.

Create `nat_agent.yaml`:

```yaml
agent:
  type: tool_calling
  llm:
    type: nim
    model: meta/llama-3.1-70b-instruct
    api_key: ${NVIDIA_API_KEY}
    base_url: https://integrate.api.nvidia.com/v1
    max_tokens: 1024
  system_prompt: >
    You are a helpful assistant. Use the provided tools when needed.
    Always show your reasoning.
  functions:
    - name: calculator
      description: "Evaluate a mathematical expression. Supports +, -, *, /, sqrt, pow."
      parameters:
        type: object
        properties:
          expression:
            type: string
            description: "A Python math expression, e.g. '2 + 3 * 4'"
        required:
          - expression
    - name: string_length
      description: "Return the character count of a given string."
      parameters:
        type: object
        properties:
          text:
            type: string
            description: "The string to measure"
        required:
          - text
```

Create `nat_agent.py`:

```python
import nat
import math
import os
from dotenv import load_dotenv

load_dotenv()

# Register tool implementations that match the YAML function names
@nat.function("calculator")
def calculator(expression: str) -> str:
    allowed_names = {"math": math, "sqrt": math.sqrt, "pow": pow, "abs": abs}
    try:
        result = eval(expression, {"__builtins__": {}}, allowed_names)
        return str(result)
    except Exception as e:
        return f"Error: {e}"

@nat.function("string_length")
def string_length(text: str) -> str:
    return str(len(text))

# Load agent from YAML
agent = nat.from_yaml("nat_agent.yaml")

if __name__ == "__main__":
    test_queries = [
        "What is the square root of 144 plus 13?",
        "How many characters are in the word 'supercalifragilisticexpialidocious'?",
        "What is 2^10 divided by the length of the string 'hello world'?",
    ]
    for q in test_queries:
        print(f"\n{'='*60}")
        print(f"QUERY: {q}")
        response = agent.run(q)
        print(f"ANSWER: {response}")
```

Run it:

```bash
python nat_agent.py 2>&1 | tee logs/nat_agent_run.txt
```

#### Step 5: Write the comparison

Create `comparison.md`. Compare the two implementations on:

| Dimension | What to measure |
|---|---|
| Lines of code | Count only the agent logic (exclude tool definitions since they are shared) |
| Error handling | How does each handle: tool not found, malformed JSON from LLM, HTTP timeout, rate limit? |
| Extensibility | How many lines change to add a third tool? |
| Readability | Which is easier for a new developer to understand? |
| Configuration | How is the agent configured? Hardcoded vs. YAML? |

Your write-up should include a concrete table with numbers and a 2-3 sentence conclusion.

## Evaluation Rubric

| Criterion | Excellent (5) | Adequate (3) | Incomplete (1) |
|---|---|---|---|
| **Bare agent works** | All 3 test queries produce correct answers with tool calls visible in logs | 2 of 3 queries work; minor issues | Agent does not complete the loop or crashes |
| **NAT agent works** | All 3 test queries produce correct answers; YAML config is complete and valid | Agent works but YAML has hardcoded values or missing fields | NAT agent does not run |
| **Tool dispatch** | Tools are correctly registered, dispatched by name, and handle errors gracefully | Tools work but error handling is missing | Tools are hardcoded or not dispatched dynamically |
| **Comparison quality** | Includes concrete metrics (LOC counts, specific error scenarios), clear conclusion | General comparison without numbers | Missing or superficial |
| **Code quality** | Clean separation of concerns, no secrets in code, type hints, docstrings | Functional but messy | Spaghetti code or secrets committed |

## Likely Bugs and Failure Cases

1. **401 Unauthorized from NIM**: Your `NVIDIA_API_KEY` is wrong or expired. Regenerate it at build.nvidia.com. Make sure the `.env` file has no trailing spaces or quotes around the value.

2. **`tool_calls` key missing from response**: Not all NIM models support function calling. Verify you are using `meta/llama-3.1-70b-instruct` or another model that supports the `tools` parameter. If the model ignores your tools and answers directly, it may not support function calling — check the NIM model card.

3. **`json.loads` fails on tool call arguments**: The LLM sometimes returns malformed JSON in the `arguments` field (e.g., single quotes instead of double quotes, trailing commas). Wrap the parse in a try/except and return a clear error to the model so it can retry.

4. **Infinite loop in bare agent**: If the model never returns `finish_reason: "stop"` and keeps requesting tools, you will loop forever. The `MAX_TURNS` guard prevents this — make sure it is in place. Also check that you are correctly appending tool results to the message history.

5. **`eval()` security in calculator tool**: The `eval()` call is sandboxed with `{"__builtins__": {}}` but this is not production-safe. Students sometimes remove the sandbox "to make it work" when an expression fails. Do not do this. If an expression is not supported, return an error string.

6. **NAT YAML environment variable substitution fails**: The `${NVIDIA_API_KEY}` syntax in YAML requires the variable to be set in the shell environment, not just in `.env`. Either export it (`export NVIDIA_API_KEY=...`) or use `load_dotenv()` before calling `nat.from_yaml()`.

7. **Rate limiting (429)**: The free NIM tier has aggressive rate limits. If you hit them, add a `time.sleep(2)` between queries or use exponential backoff. The bare agent is more vulnerable because it makes multiple calls per query.

## Extension Ideas

1. **Add a third tool — web search**: Integrate a web search tool (e.g., using the Tavily API or a simple Wikipedia lookup with `httpx`). Observe how the LLM decides between calculator, string_length, and search without any explicit routing logic.

2. **Add conversation history**: Modify `bare_agent.py` to support multi-turn conversations where the user can ask follow-up questions. This requires persisting the `messages` list across calls. Compare how much effort this takes vs. NAT's built-in conversation handling.

3. **Instrument with token counting**: Add token counting to the bare agent (parse `usage` from the NIM response). Track total input/output tokens per query. This sets up the profiling mindset used in Lab 2.
