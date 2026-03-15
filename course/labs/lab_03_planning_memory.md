# Lab 3: Planning and Memory Integration

## Interaction Classification
- **Type**: Local NVIDIA tooling interaction
- **NVIDIA Services Used**: NAT Memory systems, NAT Object Stores (Redis), NAT Reasoning Agent, NIM API
- **Credentials Required**: NVIDIA_API_KEY from build.nvidia.com
- **Hardware**: CPU laptop + Docker (for Redis)
- **Milestone**: Milestone 2 (First NAT workflow, continued)

## Objective

Build an agent with **persistent memory** and **planning capability** that can handle multi-session conversations and complex task decomposition. You will start with NAT's in-memory backend for rapid development, graduate to Redis for persistence across process restarts, and combine memory with ReWOO-style planning so the agent can decompose complex tasks, persist intermediate results, and resume work after a restart.

## Prerequisites

| Requirement | Details |
|---|---|
| Modules | Module 3 — Planning and Memory |
| Prior labs | Lab 2 (familiarity with NAT agent types, especially ReWOO and Reasoning) |
| Software | Python 3.10+, `nvidia-nat`, Redis (via Docker or local install) |
| Accounts | NVIDIA NGC account with NIM API key |
| Hardware | CPU-only; Redis needs ~50MB RAM |

Install Redis via Docker (recommended):

```bash
docker run -d --name redis-lab3 -p 6379:6379 redis:7-alpine
```

Or install locally: `brew install redis && redis-server --daemonize yes` (macOS).

## Deliverables

1. `memory_agent.yaml` — NAT config with Automatic Memory Wrapper and in-memory backend
2. `memory_agent_redis.yaml` — same config but with Redis backend
3. `planning_agent.yaml` — NAT config with ReWOO planning
4. `combined_agent.yaml` — agent config combining memory + planning
5. `agent.py` — Python driver that runs the agents
6. `test_memory.py` — script that proves memory works across sessions
7. `test_planning.py` — script that tests multi-step task decomposition
8. `test_resume.py` — script that proves the agent can resume after restart

## Recommended Repo Structure

```
lab_03_planning_memory/
├── .env
├── configs/
│   ├── memory_agent.yaml
│   ├── memory_agent_redis.yaml
│   ├── planning_agent.yaml
│   └── combined_agent.yaml
├── agent.py
├── tools.py
├── test_memory.py
├── test_planning.py
├── test_resume.py
├── requirements.txt
└── docker-compose.yaml       # Optional: Redis service
```

## Implementation Steps

### Step 1: Configure NAT memory with in-memory backend

Create `configs/memory_agent.yaml`:

```yaml
agent:
  type: tool_calling
  llm:
    type: nim
    model: meta/llama-3.1-70b-instruct
    api_key: ${NVIDIA_API_KEY}
    base_url: https://integrate.api.nvidia.com/v1
    max_tokens: 1024
    temperature: 0
  system_prompt: >
    You are a helpful research assistant with memory. You remember
    previous conversations and use that context to provide better answers.
    When a user refers to something discussed earlier, use your memory
    to recall it.
  memory:
    type: automatic
    backend:
      type: in_memory
    max_memories: 100
    relevance_threshold: 0.7
  functions:
    - name: note_taker
      description: "Save a note for future reference. Use this to record important facts the user tells you."
      parameters:
        type: object
        properties:
          topic:
            type: string
            description: "Short topic label"
          content:
            type: string
            description: "The note content"
        required: [topic, content]
    - name: search_notes
      description: "Search previously saved notes by topic keyword."
      parameters:
        type: object
        properties:
          query:
            type: string
            description: "Search query"
        required: [query]
```

### Step 2: Implement tool functions and driver

Create `tools.py`:

```python
import nat
import json

# In-memory note store (will be replaced by Redis in Step 3)
_notes = []

@nat.function("note_taker")
def note_taker(topic: str, content: str) -> str:
    _notes.append({"topic": topic, "content": content})
    return f"Note saved: [{topic}] {content}"

@nat.function("search_notes")
def search_notes(query: str) -> str:
    query_lower = query.lower()
    matches = [n for n in _notes if query_lower in n["topic"].lower() or query_lower in n["content"].lower()]
    if not matches:
        return "No matching notes found."
    return json.dumps(matches, indent=2)

@nat.function("calculator")
def calculator(expression: str) -> str:
    try:
        result = eval(expression, {"__builtins__": {}}, {"abs": abs, "max": max, "min": min})
        return str(result)
    except Exception as e:
        return f"Error: {e}"
```

