# Module 3: Reasoning, Planning, and Memory

**Primary exam domains:** Cognition, Planning, and Memory (10%)
**NVIDIA tools:** NAT Reasoning Agent, NAT ReWOO Agent, NAT Test Time Compute, NAT Memory systems, NAT Object Stores (in-memory, MySQL, Redis, S3), NAT Automatic Memory Wrapper

---

## A. Module Title

**Reasoning, Planning, and Memory**

---

## B. Why This Module Matters in Real Systems

An agent that cannot reason well will call the wrong tool, misinterpret results, and produce confident-sounding nonsense. An agent that cannot plan will waste tokens exploring dead ends or repeat the same failed approach. An agent without memory will ask the user to repeat information, lose track of multi-turn conversations, and fail to learn from past interactions. Reasoning, planning, and memory are not optional enhancements — they are the cognitive infrastructure that separates a useful agent from an expensive random number generator.

In production systems, these three capabilities interact in specific ways. A financial research agent must reason about which data sources are relevant (reasoning), decompose a complex analysis into ordered steps (planning), and remember what the user asked about in previous sessions so it does not re-fetch the same data (memory). Getting any one of these wrong has concrete consequences: wrong reasoning means wrong answers, poor planning means blown latency budgets, and missing memory means frustrated users who feel like they are talking to a goldfish.

The NVIDIA NeMo Agent Toolkit provides specific implementations for each: the Reasoning Agent and Test Time Compute for enhanced reasoning, the ReWOO Agent for structured planning, and a configurable memory system with multiple backend options (in-memory, Redis, MySQL, S3) plus an Automatic Memory Wrapper for transparent history management. This module teaches you how these work, when to use each option, and what the limits are. Where NAT provides specific implementation details, we cover them. Where the research literature provides useful frameworks that NAT does not explicitly implement, we say so.

---

## C. Learning Objectives

By the end of this module, you will be able to:

1. Explain four reasoning strategies (Chain-of-Thought, Tree-of-Thought, self-reflection, self-consistency) and describe when each is appropriate.
2. Configure and use the NAT Reasoning Agent and describe how Test Time Compute enhances response quality.
3. Distinguish plan-then-execute from interleaved planning and implement a planning agent using NAT ReWOO.
4. Classify memory into four types (short-term, long-term, episodic, semantic) and map each to appropriate storage backends.
5. Configure NAT memory with each Object Store backend (in-memory, Redis, MySQL, S3) and explain when to use each.
6. Implement the NAT Automatic Memory Wrapper for transparent conversation history management.
7. Determine when context window size is sufficient vs. when external persistent memory is required.
8. Design a memory architecture for a given agent system, including capacity planning and retrieval strategy.

---

## D. Required Concepts

Before starting this module, you should be comfortable with:

- **Module 1-2 content**: Agent loop, NVIDIA stack, NAT agent types, ReAct and ReWOO patterns.
- **Prompt engineering basics**: System prompts, few-shot examples, chain-of-thought prompting.
- **Database concepts**: Key-value stores, relational databases, object storage. Basic Redis commands. Basic SQL.
- **Context windows**: What a context window is, how token limits affect what the model can process.
- **Caching patterns**: The concept of caching, TTL, eviction policies.

---

## E. Core Lesson Content

### [BEGINNER] Reasoning Strategies

Reasoning is how an agent derives conclusions from available information. An LLM does not "reason" in the human sense — it generates token sequences that follow patterns learned during training. But specific prompting and architecture strategies can elicit more reliable, step-by-step outputs that functionally behave like reasoning. Four strategies dominate the field:

**Chain-of-Thought (CoT):**
The model is prompted to show its work step by step before giving a final answer. Instead of jumping directly to a conclusion, it generates intermediate reasoning steps.

```
Without CoT: "The answer is 42."
With CoT:    "First, I need to find X. X = 10 * 3 = 30.
              Then, I add Y. Y = 12.
              Total = 30 + 12 = 42."
```

