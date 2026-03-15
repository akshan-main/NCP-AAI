# Module 2: Agent Architecture Patterns

**Primary exam domains:** Architecture (15%)
**NVIDIA tools:** NAT agent types (ReAct, Reasoning, ReWOO, Router, Sequential Executor, Tool Calling, Responses API Agent), NVIDIA Blueprint reference architectures

---

## A. Module Title

**Agent Architecture Patterns**

---

## B. Why This Module Matters in Real Systems

Choosing the wrong agent architecture is one of the most expensive mistakes in agentic AI development. A ReAct agent where a simple tool-calling agent would suffice burns 3-5x more tokens per request and adds seconds of latency. A sequential executor where a router is needed produces wrong answers whenever user intent does not match the hardcoded sequence. These are not hypothetical failures — they are the most common source of rework in production agent systems. This module gives you the pattern vocabulary and decision framework to get the architecture right the first time.

The NVIDIA NeMo Agent Toolkit (NAT) implements seven distinct agent patterns, each optimized for different task profiles. Understanding these patterns is not just about knowing the API — it is about understanding the tradeoffs behind each one. When a system must minimize latency, you reach for a Tool Calling Agent. When a system must handle complex multi-step research, you reach for ReAct or ReWOO. When a system must route between specialized capabilities, you reach for a Router. The exam expects you to match requirements to patterns and justify your choice.

Beyond individual agent patterns, NVIDIA publishes Blueprint reference architectures that show how to compose these patterns into complete systems. This module covers the major Blueprints — AI-Q for research, Multi-Agent Warehouse for coordination, and Data Flywheel for continuous improvement — so you understand how production systems combine multiple patterns. If Module 1 was about what makes a system agentic, this module is about how to structure that agency.

---

## C. Learning Objectives

By the end of this module, you will be able to:

1. Describe seven NAT agent patterns (ReAct, Reasoning, ReWOO, Router, Sequential Executor, Tool Calling, Responses API Agent) and state the use case for each.
2. Given a set of requirements (latency, accuracy, cost, complexity), select the appropriate agent pattern and justify the choice.
3. Write YAML configuration for at least three NAT agent types.
4. Draw architecture diagrams for the ReAct loop, Router dispatch pattern, and Sequential pipeline.
5. Explain the tradeoffs between plan-first (ReWOO) and interleaved (ReAct) execution strategies.
6. Describe three NVIDIA Blueprint reference architectures and identify which agent patterns each uses.
7. Identify when a multi-agent system is necessary vs. when a single agent with multiple tools suffices.
8. Estimate token cost and latency characteristics for each agent pattern given a workload profile.

---

## D. Required Concepts

Before starting this module, you should be comfortable with:

- **Module 1 content**: Agent loop, autonomy spectrum, NVIDIA stack components, NIM API calls, NAT installation.
- **YAML syntax**: Indentation, lists, dictionaries, anchors/aliases.
- **Design patterns (general)**: The concept of a design pattern as a reusable solution to a recurring problem.
- **Latency analysis**: Ability to reason about sequential vs. parallel execution and their effect on total response time.
- **Token economics**: Understanding that each LLM call consumes tokens, which cost money and take time proportional to input + output length.

---

## E. Core Lesson Content

### [BEGINNER] The Seven NAT Agent Patterns

NAT provides seven built-in agent types. Each implements a different execution strategy. Here is the taxonomy:

| Agent Type | Strategy | When to Use |
|------------|----------|-------------|
| **Tool Calling Agent** | Single-turn: LLM decides which tool(s) to call, calls them, returns result. | Simple tasks, low latency requirements, one-step tool use. |
| **ReAct Agent** | Multi-turn loop: Reason about what to do → Act (call tool) → Observe result → Repeat. | Multi-step tasks where each step depends on previous results. |
| **Reasoning Agent** | Extended thinking: Uses test-time compute to generate multiple reasoning paths before acting. | Tasks requiring deep analysis, high accuracy requirements. |
| **ReWOO Agent** | Plan-then-execute: Generate complete plan first, then execute all steps. | Tasks where planning is separable from execution, batch operations. |
| **Router Agent** | Dispatch: Classify the request and route to the appropriate sub-agent or handler. | Multi-domain systems, intent-based routing. |
| **Sequential Executor** | Linear chain: Execute a predefined sequence of agents/steps in order. | Deterministic multi-step workflows, document processing pipelines. |
| **Responses API Agent** | OpenAI-compatible: Exposes an agent via the OpenAI Responses API format. | Interoperability with OpenAI-native tooling and clients. |