Create `agent.py`:

```python
import nat
import os
from dotenv import load_dotenv

load_dotenv()
import tools  # noqa: F401 — registers tool functions

def run_interactive(config_path: str):
    """Run an interactive conversation loop with the agent."""
    agent = nat.from_yaml(config_path)
    print(f"Agent loaded from {config_path}")
    print("Type 'quit' to exit, 'reset' to clear conversation.\n")

    session_id = "user-001"

    while True:
        user_input = input("You: ").strip()
        if user_input.lower() == "quit":
            break
        if user_input.lower() == "reset":
            agent.reset(session_id=session_id)
            print("Conversation reset.\n")
            continue

        response = agent.run(user_input, session_id=session_id)
        print(f"Agent: {response}\n")

if __name__ == "__main__":
    import sys
    config = sys.argv[1] if len(sys.argv) > 1 else "configs/memory_agent.yaml"
    run_interactive(config)
```

Test the in-memory agent:

```bash
python agent.py configs/memory_agent.yaml
```

Have a conversation like:

```
You: My name is Alex and I'm studying for the NCP-AAI certification.
You: I'm particularly interested in RAG pipelines.
You: What topics am I interested in?  # Agent should recall RAG pipelines
```

### Step 3: Switch to Redis for persistent memory

Create `configs/memory_agent_redis.yaml`:

```yaml
agent:
  type: tool_calling
  llm:
    type: nim
    model: meta/llama-3.1-70b-instruct
    api_key: ${NVIDIA_API_KEY}
    base_url: https://integrate.api.nvidia.com/v1
    max_tokens: 1024
    temperature: 0
  system_prompt: >
    You are a helpful research assistant with persistent memory. You remember
    previous conversations even across restarts.
  memory:
    type: automatic
    backend:
      type: redis
      url: redis://localhost:6379
      prefix: "lab3:memory:"
    max_memories: 100
    relevance_threshold: 0.7
  functions:
    - name: note_taker
      description: "Save a note for future reference."
      parameters:
        type: object
        properties:
          topic:
            type: string
          content:
            type: string
        required: [topic, content]
    - name: search_notes
      description: "Search previously saved notes by topic keyword."
      parameters:
        type: object
        properties:
          query:
            type: string
        required: [query]
    - name: calculator
      description: "Evaluate a mathematical expression."
      parameters:
        type: object
        properties:
          expression:
            type: string
        required: [expression]
```

Verify Redis is running:

```bash
redis-cli ping
# Should return: PONG
```

### Step 4: Write the memory persistence test

Create `test_memory.py`:

```python
"""
Proves that memory persists across agent restarts when using Redis.

Session 1: Tell the agent some facts.
Session 2: Restart the agent (new Python process) and ask about those facts.
"""
import nat
import os
import subprocess
import sys
from dotenv import load_dotenv

load_dotenv()
import tools  # noqa: F401

CONFIG = "configs/memory_agent_redis.yaml"
SESSION_ID = "test-user-memory"

def session_1():
    """First session: provide facts to the agent."""
    agent = nat.from_yaml(CONFIG)
    print("=== SESSION 1: Providing facts ===")

    responses = []
    queries = [
        "Remember this: my favorite programming language is Rust.",
        "Also remember: I have 3 years of experience with machine learning.",
        "One more thing: my project deadline is March 30th.",
    ]
    for q in queries:
        r = agent.run(q, session_id=SESSION_ID)
        print(f"  You: {q}")
        print(f"  Agent: {r}\n")
        responses.append(r)
    return responses

def session_2():
    """Second session: new agent instance, test recall."""
    agent = nat.from_yaml(CONFIG)  # Fresh instance — no in-memory state
    print("=== SESSION 2: Testing recall ===")

    test_queries = [
        "What is my favorite programming language?",
        "How much ML experience do I have?",
        "When is my deadline?",
    ]
    results = []
    for q in test_queries:
        r = agent.run(q, session_id=SESSION_ID)
        print(f"  You: {q}")
        print(f"  Agent: {r}\n")
        results.append(r)
    return results

if __name__ == "__main__":
    if "--session2" in sys.argv:
        session_2()
    else:
        session_1()
        print("\nNow run: python test_memory.py --session2")
        print("This simulates a restart — the agent should still remember the facts.")
```