CoT improves accuracy on tasks requiring multi-step logic (arithmetic, logical deduction, multi-hop question answering). It is the default reasoning strategy in most agent systems.

**Tree-of-Thought (ToT):**
Instead of a single chain, the model explores multiple reasoning paths and selects the best one. Think of it as branching — the model generates 3-5 possible next steps, evaluates each, and pursues the most promising one.

```
                    Problem
                   /   |   \
                 v     v     v
              Path A  Path B  Path C
              /    \    |      |
            v      v    v      v
          A1      A2   B1     C1
                        |
                        v
                    Best: B1 → Continue
```

ToT is more expensive (multiple LLM calls per step) but produces better results on tasks with many possible approaches, such as code generation or mathematical proofs.

**Self-reflection:**
After generating an answer, the model is prompted to critique its own output and revise if necessary. This is typically implemented as a second LLM call: "Here is my answer: [X]. Is this correct? Are there any errors in my reasoning?"

**Self-consistency:**
Generate multiple independent answers to the same question (using higher temperature or different prompts), then select the answer that appears most frequently. This is a voting mechanism — if 4 out of 5 attempts produce the same answer, that answer is likely correct.

### [INTERMEDIATE] NAT Reasoning Agent and Test Time Compute

The NAT Reasoning Agent implements enhanced reasoning through what NVIDIA calls **Test Time Compute (TTC)**. The core idea: spend more computation at inference time to produce a better response, rather than relying on a single forward pass.

NAT's TTC implementation involves:
1. **Multi-LLM generation**: Generating multiple candidate responses (potentially with different prompting strategies or temperature settings).
2. **Planning**: Structuring the reasoning process before executing it.
3. **Selection**: Evaluating candidates and choosing the best one.

**Important caveat:** NAT documents Test Time Compute at the feature level — you can enable it and configure it — but does not publish the internal algorithm specifics (e.g., exactly how candidates are scored, the selection mechanism, the prompting strategies used internally). What follows is based on published NAT documentation; internal implementation details may differ.

```python
from nat.agents import ReasoningAgent
from nat.llm import NIMEndpoint

llm = NIMEndpoint(
    model="meta/llama-3.1-70b-instruct",
    endpoint="https://integrate.api.nvidia.com/v1",
    api_key="nvapi-...",
)

agent = ReasoningAgent(
    llm=llm,
    # TTC configuration
    num_candidates=3,       # Generate 3 candidate responses
    reasoning_depth="deep", # Extended reasoning mode
    tools=[...],
)

response = agent.run("Analyze the Q3 revenue trends and identify anomalies.")
```

**When to use the Reasoning Agent:**
- Tasks requiring high accuracy where latency is secondary (research, analysis, complex Q&A).
- Tasks where the cost of a wrong answer is high (medical, legal, financial contexts).
- Tasks where simple CoT produces inconsistent results.

**When NOT to use it:**
- Latency-sensitive applications (TTC multiplies inference time by the number of candidates).
- Simple lookup or retrieval tasks where a single pass is sufficient.
- High-throughput systems where per-request cost must be minimized.

### [INTERMEDIATE] Planning: Task Decomposition and Goal-Directed Execution

Planning is the process of breaking a complex task into subtasks and determining the order of execution. Two primary approaches exist:

**Plan-then-execute (used by NAT ReWOO Agent):**
Generate the full plan first. Then execute each step sequentially. The plan is fixed once generated — there is no re-planning based on intermediate results.

