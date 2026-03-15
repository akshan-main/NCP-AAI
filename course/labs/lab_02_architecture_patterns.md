# Lab 2: Architecture Patterns in NAT

## Interaction Classification
- **Type**: Local NVIDIA tooling interaction
- **NVIDIA Services Used**: NAT (ReAct, Reasoning, ReWOO, Tool Calling agents), NAT Profiler, NIM API
- **Credentials Required**: NVIDIA_API_KEY from build.nvidia.com
- **Hardware**: CPU laptop sufficient
- **Milestone**: Milestone 3 (Agent pattern comparison)

## Objective

Implement the **same task** using four different NAT agent architectures — ReAct, Reasoning, ReWOO, and Tool Calling — and rigorously compare their behavior. By the end of this lab you will understand the structural differences between agent patterns, know when to choose each one, and be able to use NAT Profiler to make data-driven architecture decisions.

## Prerequisites

| Requirement | Details |
|---|---|
| Modules | Module 2 — Agent Architecture Patterns |
| Prior labs | Lab 1 (working NIM API access, NAT installed) |
| Software | Python 3.10+, `nvidia-nat` |

> **Install note**: This lab uses NAT Profiler for architecture comparison. Profiling features may require extras beyond the base package. If profiler imports fail, install with `pip install nvidia-nat[eval]` or check current NAT docs for the correct extra name.
| Accounts | NVIDIA NGC account with NIM API key |
| Hardware | CPU-only is sufficient |

## Deliverables

1. `agents/react_agent.yaml` — ReAct agent configuration
2. `agents/reasoning_agent.yaml` — Reasoning agent configuration
3. `agents/rewoo_agent.yaml` — ReWOO agent configuration
4. `agents/tool_calling_agent.yaml` — Tool Calling agent configuration
5. `tools.py` — shared tool definitions used by all four agents
6. `run_all.py` — script that runs all four agents on the same set of test queries and collects profiling data
7. `results/profiling_summary.json` — raw profiling data from NAT Profiler
8. `decision_report.md` — a 500-800 word report analyzing results and recommending patterns for specific use cases

## Recommended Repo Structure

```
lab_02_architecture_patterns/
├── .env
├── agents/
│   ├── react_agent.yaml
│   ├── reasoning_agent.yaml
│   ├── rewoo_agent.yaml
│   └── tool_calling_agent.yaml
├── tools.py
├── run_all.py
├── results/
│   ├── profiling_summary.json
│   ├── react_trace.json
│   ├── reasoning_trace.json
│   ├── rewoo_trace.json
│   └── tool_calling_trace.json
├── decision_report.md
└── requirements.txt
```

## Implementation Steps

### Step 1: Define the benchmark task and tools

Choose a task that genuinely requires tool use and multi-step reasoning. The recommended task is:

> "Find the population of France and Germany, calculate which country has the higher population, compute the difference, and produce a one-paragraph summary with the numbers cited."

This task requires: (a) two information lookups, (b) a comparison, (c) a calculation, (d) a synthesis step.

Create `tools.py` with three tools:

```python
import nat
import json

# Simulated knowledge base (in a real scenario, this would be a web search or database)
COUNTRY_DATA = {
    "france": {"population": 67_390_000, "capital": "Paris", "continent": "Europe"},
    "germany": {"population": 84_480_000, "capital": "Berlin", "continent": "Europe"},
    "japan": {"population": 125_700_000, "capital": "Tokyo", "continent": "Asia"},
    "brazil": {"population": 214_300_000, "capital": "Brasilia", "continent": "South America"},
}

@nat.function("country_lookup")
def country_lookup(country_name: str) -> str:
    """Look up facts about a country. Returns population, capital, and continent."""
    key = country_name.strip().lower()
    if key in COUNTRY_DATA:
        return json.dumps(COUNTRY_DATA[key])
    return json.dumps({"error": f"Country '{country_name}' not found in database."})

@nat.function("calculator")
def calculator(expression: str) -> str:
    """Evaluate a mathematical expression. Returns the numeric result as a string."""
    try:
        result = eval(expression, {"__builtins__": {}}, {"abs": abs, "max": max, "min": min})
        return str(result)
    except Exception as e:
        return f"Error: {e}"

@nat.function("write_summary")
def write_summary(title: str, body: str) -> str:
    """Save a summary to a file and return confirmation."""
    filename = f"results/{title.lower().replace(' ', '_')}.txt"
    with open(filename, "w") as f:
        f.write(body)
    return f"Summary saved to {filename}"
```