### [BEGINNER] Tool Calling Agent — The Simplest Pattern

The Tool Calling Agent is the baseline. The LLM receives the user query plus a list of available tools (as function schemas). It decides whether to call a tool, which tool to call, and with what arguments. The tool executes, and the result is returned to the user.

```
User Query ──> LLM (with tool schemas) ──> Tool Call ──> Result ──> User
```

There is no loop. One LLM call, zero or one tool calls. This is appropriate when:
- The task can be completed in a single step.
- Latency must be minimal (one LLM round-trip).
- The mapping from query to tool is straightforward.

**NAT YAML configuration:**

```yaml
# tool_calling_agent.yaml
agent:
  type: tool_calling
  llm:
    model: meta/llama-3.1-70b-instruct
    endpoint: ${NIM_ENDPOINT}
    temperature: 0.1
  tools:
    - name: get_weather
      description: "Get current weather for a city"
      parameters:
        city:
          type: string
          description: "City name"
  max_tokens: 512
```

### [INTERMEDIATE] ReAct Agent — The Workhorse

ReAct (Reasoning + Acting) is the most commonly used multi-step agent pattern. The agent alternates between reasoning (thinking about what to do) and acting (calling a tool). After each action, it observes the result and decides whether to continue or stop.

```
┌──────────────────────────────────────────────────────────┐
│                    ReAct LOOP                             │
│                                                          │
│  User Query                                              │
│      │                                                   │
│      v                                                   │
│  ┌────────┐    ┌────────┐    ┌─────────┐                │
│  │THOUGHT │───>│ ACTION │───>│OBSERVA- │──┐             │
│  │(reason)│    │(tool   │    │TION     │  │             │
│  │        │    │ call)  │    │(result) │  │             │
│  └────────┘    └────────┘    └─────────┘  │             │
│      ^                                     │             │
│      │         Loop until done             │             │
│      └─────────────────────────────────────┘             │
│      │                                                   │
│      v                                                   │
│  Final Answer                                            │
└──────────────────────────────────────────────────────────┘
```

**Key tradeoffs:**
- **Latency**: Each loop iteration requires an LLM call. A 3-step ReAct loop takes 3x the latency of a single tool call.
- **Token cost**: Each iteration sends the full conversation history (including all previous thoughts and observations) to the LLM. Token usage grows quadratically with steps if not managed.
- **Accuracy**: Generally higher than single-step for complex tasks because the agent can correct course.
- **Failure recovery**: If a tool call returns an error, the agent can reason about the failure and try an alternative.

**NAT YAML configuration:**

```yaml
# react_agent.yaml
agent:
  type: react
  llm:
    model: meta/llama-3.1-70b-instruct
    endpoint: ${NIM_ENDPOINT}
    temperature: 0.2
  tools:
    - name: search_knowledge_base
      description: "Search internal documentation"
      parameters:
        query:
          type: string
    - name: get_employee_info
      description: "Look up employee details by ID"
      parameters:
        employee_id:
          type: string
  max_iterations: 5
  max_tokens: 1024
```

The `max_iterations` parameter is critical in production. Without it, an agent that cannot solve a problem will loop indefinitely, burning tokens.

### [INTERMEDIATE] ReWOO Agent — Plan First, Execute Later

ReWOO (Reasoning Without Observation) separates planning from execution. The agent first generates a complete plan — a sequence of steps with their expected inputs and outputs — then executes all steps without re-reasoning between them.

```
┌────────────────────────────────────────────────────┐
│                  ReWOO PATTERN                      │
│                                                    │
│  User Query                                        │
│      │                                             │
│      v                                             │
│  ┌──────────┐                                      │
│  │  PLANNER │──> Step 1: search(query)             │
│  │  (single │    Step 2: extract(#1.result, field) │
│  │   LLM    │    Step 3: lookup(#2.result)         │
│  │   call)  │    Step 4: format(#3.result)         │
│  └──────────┘                                      │
│      │                                             │
│      v                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────┐│
│  │Execute 1 │─>│Execute 2 │─>│Execute 3 │─>│ 4  ││
│  └──────────┘  └──────────┘  └──────────┘  └────┘│
│      │                                             │
│      v                                             │
│  Final Answer                                      │
└────────────────────────────────────────────────────┘
```

**When to use ReWOO over ReAct:**
- When the plan can be fully determined upfront (no conditional branching needed).
- When you want to minimize LLM calls (ReWOO uses 1-2 LLM calls; ReAct uses N calls for N steps).
- When tool execution is expensive and you want to batch it.