```
┌────────────────────────────────────────────────────────┐
│              PLAN-THEN-EXECUTE                          │
│                                                        │
│  Input: "Compare revenue of companies A and B"         │
│                                                        │
│  PLAN (single LLM call):                               │
│    Step 1: search(company_A_revenue)  → #1             │
│    Step 2: search(company_B_revenue)  → #2             │
│    Step 3: compare(#1, #2)            → #3             │
│    Step 4: format_report(#3)          → final          │
│                                                        │
│  EXECUTE (no LLM calls, just tool execution):          │
│    #1 = search("company A revenue") → "$10M"           │
│    #2 = search("company B revenue") → "$15M"           │
│    #3 = compare("$10M", "$15M") → "B leads by $5M"    │
│    final = format("B leads by $5M") → report           │
│                                                        │
│  SYNTHESIZE (one LLM call):                            │
│    Combine all results into final answer               │
└────────────────────────────────────────────────────────┘
```

**Interleaved planning (used by NAT ReAct Agent):**
Plan one step at a time. After each step executes, re-evaluate and plan the next step. The plan is adaptive — it changes based on what the agent learns during execution.

**Tradeoff summary:**

| Dimension | Plan-then-Execute (ReWOO) | Interleaved (ReAct) |
|-----------|--------------------------|---------------------|
| LLM calls | 2 (plan + synthesize) | N (one per step) |
| Adaptability | Low (fixed plan) | High (replans each step) |
| Token cost | Lower | Higher |
| Failure recovery | Poor | Good |
| Best for | Predictable tasks | Exploratory tasks |

**NAT ReWOO configuration:**

```yaml
# rewoo_planning_agent.yaml
agent:
  type: rewoo
  llm:
    model: meta/llama-3.1-70b-instruct
    endpoint: ${NIM_ENDPOINT}
    temperature: 0.1
  tools:
    - name: search_financials
      description: "Search financial data for a company"
      parameters:
        company:
          type: string
        metric:
          type: string
    - name: compare_values
      description: "Compare two numerical values"
      parameters:
        value_a:
          type: string
        value_b:
          type: string
  max_plan_steps: 8
  plan_prompt: |
    You are a financial research planner. Decompose the task into
    sequential steps using the available tools. Reference previous
    step outputs as #N where N is the step number.
```

### [INTERMEDIATE] Memory Types

Agent memory research identifies four categories. **These categories are from the agent research literature (e.g., Park et al. 2023, "Generative Agents"). NAT provides memory backends but does not prescribe this specific taxonomy.** Nonetheless, the taxonomy is useful for architectural decisions.

**Short-term memory (working memory):**
The current conversation context. What the user has said, what the agent has done, what tools have returned — all within the current session. In LLM terms, this is the content of the current context window. Short-term memory is lost when the session ends.

**Long-term memory (persistent state):**
Information that persists across sessions. User preferences, previously completed tasks, learned facts. Stored in an external database. Retrieved when relevant.

**Episodic memory:**
Memories of specific past events or interactions. "Last Tuesday, the user asked about Q3 revenue and I found it in the finance database." Useful for avoiding repeated work and maintaining continuity.

**Semantic memory:**
General knowledge and facts. "Company X's fiscal year ends in March." Typically stored in a knowledge base or vector store. Overlaps with RAG (Retrieval-Augmented Generation).

**Mapping to NAT backends:**

| Memory Type | NAT Backend | Rationale |
|-------------|-------------|-----------|
| Short-term | In-memory Object Store | Fast, session-scoped, no persistence needed |
| Long-term | MySQL Object Store | Persistent, queryable, relational |
| Episodic | Redis Object Store | Fast retrieval, TTL support, session-aware |
| Semantic | S3 Object Store + NeMo Retriever | Large-scale storage with vector search |

### [INTERMEDIATE] NAT Memory Systems

NAT provides a pluggable memory architecture with four Object Store backends:

**In-Memory Object Store:**
A Python dictionary in the agent process. Zero setup. Data lost when the process exits. Use for development, testing, and stateless single-session agents.

```python
from nat.memory import InMemoryObjectStore

store = InMemoryObjectStore()
store.put("user_preference", {"language": "en", "detail_level": "high"})
result = store.get("user_preference")
```

**Redis Object Store:**
A Redis-backed store for session state. Fast reads/writes, TTL support for automatic expiration, shared across agent instances. Use for production session state and episodic memory.

