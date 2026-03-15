# Lab 7: Multi-Agent System with Router and A2A

## Interaction Classification
- **Type**: Local NVIDIA tooling interaction
- **NVIDIA Services Used**: NAT Router Agent, NAT A2A Server, NIM API
- **Credentials Required**: NVIDIA_API_KEY from build.nvidia.com
- **Hardware**: CPU laptop sufficient
- **Milestone**: Milestone 3 (continued)

> **Package note**: This lab uses `nvidia-nat` (installed via `pip install nvidia-nat`). Python imports use `from nat...`. If you see references to `nvidia-agent-toolkit` or `nvidia_agent_toolkit` elsewhere, those are the same toolkit under a different name that may appear in some NVIDIA examples. This course standardizes on `nvidia-nat`.

## Objective

Build a multi-agent system where a **Router Agent** dispatches user queries to three specialized sub-agents, and agents communicate via the **A2A (Agent-to-Agent) protocol**. By the end of this lab you will have a working system in which a single user request can fan out to multiple specialists, results are aggregated, and the router returns a unified answer. This lab is the bridge from single-agent pipelines (Labs 5-6) to production-scale orchestration.

## Prerequisites

| Requirement | Detail |
|---|---|
| Module 7 | Multi-agent architectures, A2A protocol, router patterns |
| Lab 6 | Working single-agent pipeline with tools and memory |
| Software | Python 3.10+, `nvidia-nat`, `uvicorn`, `httpx` |
| API access | NVIDIA NIM endpoint **or** `NVIDIA_API_KEY` for build.nvidia.com |

## Deliverables

1. Three specialist agent definitions (NAT workflow YAML or Python)
2. A Router Agent configuration that dispatches to the correct specialist
3. Three running A2A servers (one per specialist)
4. An integration test script that sends 10+ queries and asserts correct routing
5. A latency profile report (CSV or JSON) showing coordination overhead
6. A brief write-up (< 1 page) of failure-handling strategy

## Recommended Repo Structure

```
lab_07_multi_agent/
├── agents/
│   ├── research_agent.py        # Specialist 1: web search & summarization
│   ├── code_agent.py            # Specialist 2: code generation & review
│   └── data_agent.py            # Specialist 3: data analysis & visualization
├── router/
│   ├── router_agent.py          # Router Agent logic
│   └── router_config.yaml       # Routing rules and agent registry
├── a2a/
│   ├── server_research.py       # A2A server wrapper for research agent
│   ├── server_code.py           # A2A server wrapper for code agent
│   └── server_data.py           # A2A server wrapper for data agent
├── shared/
│   ├── state.py                 # Shared state / context passing utilities
│   └── schemas.py               # Pydantic models for inter-agent messages
├── tests/
│   ├── test_routing.py          # Routing correctness tests
│   ├── test_multi_hop.py        # Multi-agent collaboration tests
│   └── test_failures.py         # Failure and timeout tests
├── profiles/
│   └── latency_report.json      # Output from profiling step
├── requirements.txt
└── README.md
```

## Implementation Steps

### Step 1: Design Three Specialist Agents

Each specialist must have a distinct purpose, its own set of tools, and a clear system prompt that keeps it in scope.

**Research Agent** -- searches the web and summarizes findings:

```python
# agents/research_agent.py
from nat.agents import Agent
from nat.tools import WebSearchTool, SummarizerTool

research_agent = Agent(
    name="ResearchAgent",
    description="Searches the web and summarizes factual information.",
    system_prompt=(
        "You are a research specialist. Use the web_search tool to find "
        "information, then use the summarizer tool to produce a concise answer. "
        "Always cite your sources with URLs."
    ),
    llm="meta/llama-3.1-70b-instruct",   # or your NIM endpoint
    tools=[
        WebSearchTool(),
        SummarizerTool(),
    ],
)
```

**Code Agent** -- generates, explains, and reviews code:

```python
# agents/code_agent.py
from nat.agents import Agent
from nat.tools import CodeInterpreterTool

code_agent = Agent(
    name="CodeAgent",
    description="Generates, explains, and reviews code in Python and other languages.",
    system_prompt=(
        "You are a coding specialist. Write clean, well-commented code. "
        "When reviewing code, list issues by severity. Use the code_interpreter "
        "tool to validate your code before returning it."
    ),
    llm="meta/llama-3.1-70b-instruct",
    tools=[
        CodeInterpreterTool(),
    ],
)
```

**Data Analysis Agent** -- analyzes datasets and produces insights:

```python
# agents/data_agent.py
from nat.agents import Agent
from nat.tools import CodeInterpreterTool, FileReadTool

data_agent = Agent(
    name="DataAgent",
    description="Analyzes CSV/JSON data, computes statistics, and creates visualizations.",
    system_prompt=(
        "You are a data analysis specialist. Load the provided data, compute "
        "descriptive statistics, identify trends, and produce matplotlib charts "
        "when requested. Always show your methodology."
    ),
    llm="meta/llama-3.1-70b-instruct",
    tools=[
        CodeInterpreterTool(),
        FileReadTool(),
    ],
)
```

### Step 2: Configure Each Agent as a Separate NAT Workflow

Wrap each agent in a standalone workflow so it can be invoked independently and later served via A2A.

```python
# Example: wrapping the research agent
from nat.workflows import AgentWorkflow

research_workflow = AgentWorkflow(
    agent=research_agent,
    max_steps=10,
    verbose=True,
)

# Smoke-test the workflow in isolation
result = research_workflow.run("What are the latest advances in quantum computing?")
print(result)
```

Repeat for `code_agent` and `data_agent`. Ensure each workflow completes successfully on at least 3 sample queries **before** moving to routing.

### Step 3: Build the Router Agent

The Router Agent receives every user query and decides which specialist(s) to invoke. Use NAT's routing capability or build a lightweight routing function.

```python
# router/router_agent.py
from nat.agents import Agent
from nat.tools import AgentTool

# Create AgentTools that point to the specialists
research_tool = AgentTool(
    agent=research_agent,
    name="research_specialist",
    description="Delegates factual research questions, news lookups, and summarization tasks.",
)

code_tool = AgentTool(
    agent=code_agent,
    name="code_specialist",
    description="Delegates code generation, code review, debugging, and programming questions.",
)

data_tool = AgentTool(
    agent=data_agent,
    name="data_specialist",
    description="Delegates data analysis, statistics, CSV/JSON processing, and visualization tasks.",
)

router_agent = Agent(
    name="RouterAgent",
    description="Routes user queries to the best specialist agent.",
    system_prompt=(
        "You are a routing agent. Analyze the user's request and delegate it to "
        "the most appropriate specialist using the available tools. If a request "
        "needs multiple specialists, call them in sequence and combine results. "
        "Never attempt to answer directly -- always delegate."
    ),
    llm="meta/llama-3.1-70b-instruct",
    tools=[research_tool, code_tool, data_tool],
)
```

Create `router_config.yaml` for declarative tuning:

```yaml
# router/router_config.yaml
router:
  strategy: llm_routing          # let the LLM pick the specialist
  fallback: research_specialist  # default if classification fails
  max_delegations: 3             # cap fan-out per query
  timeout_seconds: 60            # per-specialist timeout

agents:
  research_specialist:
    url: "http://localhost:8001"  # A2A endpoint (Step 4)
    health: "/health"
  code_specialist:
    url: "http://localhost:8002"
    health: "/health"
  data_specialist:
    url: "http://localhost:8003"
    health: "/health"
```

### Step 4: Deploy Specialists as A2A Servers

Use NAT's A2A server to expose each specialist as an HTTP service.

```python
# a2a/server_research.py
from nat.a2a import A2AServer

server = A2AServer(
    agent=research_agent,
    host="0.0.0.0",
    port=8001,
    name="ResearchAgent",
    description="Searches the web and summarizes factual information.",
    skills=[
        {
            "name": "web_research",
            "description": "Search the web and summarize findings with citations.",
        }
    ],
)

if __name__ == "__main__":
    server.run()
```

Repeat for code (port 8002) and data (port 8003). Start all three:

```bash
# Terminal 1
python a2a/server_research.py

# Terminal 2
python a2a/server_code.py

# Terminal 3
python a2a/server_data.py
```

Verify each is healthy:

```bash
curl http://localhost:8001/health
curl http://localhost:8002/health
curl http://localhost:8003/health
```

### Step 5: Configure A2A Communication Between Router and Specialists

Update the Router Agent to use A2A client calls instead of direct Python imports.

```python
# router/router_agent.py (updated for A2A)
from nat.a2a import A2AClient

# Create A2A clients for each specialist
research_client = A2AClient(url="http://localhost:8001")
code_client = A2AClient(url="http://localhost:8002")
data_client = A2AClient(url="http://localhost:8003")

# Discover agent capabilities via A2A agent card
for client in [research_client, code_client, data_client]:
    card = client.get_agent_card()
    print(f"Agent: {card.name}, Skills: {[s['name'] for s in card.skills]}")

# Send a task via A2A protocol
response = research_client.send_task(
    message="What are the latest advances in quantum computing?",
)
print(response.result)
```

Wire these clients into the Router Agent's tool definitions so the LLM's tool calls go over A2A:

```python
from nat.tools import A2ATool

research_a2a_tool = A2ATool(
    client=research_client,
    name="research_specialist",
    description="Delegates factual research questions via A2A.",
)

# Repeat for code_client and data_client, then pass to router_agent.tools
```