**When NOT to use ReWOO:**
- When later steps depend on dynamic conditions discovered during execution.
- When tool results might require the agent to change its plan.
- When error recovery is critical (ReWOO does not re-reason after a failed step).

**NAT YAML configuration:**

```yaml
# rewoo_agent.yaml
agent:
  type: rewoo
  llm:
    model: meta/llama-3.1-70b-instruct
    endpoint: ${NIM_ENDPOINT}
    temperature: 0.1
  tools:
    - name: search
      description: "Search a document store"
      parameters:
        query:
          type: string
    - name: summarize
      description: "Summarize a text passage"
      parameters:
        text:
          type: string
  max_plan_steps: 6
```

### [INTERMEDIATE] Router Agent — Dynamic Dispatch

The Router Agent classifies the incoming request and dispatches it to the appropriate sub-agent or handler. It does not solve the task itself — it decides who should solve it.

```
┌─────────────────────────────────────────────────────┐
│                  ROUTER PATTERN                      │
│                                                     │
│  User Query                                         │
│      │                                              │
│      v                                              │
│  ┌──────────┐                                       │
│  │  ROUTER  │                                       │
│  │  (LLM    │                                       │
│  │  classif)│                                       │
│  └──┬───┬───┘                                       │
│     │   │   └──────────────┐                        │
│     v   v                  v                        │
│  ┌─────┐  ┌──────────┐  ┌──────────┐              │
│  │ RAG │  │ Database │  │ Analyst  │              │
│  │Agent│  │  Agent   │  │  Agent   │              │
│  └─────┘  └──────────┘  └──────────┘              │
│     │         │               │                     │
│     └─────────┴───────────────┘                     │
│                │                                    │
│                v                                    │
│          Final Answer                               │
└─────────────────────────────────────────────────────┘
```

This pattern is essential for multi-domain systems. A customer support system might route billing questions to a billing agent, technical questions to a troubleshooting agent, and general questions to a FAQ agent.

### [INTERMEDIATE] Sequential Executor and Remaining Patterns

**Sequential Executor** runs a fixed chain of agents or processing steps in order. Unlike ReAct, there is no decision-making between steps — the sequence is predetermined. Use this for document processing pipelines (extract → validate → enrich → store) or multi-stage analysis.

**Reasoning Agent** uses test-time compute to generate higher-quality responses. It may generate multiple candidate responses, evaluate them, and select the best one. This is covered in depth in Module 3.

**Responses API Agent** wraps a NAT agent with an OpenAI Responses API-compatible interface. This is a deployment concern rather than an architectural one — it lets you serve a NAT agent to clients that expect the OpenAI API format.

### [ADVANCED] Architecture Decision Framework

Given a new project, use this decision process:

```
1. Can the task be completed in one tool call?
   YES → Tool Calling Agent
   NO  → Continue

2. Can the full plan be determined before execution?
   YES → Is error recovery during execution needed?
         NO  → ReWOO Agent
         YES → ReAct Agent
   NO  → Continue

3. Does the task require routing between domains?
   YES → Router Agent (with sub-agents)
   NO  → Continue

4. Is the task a fixed multi-step pipeline?
   YES → Sequential Executor
   NO  → Continue

5. Does the task require extended reasoning / deep analysis?
   YES → Reasoning Agent
   NO  → ReAct Agent (default multi-step choice)
```

**Token cost estimates** (inferred from typical usage patterns — not from NVIDIA benchmarks):

| Pattern | LLM Calls per Request | Relative Token Cost |
|---------|----------------------|-------------------|
| Tool Calling | 1 | 1x (baseline) |
| ReAct (3 steps) | 3-4 | 4-6x |
| ReWOO | 2 | 2-3x |
| Router + sub-agent | 2+ | 2-4x |
| Sequential (3 steps) | 3 | 3-4x |
| Reasoning | 1-3 (with test-time compute) | 3-8x |

**Note:** These cost estimates are inferred from typical agent behavior patterns, not from NVIDIA-published benchmarks. Actual costs depend on prompt length, model size, and tool output sizes.

### [ADVANCED] NVIDIA Blueprint Reference Architectures

NVIDIA publishes Blueprint reference architectures that demonstrate how to compose agent patterns into complete production systems.