The shared tool schemas (for YAML configs):

```yaml
# Copy this functions block into each agent YAML
functions:
  - name: country_lookup
    description: "Look up facts about a country. Returns population, capital, and continent."
    parameters:
      type: object
      properties:
        country_name:
          type: string
          description: "Name of the country to look up"
      required: [country_name]
  - name: calculator
    description: "Evaluate a mathematical expression. Returns the numeric result."
    parameters:
      type: object
      properties:
        expression:
          type: string
          description: "A Python math expression"
      required: [expression]
  - name: write_summary
    description: "Save a summary document with a title and body."
    parameters:
      type: object
      properties:
        title:
          type: string
          description: "Title of the summary"
        body:
          type: string
          description: "Body text of the summary"
      required: [title, body]
```

### Step 2: Implement the ReAct agent

Create `agents/react_agent.yaml`:

```yaml
agent:
  type: react
  llm:
    type: nim
    model: meta/llama-3.1-70b-instruct
    api_key: ${NVIDIA_API_KEY}
    base_url: https://integrate.api.nvidia.com/v1
    max_tokens: 2048
  system_prompt: >
    You are a research assistant. When asked a question, use your tools
    to look up information, perform calculations, and produce a written summary.
    Think step by step. After each action, observe the result before deciding
    your next action.
  max_iterations: 15
  functions:
    # ... (paste the shared functions block above)
```

The ReAct pattern interleaves Thought/Action/Observation. The agent explicitly writes a "Thought" before each tool call.

### Step 3: Implement the Reasoning agent

Create `agents/reasoning_agent.yaml`:

```yaml
agent:
  type: reasoning
  llm:
    type: nim
    model: meta/llama-3.1-70b-instruct
    api_key: ${NVIDIA_API_KEY}
    base_url: https://integrate.api.nvidia.com/v1
    max_tokens: 2048
  system_prompt: >
    You are a research assistant. Reason carefully about each step before
    taking action. Produce a final written summary.
  max_iterations: 15
  functions:
    # ... (paste the shared functions block)
```

The Reasoning agent uses a more structured internal reasoning process before acting. It may take fewer turns but use more tokens per turn.

### Step 4: Implement the ReWOO agent

Create `agents/rewoo_agent.yaml`:

```yaml
agent:
  type: rewoo
  llm:
    type: nim
    model: meta/llama-3.1-70b-instruct
    api_key: ${NVIDIA_API_KEY}
    base_url: https://integrate.api.nvidia.com/v1
    max_tokens: 2048
  system_prompt: >
    You are a research assistant. Plan all the steps you need upfront,
    then execute them. Produce a final written summary.
  functions:
    # ... (paste the shared functions block)
```

ReWOO (Reasoning Without Observation) plans all tool calls upfront in a single LLM call, then executes them in sequence without intermediate LLM calls. This is faster and cheaper but cannot adapt based on intermediate results.

### Step 5: Implement the Tool Calling agent

Create `agents/tool_calling_agent.yaml`:

```yaml
agent:
  type: tool_calling
  llm:
    type: nim
    model: meta/llama-3.1-70b-instruct
    api_key: ${NVIDIA_API_KEY}
    base_url: https://integrate.api.nvidia.com/v1
    max_tokens: 2048
  system_prompt: >
    You are a research assistant. Use your tools to look up information,
    perform calculations, and produce a written summary.
  max_iterations: 15
  functions:
    # ... (paste the shared functions block)
```