### Step 6: Implement Shared State Management

Agents operating on the same user request need shared context. Implement a lightweight shared state mechanism.

```python
# shared/state.py
from typing import Any
import json
import redis

class SharedState:
    """Redis-backed shared state for multi-agent coordination."""

    def __init__(self, redis_url: str = "redis://localhost:6379"):
        self.redis = redis.from_url(redis_url)

    def set_context(self, session_id: str, key: str, value: Any) -> None:
        """Store a value in the shared context for a session."""
        self.redis.hset(f"session:{session_id}", key, json.dumps(value))

    def get_context(self, session_id: str, key: str) -> Any:
        """Retrieve a value from the shared context."""
        raw = self.redis.hget(f"session:{session_id}", key)
        return json.loads(raw) if raw else None

    def get_full_context(self, session_id: str) -> dict:
        """Get all shared state for a session."""
        raw = self.redis.hgetall(f"session:{session_id}")
        return {k.decode(): json.loads(v) for k, v in raw.items()}

    def expire_session(self, session_id: str, ttl: int = 3600) -> None:
        """Set TTL on a session to auto-clean."""
        self.redis.expire(f"session:{session_id}", ttl)
```

Pass the shared state into each A2A task so specialists can read prior results:

```python
shared = SharedState()
session_id = "user-123-request-456"

# After research completes, store its findings
shared.set_context(session_id, "research_results", response.result)

# Data agent can read research results if needed
context = shared.get_full_context(session_id)
```

### Step 7: Test with Single-Agent and Multi-Agent Queries

Create a test suite that covers routing correctness and multi-agent collaboration.

```python
# tests/test_routing.py
import pytest
from router.router_agent import router_agent

SINGLE_AGENT_CASES = [
    ("What is the capital of France?", "research_specialist"),
    ("Write a Python function to sort a list", "code_specialist"),
    ("Analyze this CSV and find the mean of column A", "data_specialist"),
]

@pytest.mark.parametrize("query,expected_agent", SINGLE_AGENT_CASES)
def test_single_routing(query, expected_agent):
    """Verify the router dispatches to the correct specialist."""
    result = router_agent.run(query, return_trace=True)
    tools_called = [step.tool_name for step in result.trace if step.tool_name]
    assert expected_agent in tools_called, (
        f"Expected {expected_agent} but got {tools_called}"
    )

MULTI_AGENT_CASES = [
    "Research the latest Python 3.13 features and write example code for each",
    "Find a dataset about global temperatures, then analyze the trend",
]

@pytest.mark.parametrize("query", MULTI_AGENT_CASES)
def test_multi_agent_collaboration(query):
    """Verify queries needing multiple specialists invoke more than one."""
    result = router_agent.run(query, return_trace=True)
    tools_called = set(step.tool_name for step in result.trace if step.tool_name)
    assert len(tools_called) >= 2, (
        f"Expected multi-agent collaboration but only got {tools_called}"
    )
```

Run the tests:

```bash
pytest tests/ -v --tb=short
```

### Step 8: Handle Failure Cases

Implement resilience for each failure mode.

**Specialist unavailable:**

```python
# router/router_agent.py -- add retry and fallback logic
import httpx
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(min=1, max=10))
async def call_specialist(client: A2AClient, message: str) -> str:
    try:
        response = await client.send_task_async(message, timeout=60)
        return response.result
    except httpx.ConnectError:
        raise ConnectionError(f"Specialist at {client.url} is unavailable")
```

**Timeout handling:**

```python
import asyncio

async def call_with_timeout(client, message, timeout=60):
    try:
        return await asyncio.wait_for(
            client.send_task_async(message),
            timeout=timeout,
        )
    except asyncio.TimeoutError:
        return {"error": "specialist_timeout", "agent": client.url}
```

**Conflicting responses** (when two specialists give contradictory answers):

```python
def reconcile_responses(responses: list[dict]) -> str:
    """When multiple specialists respond, reconcile conflicts."""
    # Feed all responses back to the router LLM for synthesis
    combined = "\n\n".join(
        f"[{r['agent']}]: {r['result']}" for r in responses
    )
    synthesis_prompt = (
        f"Multiple specialists provided the following answers. "
        f"Synthesize a single coherent response, noting any conflicts:\n\n{combined}"
    )
    return router_agent.llm.invoke(synthesis_prompt)
```

Write failure tests:

```python
# tests/test_failures.py
def test_specialist_unavailable():
    """Kill one A2A server and verify graceful degradation."""
    # Stop the research server, then send a research query
    result = router_agent.run("What is the latest news on AI?")
    assert "unavailable" in result.lower() or len(result) > 0  # graceful fallback

def test_timeout_handling():
    """Send a query to a specialist that will exceed timeout."""
    # Configure a 2-second timeout and send a complex query
    result = router_agent.run("Analyze all public datasets on climate change")
    assert result is not None  # should not crash
```