**AI-Q (Research Assistant Blueprint):**
Combines RAG with agentic reasoning. A ReAct agent uses NeMo Retriever to search a knowledge base, reasons about the results, and synthesizes answers with citations. Uses NeMo Guardrails to prevent hallucinated citations. This is the go-to reference for enterprise knowledge systems.

**Multi-Agent Warehouse Blueprint:**
A Router agent dispatches tasks to specialized sub-agents, each handling a different function (inventory lookup, order processing, shipping coordination). Sub-agents are ReAct agents with domain-specific tools. Demonstrates inter-agent communication and result aggregation. This is the reference for multi-domain enterprise systems.

**Data Flywheel Blueprint:**
Not an agent architecture per se, but a system-level pattern. User interactions with agents are logged, evaluated by NeMo Evaluator, and used to improve the system via NeMo Curator (data preparation) and NeMo Customizer (fine-tuning). This demonstrates continuous improvement for agent systems.

**Note:** Blueprint details are based on NVIDIA's published documentation. Internal implementation specifics may differ from what is publicly described. Always consult the latest Blueprint documentation for current architecture details.

### [ADVANCED] Multi-Agent vs. Single Agent with Multiple Tools

A common design question: should you build one agent with many tools, or multiple specialized agents?

**Single agent, many tools:**
- Simpler architecture. One LLM context, one configuration.
- Works well when tools are in the same domain.
- Breaks down when the tool count exceeds ~15-20 (the LLM struggles to select the right tool from a large set).
- Lower latency (no inter-agent routing).

**Multiple specialized agents:**
- Each agent has a focused tool set and domain-specific system prompt.
- Better accuracy for multi-domain tasks.
- Higher latency (routing + sub-agent execution).
- More complex to deploy and monitor.
- Required when different domains need different LLMs or different safety policies.

Rule of thumb: start with a single agent. Split into multi-agent only when accuracy degrades due to tool-set size or domain confusion.

---

## Hands-on NVIDIA Platform Interaction

**Title:** Build and Compare NAT Agent Types

**NVIDIA service/tool touched:** NVIDIA NeMo Agent Toolkit (NAT) — ReAct Agent, Tool Calling Agent, Sequential Executor agent types; NVIDIA NIM hosted endpoint (via build.nvidia.com)

**Required credentials:** `NVIDIA_API_KEY` (obtained from build.nvidia.com)

**Execution environment:** Local NAT installation + hosted NIM endpoint (LLM inference runs on NVIDIA-hosted infrastructure; agent orchestration runs locally)

**Exercise:**

1. Take the ReAct Agent YAML config shown in the lesson and actually run it against an NVIDIA-hosted NIM endpoint. Use a simple task like "What is the weather in San Francisco?" with a mock weather tool.
2. Modify the YAML to switch to a Tool Calling Agent. Run the same task. Note the difference in execution trace.
3. Modify to a Sequential Executor for a 2-step task. Run and observe.

**Expected artifact/output:** A comparison table showing agent type, number of LLM calls, token usage, and output quality for the same task across 3 agent types.

**Verification step:** You have 3 execution traces saved and can explain why each agent type produced a different trace. Specifically: the Tool Calling Agent should show a single LLM call, the ReAct Agent should show a multi-step thought-action-observation loop, and the Sequential Executor should show a fixed sequence of steps executed in order.

**Interaction classification:** Local NVIDIA tooling interaction (NAT) + hosted NVIDIA API interaction (NIM endpoint)

---

## F. Terminology Box

| Term | Definition |
|------|-----------|
| **ReAct** | Reasoning + Acting. An agent pattern that alternates between reasoning and tool use in a loop. |
| **ReWOO** | Reasoning Without Observation. A plan-first agent pattern that generates a complete plan before executing any steps. |
| **Router Agent** | An agent that classifies requests and dispatches them to specialized sub-agents. |
| **Sequential Executor** | An agent pattern that executes a fixed chain of steps in order without inter-step decision-making. |
| **Test-time compute** | Additional computation at inference time (e.g., generating multiple candidates, selecting the best) to improve response quality. |
| **Blueprint** | An NVIDIA reference architecture that demonstrates how to compose multiple NVIDIA components into a production system. |
| **Tool schema** | A structured description of a tool's name, description, and parameters, provided to the LLM so it can generate valid tool calls. |
| **Max iterations** | A configuration parameter that limits the number of loop iterations in an agent, preventing runaway execution. |
| **Token cost** | The number of tokens consumed by an agent workflow, which determines both monetary cost and latency. |
| **Inter-agent routing** | The process of one agent delegating work to another agent based on task classification. |

---