Run it in two steps to prove persistence:

```bash
python test_memory.py            # Session 1: provide facts
python test_memory.py --session2 # Session 2: verify recall
```

### Step 5: Implement a planning agent with ReWOO

Create `configs/planning_agent.yaml`:

```yaml
agent:
  type: rewoo
  llm:
    type: nim
    model: meta/llama-3.1-70b-instruct
    api_key: ${NVIDIA_API_KEY}
    base_url: https://integrate.api.nvidia.com/v1
    max_tokens: 2048
    temperature: 0
  system_prompt: >
    You are a research assistant that plans before acting. When given a complex
    task, first create a step-by-step plan, then execute each step using your
    tools. Report the results of each step.
  functions:
    - name: note_taker
      description: "Save a note for future reference."
      parameters:
        type: object
        properties:
          topic:
            type: string
          content:
            type: string
        required: [topic, content]
    - name: search_notes
      description: "Search previously saved notes."
      parameters:
        type: object
        properties:
          query:
            type: string
        required: [query]
    - name: calculator
      description: "Evaluate a mathematical expression."
      parameters:
        type: object
        properties:
          expression:
            type: string
        required: [expression]
```

Create `test_planning.py`:

```python
import nat
import os
from dotenv import load_dotenv

load_dotenv()
import tools  # noqa: F401

agent = nat.from_yaml("configs/planning_agent.yaml")

COMPLEX_TASK = """
I need you to do the following multi-step research task:
1. Calculate 2^10, 3^7, and 5^5.
2. Save each result as a separate note with the expression as the topic.
3. Then search your notes for all saved calculations.
4. Finally, calculate the sum of all three results and save that as a note too.
Report back with all the numbers and the final sum.
"""

print("Task:", COMPLEX_TASK.strip())
print("\nAgent working...\n")

with nat.profiler() as prof:
    response = agent.run(COMPLEX_TASK)

print(f"Response:\n{response}")
print(f"\nProfiling: {prof.llm_call_count} LLM calls, {prof.tool_call_count} tool calls, {prof.total_tokens} tokens")
```

Run it:

```bash
python test_planning.py
```

Observe: the ReWOO agent should create the entire plan in its first LLM call, then execute all tool calls without intermediate LLM reasoning.

### Step 6: Combine memory + planning with resume capability

Create `configs/combined_agent.yaml`:

```yaml
agent:
  type: rewoo
  llm:
    type: nim
    model: meta/llama-3.1-70b-instruct
    api_key: ${NVIDIA_API_KEY}
    base_url: https://integrate.api.nvidia.com/v1
    max_tokens: 2048
    temperature: 0
  system_prompt: >
    You are a research assistant with persistent memory and planning ability.
    You can plan multi-step tasks and remember results across sessions.
    When resuming a task, check your memory for prior progress before
    starting from scratch.
  memory:
    type: automatic
    backend:
      type: redis
      url: redis://localhost:6379
      prefix: "lab3:combined:"
    max_memories: 200
    relevance_threshold: 0.6
  functions:
    - name: note_taker
      description: "Save a note for future reference."
      parameters:
        type: object
        properties:
          topic:
            type: string
          content:
            type: string
        required: [topic, content]
    - name: search_notes
      description: "Search previously saved notes."
      parameters:
        type: object
        properties:
          query:
            type: string
        required: [query]
    - name: calculator
      description: "Evaluate a mathematical expression."
      parameters:
        type: object
        properties:
          expression:
            type: string
        required: [expression]
```

Create `test_resume.py`:

```python
"""
Demonstrates task resumption across sessions.

Session 1: Start a multi-step task, complete steps 1-2.
Session 2: Restart the agent, ask it to continue where it left off.
"""
import nat
import os
import sys
from dotenv import load_dotenv

load_dotenv()
import tools  # noqa: F401

CONFIG = "configs/combined_agent.yaml"
SESSION_ID = "resume-test-user"

def session_1():
    agent = nat.from_yaml(CONFIG)
    print("=== SESSION 1: Start a multi-step research task ===\n")

    task = (
        "I need to analyze three calculations: 17*23, 45*67, and 89*12. "
        "Calculate the first two and save the results as notes. "
        "We'll do the third one later."
    )
    response = agent.run(task, session_id=SESSION_ID)
    print(f"Agent: {response}\n")

def session_2():
    agent = nat.from_yaml(CONFIG)
    print("=== SESSION 2: Resume the task ===\n")

    task = (
        "We were working on calculating three products earlier. "
        "I've done 17*23 and 45*67 already. "
        "Please check your memory for the results, then calculate the third one (89*12), "
        "and give me the sum of all three."
    )
    response = agent.run(task, session_id=SESSION_ID)
    print(f"Agent: {response}\n")

if __name__ == "__main__":
    if "--session2" in sys.argv:
        session_2()
    else:
        session_1()
        print("Now run: python test_resume.py --session2")
```

Run it:

```bash
python test_resume.py
python test_resume.py --session2
```

Verify that Session 2 retrieves the results from Session 1's memory rather than recalculating everything.

## Evaluation Rubric

| Criterion | Excellent (5) | Adequate (3) | Incomplete (1) |
|---|---|---|---|
| **In-memory agent works** | Multi-turn conversation with correct recall within a session | Recall works sometimes | Agent does not remember prior turns |
| **Redis persistence** | Facts survive across two separate Python processes (test_memory.py passes) | Redis is configured but recall is unreliable | No Redis integration |
| **Planning works** | ReWOO agent decomposes a 4+ step task and executes all steps | Agent plans but misses some steps | No planning implementation |
| **Resume capability** | Session 2 correctly retrieves Session 1 results from memory and builds on them | Partial recall on resume | Cannot resume |
| **Test scripts** | All 3 test scripts run successfully and demonstrate their purpose clearly | 2 of 3 test scripts work | Missing or non-functional tests |

## Likely Bugs and Failure Cases

1. **Redis connection refused**: Docker container is not running. Run `docker ps` to check. If the container stopped, restart it: `docker start redis-lab3`. If you never created it, go back to the Prerequisites section.

2. **Memory retrieval returns irrelevant results**: The `relevance_threshold` is set too low (e.g., 0.3), causing the agent to retrieve memories that are not relevant to the current query. Start at 0.7 and tune down if the agent misses things it should remember.

3. **Automatic Memory Wrapper stores too much**: The wrapper may store every turn, including filler like "OK" and "Got it." This fills up the memory store with low-value entries. If this happens, increase the relevance threshold or configure memory filtering rules (check NAT documentation for `memory.filter` options).

4. **ReWOO plan references results that do not exist yet**: Because ReWOO plans upfront, it may write a plan like "Step 3: Calculate the sum of Step 1 result and Step 2 result" but the actual tool call has no access to those variable names. The tool will receive literal strings like "Step 1 result" instead of numbers. This is a known limitation — the system prompt must instruct the agent to use concrete values in plans.

5. **Session ID mismatch**: If you forget to pass `session_id` to `agent.run()`, the memory backend may use a default session ID, meaning all users share memory. Always pass an explicit `session_id`.

6. **Redis data not cleared between test runs**: Old test data in Redis can confuse new test runs. Flush the lab's keys before re-running: `redis-cli --scan --pattern "lab3:*" | xargs redis-cli del`. Do not run `FLUSHALL` — it will delete everything in Redis.

7. **`nat.from_yaml()` called multiple times creates separate agent instances**: Each call creates a new agent. If you call it in `session_1` and again in `session_2`, the in-memory backend will be empty in Session 2 (that is the whole point of testing Redis). Make sure you understand this distinction.

## Extension Ideas

1. **Memory summarization**: After 50+ memories accumulate, implement a summarization step where the agent condenses old memories into higher-level summaries. This mirrors how production systems manage growing context windows.

2. **Hierarchical planning**: Modify the combined agent to handle tasks with sub-tasks. For example: "Research 3 topics, write a summary for each, then write a meta-summary." The agent should plan the top level, then plan each sub-task independently. This may require switching from ReWOO (which does not re-plan) to a Reasoning agent with explicit planning prompts.

3. **Memory-aware tool selection**: Add 5+ tools to the agent and observe how memory influences tool selection. For example, if the user previously said "I prefer detailed explanations," does the agent's behavior change? Experiment with injecting memory into the system prompt vs. relying on NAT's automatic injection.