The Tool Calling agent relies on the LLM's native function-calling capability. It does not inject explicit Thought/Action formatting — the model decides when to call tools based on its training.

### Step 6: Build the comparison runner with profiling

Create `run_all.py`:

```python
import nat
import json
import time
import os
from dotenv import load_dotenv

load_dotenv()
os.makedirs("results", exist_ok=True)

# Import tool implementations so they are registered
import tools  # noqa: F401

AGENT_CONFIGS = {
    "react": "agents/react_agent.yaml",
    "reasoning": "agents/reasoning_agent.yaml",
    "rewoo": "agents/rewoo_agent.yaml",
    "tool_calling": "agents/tool_calling_agent.yaml",
}

TEST_QUERIES = [
    "Find the population of France and Germany, calculate which has the higher population, compute the difference, and write a one-paragraph summary with the numbers cited.",
    "What is the combined population of Japan and Brazil? Write a summary comparing them.",
    "Look up Germany's capital and France's capital, then tell me which country name has more characters.",
]

results = {}

for agent_name, config_path in AGENT_CONFIGS.items():
    print(f"\n{'='*60}")
    print(f"Running: {agent_name}")
    print(f"{'='*60}")

    agent = nat.from_yaml(config_path)
    agent_results = []

    for i, query in enumerate(TEST_QUERIES):
        print(f"\n  Query {i+1}: {query[:80]}...")
        start = time.time()

        # Use NAT Profiler to capture metrics
        with nat.profiler() as prof:
            response = agent.run(query)

        elapsed = time.time() - start

        run_data = {
            "query": query,
            "response": str(response),
            "elapsed_seconds": round(elapsed, 2),
            "total_tokens": prof.total_tokens,
            "input_tokens": prof.input_tokens,
            "output_tokens": prof.output_tokens,
            "llm_calls": prof.llm_call_count,
            "tool_calls": prof.tool_call_count,
        }
        agent_results.append(run_data)

        print(f"  Time: {elapsed:.2f}s | Tokens: {prof.total_tokens} | LLM calls: {prof.llm_call_count} | Tool calls: {prof.tool_call_count}")

    results[agent_name] = agent_results

    # Save individual trace
    with open(f"results/{agent_name}_trace.json", "w") as f:
        json.dump(agent_results, f, indent=2)

# Save combined summary
with open("results/profiling_summary.json", "w") as f:
    json.dump(results, f, indent=2)

# Print comparison table
print(f"\n{'='*80}")
print(f"{'Agent':<15} {'Avg Tokens':>12} {'Avg LLM Calls':>15} {'Avg Tool Calls':>16} {'Avg Time (s)':>14}")
print(f"{'-'*80}")
for agent_name, runs in results.items():
    avg_tokens = sum(r["total_tokens"] for r in runs) / len(runs)
    avg_llm = sum(r["llm_calls"] for r in runs) / len(runs)
    avg_tools = sum(r["tool_calls"] for r in runs) / len(runs)
    avg_time = sum(r["elapsed_seconds"] for r in runs) / len(runs)
    print(f"{agent_name:<15} {avg_tokens:>12.0f} {avg_llm:>15.1f} {avg_tools:>16.1f} {avg_time:>14.2f}")
```

Run it:

```bash
python run_all.py
```

### Step 7: Evaluate accuracy manually

For each agent's responses to the 3 test queries, manually verify:

- Did it call the correct tools?
- Are the numbers accurate?
- Did it produce a coherent summary?
- Did it write the summary file (for queries that ask for it)?

Create a simple scoring table in your `decision_report.md`:

| Query | ReAct | Reasoning | ReWOO | Tool Calling |
|---|---|---|---|---|
| Q1: France vs Germany | Correct/Partial/Wrong | ... | ... | ... |
| Q2: Japan + Brazil | ... | ... | ... | ... |
| Q3: Capital comparison | ... | ... | ... | ... |

### Step 8: Write the decision report

In `decision_report.md`, include:

1. **Data table**: Copy the profiling comparison table from the script output.
2. **Accuracy table**: From Step 7.
3. **Analysis by pattern**:
   - **ReAct**: When does explicit Thought/Action/Observation help? When is it wasteful?
   - **Reasoning**: How does the deeper reasoning phase affect token usage vs. accuracy?
   - **ReWOO**: When does upfront planning work? When does it fail (hint: when a later step depends on an earlier result)?
   - **Tool Calling**: How does native function calling compare to prompt-engineered approaches?
4. **Recommendation matrix**: Map use cases to patterns:
   - Simple single-tool queries -> ?
   - Complex multi-step research -> ?
   - Latency-sensitive applications -> ?
   - Cost-sensitive applications -> ?

## Evaluation Rubric

| Criterion | Excellent (5) | Adequate (3) | Incomplete (1) |
|---|---|---|---|
| **All 4 agents run** | All agents complete all 3 queries without crashing | 3 of 4 agents work | Fewer than 3 agents work |
| **Profiling data collected** | Token counts, LLM calls, tool calls, and latency captured for all runs | Some metrics missing | No profiling data |
| **Accuracy evaluation** | All 12 responses manually checked with a clear scoring rubric | Spot-checked | Not evaluated |
| **Decision report** | Data-driven, cites specific numbers, clear recommendation matrix | General observations without data | Missing or trivial |
| **Code quality** | Shared tools, DRY configs, reproducible runner script | Some duplication | Copy-paste with hardcoded values |

## Likely Bugs and Failure Cases

1. **ReWOO fails on dependent steps**: The ReWOO agent plans all steps upfront. If Step 3 depends on the result of Step 1 (e.g., "calculate the difference between the two populations"), ReWOO may plan the calculation with placeholder values or wrong variable references. This is a fundamental limitation — document it, do not try to "fix" it.

2. **ReAct agent gets stuck in a loop**: The ReAct agent sometimes repeats the same Thought/Action cycle if the observation is ambiguous. If this happens, increase `max_iterations` and check if the system prompt clearly instructs the agent to synthesize a final answer after gathering enough information.

3. **Token budget exceeded**: The Reasoning agent uses more tokens per turn for its internal reasoning. If `max_tokens` is too low (e.g., 512), the reasoning will be truncated and the agent will produce incomplete actions. Set `max_tokens: 2048` or higher.

4. **NAT Profiler context manager not available in your version**: If `nat.profiler()` is not available, you may need to update NAT (`pip install --upgrade nvidia-nat`). Alternatively, measure tokens by parsing the raw LLM response's `usage` field — check NAT's documentation for how to access raw responses.

5. **Tool Calling agent makes parallel tool calls**: Some models (especially GPT-4-class models) emit multiple tool calls in a single response. The NIM Llama model may or may not do this. If it does, your `tools.py` functions must be safe to call in any order. The `write_summary` tool depends on prior lookups being complete — this can cause issues.

6. **Inconsistent results across runs**: LLMs are non-deterministic. The same agent may take different paths on the same query. Run each agent 2-3 times and average the metrics. Set `temperature: 0` in the YAML configs to reduce variance.

7. **`results/` directory does not exist**: The `write_summary` tool writes to `results/`. Create the directory before running, or add `os.makedirs("results", exist_ok=True)` at the top of `tools.py`.

## Extension Ideas

1. **Add a fifth pattern — human-in-the-loop**: Modify the Tool Calling agent to pause and ask for user confirmation before executing the `write_summary` tool. Measure how this affects the user experience and compare the "trust boundary" concept across patterns.

2. **Stress test with harder queries**: Create 3 queries that are deliberately adversarial: ambiguous country names, calculations that require intermediate results, or queries where the answer is "not in the database." Observe which patterns degrade gracefully and which crash.

3. **Visualize the execution traces**: Parse the trace JSON files and create a visual diagram (using Mermaid or graphviz) showing the sequence of LLM calls and tool calls for each agent on the same query. This makes the architectural differences viscerally clear.