### Step 9: Profile the Multi-Agent System

Measure coordination overhead using NAT's profiler.

```python
# profiles/run_profiling.py
from nat.profiler import Profiler
import json
import time

profiler = Profiler()

test_queries = [
    "What is quantum computing?",                           # single-agent
    "Write a quicksort in Python",                          # single-agent
    "Research AI trends and write code examples for each",  # multi-agent
    "Find GDP data and create a visualization",             # multi-agent
]

results = []
for query in test_queries:
    start = time.perf_counter()
    with profiler.trace() as trace:
        response = router_agent.run(query)
    elapsed = time.perf_counter() - start

    results.append({
        "query": query,
        "total_time_s": round(elapsed, 2),
        "total_tokens": trace.total_tokens,
        "llm_calls": trace.llm_call_count,
        "tool_calls": trace.tool_call_count,
        "agents_used": trace.agents_invoked,
        "routing_overhead_s": round(trace.routing_time_s, 3),
    })

with open("profiles/latency_report.json", "w") as f:
    json.dump(results, f, indent=2)

# Print summary
for r in results:
    overhead_pct = (r["routing_overhead_s"] / r["total_time_s"]) * 100
    print(f"Query: {r['query'][:50]}...")
    print(f"  Total: {r['total_time_s']}s | Routing overhead: {overhead_pct:.1f}%")
    print(f"  Tokens: {r['total_tokens']} | LLM calls: {r['llm_calls']}")
    print()
```

## Evaluation Rubric

| Criterion | Excellent (5) | Satisfactory (3) | Needs Work (1) |
|---|---|---|---|
| **Agent design** | 3 specialists with distinct tools, clear system prompts, tested in isolation | 3 agents defined but overlap in scope or untested | Fewer than 3 agents or poorly differentiated |
| **Router accuracy** | Routes correctly for 90%+ of test cases including ambiguous queries | Routes correctly for 70-89% of cases | Below 70% or frequent misroutes |
| **A2A integration** | All 3 specialists served via A2A, agent cards discoverable, protocol used correctly | A2A servers run but some protocol details skipped | Agents called directly (no A2A) |
| **Multi-agent collaboration** | Queries requiring 2+ agents handled with result synthesis | Multi-agent queries attempted but results not combined well | Only single-agent routing works |
| **Failure handling** | All 3 failure modes (unavailable, timeout, conflict) handled gracefully with tests | 2 of 3 failure modes handled | No failure handling |
| **Shared state** | Redis-backed state passing between agents within a session | State passing works but no persistence | No shared state mechanism |
| **Profiling** | Latency report with routing overhead analysis, at least 10 queries profiled | Some profiling done but incomplete analysis | No profiling |

**Minimum passing score**: 21/35 (average 3 across all criteria)

## Likely Bugs / Failure Cases

1. **Port conflicts**: If ports 8001-8003 are already in use, the A2A servers fail silently or crash. Fix: check ports before starting (`lsof -i :8001`) and configure alternative ports in the YAML.

2. **Circular routing**: The router calls itself or a specialist calls back to the router, creating an infinite loop. Fix: set `max_delegations` in the config and ensure specialists do not have routing tools.

3. **A2A serialization errors**: Complex objects (DataFrames, matplotlib figures) cannot be serialized over A2A JSON. Fix: convert all tool outputs to strings or base64-encoded images before returning.

4. **Redis not running**: SharedState fails with `ConnectionRefusedError` if Redis is not started. Fix: add a health check at startup and provide a fallback in-memory dict for development.

5. **LLM misroutes ambiguous queries**: "Help me with this data" could go to any specialist. Fix: improve specialist descriptions, add few-shot examples to the router's system prompt, or implement a confidence threshold that triggers clarification.

6. **Token budget explosion**: Multi-agent queries can consume 3-5x the tokens of single-agent queries because each specialist gets the full context. Fix: implement context summarization before passing to the next specialist.

7. **Race conditions in shared state**: If two specialists write to the same key concurrently, one result is lost. Fix: use Redis transactions or append-only keys.

## Extension Ideas

1. **Dynamic agent registration**: Build an agent registry where new specialists can register themselves at runtime. The router discovers new agents via their A2A agent cards and incorporates them without restart.

2. **Parallel fan-out with streaming**: For queries that need all three specialists, dispatch in parallel using `asyncio.gather()` and stream partial results back to the user as each specialist completes.

3. **Agent voting and consensus**: For high-stakes queries, send the same question to multiple specialists and implement a voting mechanism -- only return an answer if 2+ agents agree.