```python
from nat.memory import RedisObjectStore

store = RedisObjectStore(
    host="redis.internal.example.com",
    port=6379,
    db=0,
    password="...",
    ttl=3600,  # Keys expire after 1 hour
)
store.put("session:abc123:context", {"last_query": "revenue Q3"})
```

**MySQL Object Store:**
A relational database backend for persistent, queryable storage. Use for long-term memory that must survive restarts, be queried with SQL, or be shared across services.

```python
from nat.memory import MySQLObjectStore

store = MySQLObjectStore(
    host="mysql.internal.example.com",
    port=3306,
    database="agent_memory",
    user="agent_svc",
    password="...",
)
store.put("user:u456:preferences", {"timezone": "PST", "role": "analyst"})
```

**S3 Object Store:**
Object storage for large, archival data. Use for storing conversation logs, large documents, model artifacts, or any data where retrieval latency is secondary to storage cost.

```python
from nat.memory import S3ObjectStore

store = S3ObjectStore(
    bucket="agent-memory-prod",
    region="us-east-1",
    prefix="conversations/",
)
store.put("conv:c789:full_log", large_conversation_data)
```

### [INTERMEDIATE] Automatic Memory Wrapper

NAT provides an **Automatic Memory Wrapper** that transparently manages conversation history. Instead of manually storing and retrieving conversation turns, the wrapper handles it:

```python
from nat.agents import ReActAgent
from nat.memory import AutomaticMemoryWrapper, RedisObjectStore

# Create the base agent
agent = ReActAgent(
    llm=llm,
    tools=[...],
)

# Wrap with automatic memory
memory_store = RedisObjectStore(host="redis.internal.example.com", port=6379)
agent_with_memory = AutomaticMemoryWrapper(
    agent=agent,
    object_store=memory_store,
    session_id="user_session_abc123",
    max_history_turns=20,  # Keep last 20 conversation turns
)

# Conversation history is now automatic
response1 = agent_with_memory.run("What was our Q3 revenue?")
# Memory stores: user query + agent response

response2 = agent_with_memory.run("How does that compare to Q2?")
# Memory retrieves previous turn, agent has context about Q3 revenue
```

The wrapper handles:
- Storing each user query and agent response.
- Retrieving relevant history before each agent invocation.
- Truncating history when it exceeds `max_history_turns`.
- Keying conversations by `session_id` for multi-user support.

**Note:** The Automatic Memory Wrapper manages conversation history (sequential turns). It does not perform semantic retrieval over memory — for that, you would combine it with NeMo Retriever for vector-based memory search.

### [ADVANCED] Context Window vs. External Memory

Modern LLMs have large context windows — 128K tokens or more. When is the context window sufficient, and when do you need external memory?

**Context window is sufficient when:**
- The entire conversation history fits within the window.
- The agent operates in single sessions only (no cross-session continuity).
- All relevant information can be injected into the prompt.
- Latency from longer prompts is acceptable.

**External memory is necessary when:**
- Conversations span multiple sessions (days, weeks).
- The total information exceeds the context window.
- The agent must share state with other agents or services.
- Cost of sending the full history in every prompt is prohibitive (cost grows linearly with context length).
- Retrieval must be selective (fetch only relevant memories, not everything).

**Capacity planning rule of thumb:**
- 1 conversation turn ≈ 200-500 tokens (user query + agent response).
- 128K token context window ≈ 250-600 turns.
- If your typical session is under 50 turns, the context window is likely sufficient for within-session memory.
- For cross-session memory, always use external storage.

### [ADVANCED] Memory Retrieval Strategies

When memory exceeds what fits in the context window, the agent must retrieve selectively. Common strategies:

**Recency-based:** Retrieve the N most recent turns. Simple, works for conversational continuity. Fails when important information was mentioned 100 turns ago.

**Relevance-based (semantic search):** Embed memory entries as vectors, embed the current query, retrieve the most similar entries. Good for finding relevant past interactions. Requires NeMo Retriever or similar vector search infrastructure.