## G. Common Misconceptions

1. **"ReAct is always the best choice for multi-step tasks."** Incorrect. ReWOO is more token-efficient when the plan can be determined upfront. Sequential Executor is more predictable when the steps are fixed. ReAct's advantage is adaptability, not universal superiority.

2. **"Router agents add intelligence."** Incorrect. A Router agent adds indirection, not intelligence. The router classifies and dispatches — the intelligence is in the sub-agents. A badly designed router that misclassifies requests makes the entire system worse.

3. **"More agents means better performance."** Incorrect. Each additional agent adds latency, cost, and complexity. Multi-agent systems are justified only when domain separation provides measurable accuracy gains.

4. **"YAML configuration replaces code."** Partially incorrect. NAT YAML configuration defines agent structure and parameters, but custom tool implementations, preprocessing logic, and integration code are still written in Python.

5. **"The Responses API Agent is a different architecture."** Incorrect. It is a deployment wrapper, not a distinct execution strategy. It wraps another NAT agent type with an OpenAI-compatible API interface.

6. **"Blueprints are production-ready templates."** Partially incorrect. Blueprints are reference architectures — they demonstrate patterns and best practices. Production deployment requires customization for your specific infrastructure, data, security requirements, and scale.

---

## H. Failure Modes / Anti-Patterns

1. **Overloaded tool sets.** Giving a single agent 30+ tools. The LLM cannot reliably select from a large tool set. Accuracy drops sharply beyond ~15-20 tools. Solution: use a Router to split into domain-specific agents.

2. **ReAct without max iterations.** Deploying a ReAct agent with no iteration limit. A confused agent will loop endlessly, generating "Thought: I should try again" until tokens are exhausted. Always set `max_iterations`.

3. **ReWOO for dynamic tasks.** Using a plan-first approach for tasks where the plan depends on intermediate results. The agent generates a plan based on assumptions that turn out to be wrong, and it cannot adapt.

4. **Router with overlapping domains.** Defining sub-agent domains that overlap (e.g., a "billing agent" and a "payments agent" where half of queries could go to either). This causes inconsistent routing. Sub-agent domains must be clearly delineated.

5. **Sequential Executor for conditional workflows.** Using Sequential Executor when some steps should be skipped based on conditions. Sequential Executor does not support branching — use a Router or ReAct agent instead.

6. **Ignoring token growth in ReAct.** Each ReAct iteration appends the full thought-action-observation to the context. After 5-7 iterations, the context can exceed the model's window. Solution: summarize intermediate steps or use sliding-window context management.

---

## I. Hands-On Lab

**Lab 2: Building and Comparing Agent Architectures**

- Implement three agents for the same task (e.g., "find information about a topic, verify it, and summarize"):
  1. A Tool Calling Agent (single-step, observe what it misses).
  2. A ReAct Agent (multi-step, observe the reasoning loop).
  3. A ReWOO Agent (plan-first, observe the plan quality).
- Compare all three on: accuracy (correct answers out of 10 test queries), latency (average response time), token usage (total tokens per request).
- Configure a Router Agent that dispatches between a Q&A sub-agent and a summarization sub-agent.

Full lab specifications are in a separate file.

---

## J. Stretch Lab

**Stretch: Multi-Agent System Design**

Design and implement a multi-agent customer support system with:
- A Router agent that classifies requests into billing, technical, and general.
- Three specialized ReAct sub-agents, each with domain-specific tools.
- A fallback path: if the Router is uncertain, escalate to a human.
- Measure routing accuracy on 20 test queries. Identify misrouted queries and explain why the Router failed.

---

## K. Review Quiz

**1.** Which NAT agent type generates a complete plan before executing any steps?
a) ReAct b) ReWOO c) Router d) Tool Calling
**Answer:** b) ReWOO generates the full plan first, then executes.

**2.** A task requires the agent to adapt its approach based on intermediate results. Which pattern is most appropriate?
a) Sequential Executor b) ReWOO c) ReAct d) Tool Calling
**Answer:** c) ReAct's loop allows the agent to change course based on observations.

**3.** What is the primary risk of giving a single agent 30+ tools?
a) The agent becomes too slow. b) The LLM cannot reliably select the correct tool. c) NAT does not support more than 30 tools. d) Memory usage is too high.
**Answer:** b) LLMs struggle with tool selection accuracy as the tool set grows.

**4.** In a Router agent pattern, where does the actual task-solving intelligence reside?
a) In the Router itself b) In the sub-agents c) In the LLM d) In the YAML configuration
**Answer:** b) The Router classifies and dispatches; sub-agents solve the tasks.