**Importance-based:** Assign importance scores to memories (e.g., user-stated preferences are high importance, casual remarks are low). Retrieve high-importance memories first. Requires a scoring mechanism (can be LLM-based).

**Hybrid:** Combine recency and relevance. Always include the last N turns (recency) plus the top K semantically similar past memories (relevance). This is the most robust approach for production systems.

```
┌──────────────────────────────────────────────────┐
│           HYBRID MEMORY RETRIEVAL                │
│                                                  │
│  Current Query: "Update the Q3 forecast"         │
│                                                  │
│  ┌──────────────┐    ┌───────────────────┐      │
│  │   RECENCY    │    │    RELEVANCE      │      │
│  │  Last 5      │    │  Vector search    │      │
│  │  turns       │    │  for "Q3 forecast"│      │
│  └──────┬───────┘    └────────┬──────────┘      │
│         │                     │                  │
│         v                     v                  │
│  ┌──────────────────────────────────────┐       │
│  │         MERGE & DEDUPLICATE          │       │
│  │  Combine, remove duplicates,         │       │
│  │  truncate to token budget            │       │
│  └──────────────────┬───────────────────┘       │
│                     │                            │
│                     v                            │
│          Inject into agent context               │
└──────────────────────────────────────────────────┘
```

### [ADVANCED] Designing a Memory Architecture

For a production agent system, memory design involves these decisions:

1. **What to store:** Not everything is worth storing. Store user preferences, task outcomes, important facts. Do not store every intermediate reasoning step (unless needed for debugging).

2. **Storage backend per memory type:** Use the mapping from the table above. Multiple backends in the same system is normal — Redis for session state, MySQL for long-term preferences, S3 for conversation logs.

3. **Retention policy:** How long to keep memories. Session state: hours (Redis TTL). User preferences: indefinitely (MySQL). Conversation logs: 90 days (S3 lifecycle policy). Define retention upfront — storage costs accumulate.

4. **Retrieval budget:** How many tokens of memory to inject per request. More memory means better context but higher cost and latency. Typical budget: 20-30% of the context window for memory, 70-80% for the current task.

5. **Consistency requirements:** Can the agent tolerate stale memory? Redis with eventual consistency may be fine for session state. User preferences may require strong consistency (MySQL with read-after-write).

---

## Hands-on NVIDIA Platform Interaction

**Title:** Configure and Test NAT Memory Backends

**NVIDIA service/tool touched:** NVIDIA NeMo Agent Toolkit (NAT) — In-Memory Object Store, Redis Object Store, Automatic Memory Wrapper, Reasoning Agent; NVIDIA NIM hosted endpoint (via build.nvidia.com)

**Required credentials:** `NVIDIA_API_KEY` (obtained from build.nvidia.com)

**Additional infrastructure:** Docker (for running Redis locally via `docker run -p 6379:6379 redis`)

**Execution environment:** Local NAT installation + local Redis container + hosted NIM endpoint (LLM inference runs on NVIDIA-hosted infrastructure; agent orchestration and memory storage run locally)

**Exercise:**

1. Create a NAT workflow with in-memory Object Store. Chat with the agent for 3 turns. Restart the process. Verify memory is lost.
2. Switch to Redis Object Store (run Redis via Docker: `docker run -p 6379:6379 redis`). Chat for 3 turns. Restart the NAT process. Verify memory persists.
3. Enable Automatic Memory Wrapper. Chat for 5 turns. Inspect what the wrapper stored vs. raw conversation history.
4. Run the NAT Reasoning Agent on a multi-step problem (e.g., "Plan a 3-day trip to Tokyo with budget constraints"). Observe the reasoning trace. Compare with the Tool Calling Agent on the same problem.

**Expected artifact/output:** (a) Screenshot or log showing memory persistence across restarts with Redis. (b) Reasoning trace from the Reasoning Agent showing multi-step deliberation.

**Verification step:** Memory survives process restart when using Redis (re-launching the NAT process and issuing a follow-up query returns context from the previous session). Reasoning Agent produces a structured plan with visible deliberation steps that differ meaningfully from the Tool Calling Agent's single-pass output.

**Interaction classification:** Local NVIDIA tooling interaction (NAT, Redis) + hosted NVIDIA API interaction (NIM endpoint)

---

## F. Terminology Box

| Term | Definition |
|------|-----------|
| **Chain-of-Thought (CoT)** | A prompting strategy that elicits step-by-step reasoning before a final answer. |
| **Tree-of-Thought (ToT)** | A reasoning strategy that explores multiple reasoning branches and selects the best path. |
| **Self-reflection** | A reasoning pattern where the model critiques and revises its own output. |
| **Self-consistency** | Generating multiple answers and selecting the most common one (majority voting). |
| **Test Time Compute (TTC)** | Spending additional computation at inference time (e.g., generating multiple candidates) to improve output quality. |
| **Plan-then-execute** | A planning strategy where the full plan is generated before any execution occurs. |
| **Interleaved planning** | A planning strategy where planning and execution alternate, allowing the plan to adapt. |
| **Short-term memory** | Information retained within the current session/conversation context. |
| **Long-term memory** | Information that persists across sessions in external storage. |
| **Episodic memory** | Memories of specific past events or interactions. |
| **Semantic memory** | General knowledge and facts, typically stored in a knowledge base. |
| **Object Store** | NAT's abstraction for memory backends (in-memory, Redis, MySQL, S3). |
| **Automatic Memory Wrapper** | NAT component that transparently manages conversation history storage and retrieval. |
| **Retrieval budget** | The portion of the context window allocated to retrieved memories. |

---

## G. Common Misconceptions

1. **"Chain-of-Thought always improves performance."** Incorrect. CoT helps on multi-step reasoning tasks but can actually hurt performance on simple tasks where direct answers are more reliable. It also increases token usage for every request.

2. **"Larger context windows eliminate the need for external memory."** Incorrect. Even with a 1M-token window, cross-session persistence, multi-agent state sharing, and cost management still require external storage. Sending 500K tokens per request is expensive, even if the window supports it.

3. **"NAT's Test Time Compute uses a specific published algorithm."** Not confirmed. NAT documents TTC at the feature level (multi-candidate generation, planning, selection) but does not publish the internal scoring or selection algorithm. Do not assume a specific algorithm in exam answers — describe it at the documented level.

4. **"Memory types (episodic, semantic, etc.) are NAT configuration categories."** Incorrect. The four-type memory taxonomy comes from cognitive science and agent research literature. NAT provides Object Store backends that can implement any of these types, but NAT itself does not label its stores as "episodic" or "semantic." The mapping is an architectural design decision you make.

5. **"Redis is always better than in-memory for development."** Incorrect. In-memory is simpler, requires no infrastructure, and is sufficient for development and testing. Redis adds operational complexity (running a Redis server). Use in-memory for development; switch to Redis for production.

6. **"The Automatic Memory Wrapper handles semantic search."** Incorrect. The Automatic Memory Wrapper manages sequential conversation history (store turns, retrieve recent turns). It does not perform semantic similarity search over past memories. For semantic retrieval, combine it with NeMo Retriever.

---

## H. Failure Modes / Anti-Patterns

1. **Unbounded memory injection.** Injecting the entire conversation history into the context on every request. After 100 turns, the prompt is enormous, latency is high, and cost is through the roof. Always cap the number of turns or total tokens injected.

2. **No memory eviction policy.** Storing everything in Redis without TTL. Over time, the Redis instance fills up. Define TTLs for all session-scoped data and lifecycle policies for archival storage.

3. **Using long-term memory for session state.** Storing ephemeral session data in MySQL. This creates unnecessary write load and cleanup complexity. Use Redis with TTL for session state; use MySQL for data that must persist indefinitely.