**5.** What is the key advantage of ReWOO over ReAct in terms of resource usage?
a) ReWOO uses less memory. b) ReWOO requires fewer LLM calls. c) ReWOO is faster at tool execution. d) ReWOO supports more tools.
**Answer:** b) ReWOO typically needs 1-2 LLM calls vs. N calls for N-step ReAct.

**6.** The NVIDIA AI-Q Blueprint demonstrates:
a) Multi-agent warehouse operations b) RAG combined with agentic reasoning c) Model fine-tuning pipelines d) Real-time streaming agents
**Answer:** b) AI-Q combines NeMo Retriever (RAG) with agentic reasoning.

**7.** When should you choose Sequential Executor over ReAct?
a) When the task requires dynamic decision-making. b) When the steps are fixed and deterministic. c) When accuracy is paramount. d) When the task has many possible branches.
**Answer:** b) Sequential Executor is for predetermined, non-branching step sequences.

**8.** What does the `max_iterations` parameter in a ReAct agent prevent?
a) Memory overflow b) Unbounded looping and token consumption c) Tool call failures d) Concurrent execution
**Answer:** b) It caps the number of reason-act-observe cycles to prevent runaway execution.

**9.** The NAT Responses API Agent is best described as:
a) A unique architecture pattern b) A deployment wrapper providing OpenAI API compatibility c) The fastest agent type d) A replacement for ReAct
**Answer:** b) It is a deployment interface, not a distinct execution strategy.

**10.** Which decision is MOST important to get right early in agent system design?
a) The specific LLM model to use b) The programming language c) The agent architecture pattern d) The cloud provider
**Answer:** c) The architecture pattern determines the system's behavior, cost, and performance characteristics. Models and infrastructure can be swapped later.

---

## L. Mini Project

**Project: Architecture Pattern Selector Tool**

Build a command-line tool or web form that:
1. Accepts requirements as input: latency budget (ms), accuracy priority (low/medium/high), cost sensitivity (low/medium/high), task complexity (single-step/multi-step/multi-domain), plan stability (static/dynamic).
2. Outputs a recommended NAT agent type with justification.
3. Generates a skeleton YAML configuration for the recommended agent type.
4. Warns about common anti-patterns for the selected pattern.

Test with at least 7 different requirement profiles and verify the recommendations make sense.

---

## M. How This May Appear on the Exam

1. **Pattern selection:** "A system must answer research questions by searching multiple databases, evaluating source credibility, and synthesizing a report. The plan can be determined upfront. Which NAT agent type is most appropriate?" (ReWOO, because the plan is known upfront.)

2. **Tradeoff analysis:** "What is the primary disadvantage of using a ReAct agent compared to a Tool Calling agent for a single-step lookup task?" (Unnecessary latency and token cost from the reasoning loop.)

3. **Blueprint identification:** "Which NVIDIA Blueprint demonstrates continuous improvement of an agentic system using evaluation and fine-tuning?" (Data Flywheel.)

4. **Anti-pattern recognition:** "An agent with 40 tools frequently calls the wrong tool. What architectural change would most likely fix this?" (Router agent that dispatches to specialized sub-agents with smaller tool sets.)

5. **Configuration knowledge:** "In a NAT ReAct agent YAML configuration, which parameter prevents the agent from looping indefinitely?" (`max_iterations`.)

---

## N. Checklist for Mastery

Before moving to Module 3, confirm you can:

- [ ] Name all seven NAT agent types and state the core execution strategy of each.
- [ ] Draw the ReAct loop (Thought → Action → Observation) from memory.
- [ ] Draw the Router dispatch pattern from memory.
- [ ] Explain when to use ReWOO instead of ReAct, and vice versa.
- [ ] Write a NAT YAML configuration for a Tool Calling Agent, a ReAct Agent, and a ReWOO Agent without consulting documentation.
- [ ] Estimate relative token cost for each agent pattern given a task description.
- [ ] Describe the AI-Q, Multi-Agent Warehouse, and Data Flywheel Blueprints.
- [ ] Apply the architecture decision framework to a novel requirements set and justify your choice.
- [ ] Explain why overloaded tool sets degrade agent accuracy.
- [ ] Identify the anti-pattern when someone uses Sequential Executor for conditional workflows.
- [ ] Explain the difference between single-agent-many-tools and multi-agent architectures.
- [ ] Describe what the Responses API Agent provides and when to use it.