4. **Reasoning without termination.** Using the Reasoning Agent with high `num_candidates` on every request, including trivial ones. This wastes compute. Use Test Time Compute selectively — for high-stakes or complex queries only. Route simple queries to a lightweight agent.

5. **Plan-then-execute for unpredictable tasks.** Using ReWOO when the plan heavily depends on intermediate results. The agent generates an optimistic plan, an early step returns unexpected data, and the remaining steps operate on invalid assumptions. Use ReAct for exploratory tasks.

6. **Memory shared without access control.** Multiple agents reading from and writing to the same memory store without namespace isolation. Agent A overwrites Agent B's data. Use key prefixes or separate stores per agent.

7. **Ignoring memory retrieval latency.** Adding a vector similarity search over millions of memory entries without considering the latency impact. Memory retrieval adds to the critical path of every request. Benchmark retrieval latency and set budgets.

---

## I. Hands-On Lab

**Lab 3: Reasoning, Planning, and Memory in Practice**

- Configure a NAT Reasoning Agent and compare its output quality to a standard ReAct Agent on 5 complex analysis questions. Measure: accuracy, latency, token cost.
- Implement a ReWOO planning agent for a multi-step data analysis task. Inspect the generated plan. Introduce a scenario where the plan fails (unexpected data) and observe the failure mode.
- Set up NAT memory with three backends: in-memory (development), Redis (session state), MySQL (persistent preferences). Write and read data from each.
- Implement the Automatic Memory Wrapper on a ReAct Agent. Verify that the agent maintains conversational context across multiple turns.

Full lab specifications are in a separate file.

---

## J. Stretch Lab

**Stretch: Hybrid Memory System**

Build an agent with a hybrid memory architecture:
- Redis for the last 10 conversation turns (recency).
- A vector store (using NeMo Retriever or a local FAISS index) for semantic search over all past conversations (relevance).
- MySQL for user preferences (persistent).

Implement a retrieval function that combines recency-based and relevance-based retrieval, deduplicates, and truncates to a token budget. Test with a 50-turn conversation where the user references information from turn 5 at turn 45. Verify the agent recalls the information.

---

## K. Review Quiz

**1.** Which reasoning strategy generates multiple reasoning branches and selects the best one?
a) Chain-of-Thought b) Tree-of-Thought c) Self-reflection d) Self-consistency
**Answer:** b) Tree-of-Thought explores multiple paths and selects the most promising.

**2.** What does NAT's Test Time Compute involve at the documented feature level?
a) Fine-tuning the model at inference time b) Multi-candidate generation, planning, and selection c) Increasing the model's parameter count d) Caching previous responses
**Answer:** b) TTC generates multiple candidates, plans the reasoning, and selects the best output.

**3.** Which NAT Object Store backend is most appropriate for production session state?
a) In-memory b) MySQL c) Redis d) S3
**Answer:** c) Redis provides fast reads/writes with TTL support for session-scoped data.

**4.** The four-type memory taxonomy (short-term, long-term, episodic, semantic) originates from:
a) NAT documentation b) Agent research literature and cognitive science c) The NVIDIA Blueprint specifications d) The OpenAI API documentation
**Answer:** b) This taxonomy is from research literature; NAT provides backends but does not prescribe this taxonomy.

**5.** What is the primary advantage of plan-then-execute (ReWOO) over interleaved planning (ReAct)?
a) Better error recovery b) Fewer LLM calls c) More adaptive plans d) Higher accuracy on all tasks
**Answer:** b) ReWOO typically requires only 2 LLM calls (plan + synthesize) vs. N calls for ReAct.

**6.** When does the context window become insufficient, requiring external memory?
a) When the model has more than 32K tokens b) When conversations span multiple sessions c) When using NVIDIA NIM d) When the agent has more than 5 tools
**Answer:** b) Cross-session persistence always requires external memory, regardless of context window size.

**7.** What does the NAT Automatic Memory Wrapper manage?
a) Semantic search over past interactions b) Sequential conversation history c) Model weights d) Tool configurations
**Answer:** b) It transparently stores and retrieves sequential conversation turns.

**8.** A retrieval budget of 30% of the context window means:
a) 30% of tokens are for the system prompt b) 30% of tokens are allocated to retrieved memories c) 30% of tokens are for tool outputs d) The agent uses 30% fewer tokens overall
**Answer:** b) The retrieval budget defines how much of the context window is reserved for injected memories.

**9.** Which is an anti-pattern when using the Reasoning Agent?
a) Using it for complex analysis b) Generating multiple candidates c) Applying it to every request including trivial ones d) Combining it with tools
**Answer:** c) Using Test Time Compute on trivial requests wastes compute without improving quality.

**10.** Hybrid memory retrieval combines:
a) Redis and MySQL b) Recency-based and relevance-based retrieval c) CoT and ToT d) Short-term and long-term models
**Answer:** b) Hybrid retrieval merges recent turns (recency) with semantically similar past memories (relevance).

---

## L. Mini Project

**Project: Memory-Augmented Research Agent**

Build an agent that:
1. Uses ReAct for multi-step research queries.
2. Stores all conversation turns in Redis (via Automatic Memory Wrapper).
3. Stores user-stated preferences in MySQL (e.g., "I prefer detailed answers," "My fiscal year starts in April").
4. On each new query, retrieves: last 5 conversation turns + any matching user preferences.
5. Demonstrates that preferences set in session 1 affect behavior in session 2.

Document: memory architecture diagram, backend choices with justification, token budget allocation, and observed behavior across sessions.

---

## M. How This May Appear on the Exam

1. **Reasoning strategy selection:** "A task requires the agent to evaluate multiple possible approaches to a coding problem and select the best one. Which reasoning strategy is most appropriate?" (Tree-of-Thought or Test Time Compute with multiple candidates.)

2. **Memory backend matching:** "An agent must persist user preferences across sessions with strong consistency. Which NAT Object Store backend is most appropriate?" (MySQL.)

3. **Planning pattern tradeoff:** "An analyst agent must research a topic where the required data sources are unknown upfront. Should it use ReWOO or ReAct?" (ReAct, because the plan depends on intermediate discoveries.)

4. **Context window vs. external memory:** "A support agent handles 10-turn conversations within single sessions and has a 128K-token context window. Does it need external memory?" (No, for within-session state. Yes, if cross-session continuity is required.)

5. **Anti-pattern identification:** "An agent stores all data in Redis without TTL. What problem will this cause?" (Redis memory fills up over time; data is never evicted.)

---

## N. Checklist for Mastery

Before moving to Module 4, confirm you can:

- [ ] Explain CoT, ToT, self-reflection, and self-consistency in one sentence each, without notes.
- [ ] Describe what NAT's Test Time Compute does at the documented feature level, and explicitly state what is not publicly documented.
- [ ] Configure a NAT Reasoning Agent in Python and explain when to use it vs. a standard ReAct Agent.
- [ ] Explain the tradeoff between plan-then-execute (ReWOO) and interleaved planning (ReAct) with specific scenarios for each.
- [ ] Name the four memory types from research literature and acknowledge that NAT does not prescribe this taxonomy.
- [ ] Configure each NAT Object Store backend (in-memory, Redis, MySQL, S3) and state the use case for each.
- [ ] Implement the Automatic Memory Wrapper and explain what it does and does not handle (sequential history yes, semantic search no).
- [ ] Calculate when the context window is sufficient vs. when external memory is needed, given conversation length and token estimates.
- [ ] Design a hybrid memory retrieval strategy combining recency and relevance.
- [ ] Identify at least five memory-related anti-patterns and explain how to avoid each.
- [ ] Set appropriate TTL values for session state in Redis and justify the choice.
- [ ] Draw a memory architecture diagram for a multi-session agent system, including backend choices and data flow.
