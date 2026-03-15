# Module 7: Multi-Agent Systems

**Primary exam domains:** Architecture (15%), Development (15%)
**NVIDIA tools:** NAT Router Agent, NAT A2A Server, NAT A2A protocol support, Multi-Agent Intelligent Warehouse Blueprint, Retail Agentic Commerce Blueprint (ACP/UCP protocols)

---

## A. Module Title

**Multi-Agent Systems: Coordination, Communication, and Real-World Architectures**

---

## B. Why This Module Matters in Real Systems

Single-agent architectures hit a ceiling fast. The moment your system needs to simultaneously query a database, call an external API, run a code interpreter, and maintain a conversation — all with different latency profiles and failure modes — a single agent becomes a bottleneck. It either serializes everything (slow) or tries to juggle parallel tool calls with a monolithic prompt that degrades in quality as complexity grows. Multi-agent systems exist to solve this structural problem.

In production, multi-agent architectures are how organizations decompose large agentic workloads into manageable, testable, and independently deployable units. A warehouse optimization system does not use one agent that understands inventory, routing, sensor data, and natural language interaction. It uses specialized agents that each own a domain, coordinated by a routing or orchestration layer. NVIDIA's blueprints — the Multi-Agent Intelligent Warehouse and the Retail Agentic Commerce system — are concrete examples of this decomposition in production-grade architectures.

The engineering challenge is not "can I make agents talk to each other" — it is coordination, shared state, failure isolation, and knowing when multi-agent is the right call versus over-engineering. This module covers the patterns, protocols, and NVIDIA-specific tooling that make multi-agent systems practical rather than theoretical.

---

## C. Learning Objectives

After completing this module, you will be able to:

1. Identify when a single-agent architecture is insufficient and articulate the specific criteria that justify moving to multi-agent.
2. Compare and contrast five multi-agent coordination patterns (centralized orchestrator, hierarchical, peer-to-peer, pipeline, competitive) with their tradeoffs.
3. Configure a NAT Router Agent to dynamically dispatch requests to specialized sub-agents with routing logic and fallback handling.
4. Set up and configure a NAT A2A Server for distributed agent-to-agent communication, including agent discovery.
5. Analyze the Multi-Agent Intelligent Warehouse Blueprint architecture and extract reusable coordination patterns.
6. Explain the Agentic Commerce Protocol (ACP) and Universal Commerce Protocol (UCP) from the Retail Agentic Commerce Blueprint.
7. Design a shared state management strategy for multi-agent systems that avoids race conditions and state inconsistency.
8. Diagnose multi-agent failure modes including cascading failures, infinite delegation loops, and conflicting outputs.

---

## D. Required Concepts

- Completion of Module 3 (Tool Use and Function Calling) — you must understand how a single agent invokes tools.
- Completion of Module 5 (Agentic Reasoning Patterns) — you must understand ReAct, planning, and single-agent control flow.
- Basic understanding of distributed systems concepts: message passing, eventual consistency, fault tolerance.
- Familiarity with NAT agent configuration (Module 6).

---

## E. Core Lesson Content

### [BEGINNER] Why Multi-Agent: When One Agent Is Not Enough

A single agent should be your default. Multi-agent adds coordination overhead, debugging complexity, and failure surface area. Move to multi-agent only when you hit one of these walls:

**Task complexity exceeding context window utility.** When the combined prompt — system instructions, tool definitions, conversation history, and retrieved context — grows so large that LLM output quality degrades. Splitting into specialized agents with focused prompts keeps each agent in its effective operating range.

**Domain specialization.** A financial analysis agent and a customer service agent require fundamentally different system prompts, tools, and evaluation criteria. Forcing both into one agent creates prompt conflicts and tool namespace collisions.

**Parallel execution requirements.** A user query that requires simultaneously searching a knowledge base, querying a live database, and checking an external API benefits from agents running in parallel rather than serial tool calls.

**Separation of concerns for deployment.** When different teams own different capabilities, multi-agent lets each team deploy, version, and monitor their agent independently.

**Decision rule:** If you can solve it with a single agent using well-structured tools, do that. Multi-agent is a scaling response, not a starting point.

### [BEGINNER] Multi-Agent Coordination Patterns

```
Pattern 1: CENTRALIZED ORCHESTRATOR
         +------------------+
         |   Orchestrator   |
         |      Agent       |
         +--+-----+-----+--+
            |     |     |
         +--v+ +--v+ +--v+
         | A1| | A2| | A3|
         +---+ +---+ +---+
   One agent dispatches to others.
   Simple control flow, single point of failure.

Pattern 2: HIERARCHICAL
         +----------+
         | Manager  |
         +----+-----+
              |
       +------+------+
       |             |
   +---v---+    +----v---+
   | Team   |    | Team   |
   | Lead A |    | Lead B |
   +--+--+--+    +---+----+
      |  |            |
    +v+ +v+        +--v+
    |W1| |W2|      | W3|
    +--+ +--+      +---+
   Tree structure. Managers delegate to leads, leads to workers.
   Good for complex organizations of agents.

Pattern 3: PEER-TO-PEER
    +----+     +----+
    | A1 |<--->| A2 |
    +--+-+     +-+--+
       ^         ^
       |         |
       v         v
    +--+-+     +-+--+
    | A3 |<--->| A4 |
    +----+     +----+
   Agents communicate directly. No central coordinator.
   Flexible but hard to debug and control.

Pattern 4: PIPELINE
    +----+    +----+    +----+    +----+
    | A1 |--->| A2 |--->| A3 |--->| A4 |
    +----+    +----+    +----+    +----+
   Sequential handoff. Each agent processes and passes results.
   Simple, predictable, but no parallelism.

Pattern 5: COMPETITIVE
         +----------+
         |  Query   |
         +----+-----+
              |
       +------+------+
       |      |      |
    +--v+ +---v+ +---v+
    | A1| | A2 | | A3 |
    +--++ +--+-+ +-+--+
       |    |      |
       v    v      v
    +-----------------+
    |   Judge/Merger  |
    +-----------------+
   Multiple agents propose answers. A judge selects or merges.
   High compute cost, but better quality for ambiguous tasks.
```

Each pattern has distinct tradeoffs in latency, fault tolerance, complexity, and cost. Centralized orchestrator is the most common starting point. Pipeline works for well-defined sequential workflows. Competitive is expensive but useful for high-stakes decisions.

### [INTERMEDIATE] NAT Router Agent: Dynamic Dispatch

The NAT Router Agent implements the centralized orchestrator pattern within NVIDIA Agent Toolkit. It accepts incoming requests and routes them to specialized sub-agents based on configurable routing logic.

**Key configuration elements:**

- **Sub-agent registration:** Each specialized agent is registered with the router, including its capabilities description, endpoint, and input/output schema.
- **Routing logic:** The router uses an LLM call (or rule-based logic) to determine which sub-agent should handle a request. The LLM receives the user query and the capability descriptions of all registered sub-agents.
- **Fallback handling:** When no sub-agent matches or the selected sub-agent fails, the router can fall back to a default agent, return an error, or retry with a different sub-agent.

```
Router Agent Dispatch Flow:
    User Query
        |
        v
  +-----------+
  |  Router   |   1. Analyze query
  |  Agent    |   2. Match to sub-agent capabilities
  |           |   3. Route to best match
  +--+--+--+--+
     |  |  |
     |  |  +---------> [Finance Agent] - handles financial queries
     |  +------------> [Search Agent]  - handles knowledge retrieval
     +---------------> [Code Agent]   - handles code generation
                        |
                   (fallback)
                        v
                [Default Agent] - handles unmatched queries
```

**Configuration pattern (illustrative):**
```python
from nat import RouterAgent, SubAgent

router = RouterAgent(
    name="main_router",
    llm_model="meta/llama-3.1-70b-instruct",
    routing_strategy="llm",  # or "rule_based"
    sub_agents=[
        SubAgent(name="finance", description="Financial analysis and reporting",
                 endpoint="http://finance-agent:8080"),
        SubAgent(name="search", description="Knowledge base search and retrieval",
                 endpoint="http://search-agent:8080"),
    ],
    fallback_agent="search",
    max_routing_retries=2
)
```

*Note: The exact API shown above is illustrative of NAT Router Agent patterns. Refer to current NAT documentation for precise method signatures.*

### [INTERMEDIATE] NAT A2A Protocol: Distributed Agent Communication

The Agent-to-Agent (A2A) protocol enables agents to discover and communicate with each other across network boundaries. Unlike the Router Agent pattern where one agent controls dispatch, A2A enables peer-to-peer and hierarchical patterns across distributed deployments.

**Core A2A concepts:**

- **Agent discovery:** Agents register their capabilities with a discovery service. Other agents query this service to find agents that can handle specific tasks.
- **A2A Server:** Each agent exposes an A2A server endpoint that accepts standardized requests from other agents.
- **Message protocol:** Structured message format including task description, context, constraints, and response format expectations.
- **Session management:** A2A maintains session context across multi-turn agent interactions.

```
A2A Communication Topology:
    +-------------------+
    | Discovery Service |
    +--------+----------+
             |
    +--------+--------+--------+
    |        |        |        |
 +--v--+  +--v--+  +--v--+  +--v--+
 |Agent|  |Agent|  |Agent|  |Agent|
 | A2A |  | A2A |  | A2A |  | A2A |
 |Srvr |  |Srvr |  |Srvr |  |Srvr |
 +--+--+  +--+--+  +--+--+  +--+--+
    |        |        |        |
    +--------+--------+--------+
         Direct A2A messaging
```

The A2A pattern is essential when agents are owned by different teams, deployed across different infrastructure, or need to evolve independently. It trades simplicity for flexibility and independent deployability.

### [INTERMEDIATE] NVIDIA Multi-Agent Intelligent Warehouse Blueprint

This blueprint demonstrates multi-agent coordination for warehouse operations. The architecture uses specialized agents for distinct warehouse functions:

- **Inventory Agent:** Tracks stock levels, predicts replenishment needs, integrates with sensor data.
- **Routing Agent:** Optimizes pick paths and delivery routes within the warehouse.
- **Monitoring Agent:** Processes real-time sensor feeds for anomaly detection (temperature, equipment status).
- **NL Interface Agent:** Handles natural language queries from warehouse operators ("What's the status of zone B?").

**Extractable patterns from this blueprint:**

1. **Domain-bounded agents:** Each agent owns a clearly bounded domain with its own data sources and tools.
2. **Event-driven coordination:** Agents react to events (sensor alerts, inventory thresholds) rather than polling.
3. **Shared context via data layer:** Agents share state through a common data store rather than passing full context in messages.
4. **Human-in-the-loop via NL agent:** The NL agent serves as a bridge between human operators and the agent system.

### [ADVANCED] Retail Agentic Commerce Blueprint: ACP and UCP

The Retail Agentic Commerce Blueprint introduces two protocols for multi-agent commerce:

**Agentic Commerce Protocol (ACP):** Defines how shopping agents interact with merchant systems. An agent representing a customer can browse products, negotiate prices, and initiate checkout — all through structured protocol messages. ACP ensures that the merchant retains control over pricing, inventory, and fulfillment decisions.

**Universal Commerce Protocol (UCP):** A standardization layer that allows agents from different platforms to interact with any UCP-compliant merchant. This enables interoperability — a customer's agent built on one platform can transact with a merchant's agent built on another.

**Key architectural insight:** These protocols enforce *asymmetric trust*. The customer agent proposes, the merchant agent disposes. The merchant agent has veto power over any transaction, preventing autonomous agents from making unauthorized purchases.

### [ADVANCED] Shared State Management

Multi-agent systems must share context. The approaches, in order of complexity:

1. **Message passing (stateless):** Each message contains all necessary context. Simple, but messages grow large.
2. **Shared key-value store:** Agents read/write to a common store (Redis, etc.). Risk of race conditions.
3. **Event sourcing:** All state changes are events in an append-only log. Agents reconstruct state by replaying events. Complex but auditable.
4. **CQRS (Command Query Responsibility Segregation):** Separate read and write models. Agents that modify state use commands; agents that read use queries.

**Preventing race conditions:** Use optimistic locking (version numbers on state objects), idempotent operations, or designate a single agent as the state owner for each data domain.

### [ADVANCED] Scaling Decisions

**When to add more agents:** The task naturally decomposes into independent subtasks with different latency profiles, failure modes, or team ownership.

**When to make one agent smarter:** The task is inherently sequential, the coordination overhead of multiple agents exceeds the benefit, or the quality degradation comes from poor prompting rather than architectural limitations.

---

## Hands-on NVIDIA Platform Interaction

**Title:** Build a Router Agent with Specialized Sub-Agents

**NVIDIA service/tool touched:** NVIDIA NeMo Agent Toolkit (NAT) — Router Agent, Tool Calling Agent (sub-agents); NVIDIA NIM hosted endpoint (via build.nvidia.com); Multi-Agent Intelligent Warehouse Blueprint reference (from build.nvidia.com/blueprints)

**Required credentials:** `NVIDIA_API_KEY` (obtained from build.nvidia.com)

**Execution environment:** Local NAT installation + hosted NIM endpoint (LLM inference runs on NVIDIA-hosted infrastructure; agent orchestration runs locally)

**Exercise:**

1. Create 2 specialist agent YAML configs: (a) a "math agent" with a calculator tool, (b) a "search agent" with a document search tool.
2. Create a Router Agent YAML config that dispatches to the math agent for calculation queries and the search agent for knowledge queries.
3. Run the Router Agent. Send it a math query and verify it routes to the math agent. Send a knowledge query and verify it routes to the search agent. Send an ambiguous query and observe routing behavior.
4. Study the Multi-Agent Warehouse Blueprint architecture diagram (from build.nvidia.com/blueprints). Map the coordination patterns to the patterns taught in this module. Document: which pattern does the Warehouse Blueprint use? What would you change if latency requirements were tighter?

**Expected artifact/output:** (a) Working Router Agent with 2 specialists that correctly dispatches based on query type. (b) 1-page architecture comparison between your Router system and the Warehouse Blueprint, identifying coordination patterns used in each.

**Verification step:** Router correctly dispatches to specialists based on query type. Execution traces show routing decisions — the math query trace shows the Router selecting the math agent and the math agent executing the calculator tool, while the knowledge query trace shows the Router selecting the search agent. The ambiguous query trace reveals the Router's fallback or best-guess behavior.

**Interaction classification:** Local NVIDIA tooling interaction (NAT Router Agent) + hosted NVIDIA API interaction (NIM endpoint) + Blueprint reference study

---

## F. Terminology Box

| Term | Definition |
|------|-----------|
| **Router Agent** | An agent that accepts requests and dispatches them to specialized sub-agents based on routing logic |
| **A2A (Agent-to-Agent)** | Protocol enabling agents to discover and communicate with each other across network boundaries |
| **Centralized Orchestrator** | Multi-agent pattern where one coordinator agent dispatches work to all other agents |
| **Pipeline Pattern** | Multi-agent pattern where agents process sequentially, each passing output to the next |
| **Competitive Pattern** | Multi-agent pattern where multiple agents produce answers and a judge selects the best |
| **ACP (Agentic Commerce Protocol)** | Protocol defining how shopping agents interact with merchant systems |
| **UCP (Universal Commerce Protocol)** | Standardization layer enabling cross-platform agent-to-merchant interoperability |
| **Agent Discovery** | Mechanism by which agents register and find other agents' capabilities |
| **Cascading Failure** | When one agent's failure causes downstream agents to fail in sequence |
| **Delegation Loop** | Pathological state where agents repeatedly delegate to each other without completing work |

---

## G. Common Misconceptions

1. **"Multi-agent is always better than single-agent."** False. Multi-agent adds coordination overhead, debugging complexity, and latency. A well-designed single agent with good tools outperforms a poorly designed multi-agent system every time. Multi-agent is a response to specific scaling constraints.

2. **"Agents in a multi-agent system need to use the same LLM."** Each agent can use a different model optimized for its task. A routing agent might use a fast, cheap model. A complex reasoning agent might use a larger model. A code agent might use a code-specialized model.

3. **"The Router Agent pattern requires an LLM for routing."** Routing can be rule-based (keyword matching, regex, classifier) or LLM-based. LLM routing is more flexible but slower and more expensive. Many production systems use rule-based routing with LLM fallback.

4. **"A2A means agents talk in natural language."** A2A uses structured message protocols. While the content may include natural language descriptions, the protocol itself is structured with defined fields for task type, context, constraints, and response format.

5. **"More agents means more intelligence."** Agent count does not correlate with system intelligence. Poorly decomposed multi-agent systems where agents have overlapping responsibilities or unclear boundaries perform worse than consolidated alternatives.

6. **"Multi-agent systems are self-organizing."** In practice, multi-agent systems require explicit coordination patterns, failure handling, and monitoring. Emergent behavior in production multi-agent systems is a bug, not a feature.

---

## H. Failure Modes / Anti-Patterns

1. **Cascading failure:** Agent A depends on Agent B, which depends on Agent C. When C fails, B fails, then A fails. **Fix:** Circuit breakers, timeouts, and fallback behaviors at each agent boundary.

2. **Infinite delegation loop:** Agent A decides Agent B should handle the query. Agent B decides Agent A should handle it. **Fix:** Maximum delegation depth counters, loop detection via request ID tracking.

3. **Conflicting agent outputs:** Two agents give contradictory answers to the same question. **Fix:** Explicit conflict resolution strategy — either a judge agent, priority ranking, or domain-based authority rules.

4. **State synchronization bugs:** Agent A reads stale state because Agent B's update has not propagated. **Fix:** Version numbers on shared state, read-after-write consistency guarantees, or single-writer patterns.

5. **Over-decomposition:** Creating 15 agents for a task that needs 3. Each agent boundary adds latency and failure surface. **Fix:** Start with fewer, larger agents and split only when you have evidence of the constraints listed in the "Why Multi-Agent" section.

6. **God Router:** A router agent that contains so much logic it becomes a single-agent system with extra steps. **Fix:** Router should only route. Business logic belongs in sub-agents.

7. **Missing observability:** Cannot trace a request through the multi-agent system to understand which agent did what. **Fix:** Distributed tracing (request IDs, correlation IDs) propagated through all agent calls.

---

## I. Hands-On Lab

**Lab 7: Multi-Agent Customer Service System**

Build a multi-agent system using NAT Router Agent with three specialized sub-agents: a product information agent, an order status agent, and a returns/refund agent. The router dispatches customer queries to the appropriate sub-agent. Implement fallback handling when the router cannot determine the correct sub-agent. Test with ambiguous queries that could match multiple agents.

---

## J. Stretch Lab

**Stretch Lab 7: A2A Cross-Service Communication**

Extend the customer service system to use NAT A2A protocol. Deploy each sub-agent as an independent A2A server. Implement agent discovery so the router dynamically finds available agents. Add a fourth agent mid-session and verify the router begins routing to it without restart. Simulate agent failure and verify fallback behavior.

---

## K. Review Quiz

**Q1:** A user query requires searching a knowledge base and running a SQL query simultaneously. Which multi-agent pattern is most appropriate?
**(a)** Pipeline **(b)** Centralized orchestrator with parallel dispatch **(c)** Competitive **(d)** Peer-to-peer

**Answer:** (b). The orchestrator dispatches to both agents in parallel and combines results. Pipeline would serialize them unnecessarily.

**Q2:** What is the primary purpose of the NAT Router Agent?
**Answer:** To accept incoming requests and dynamically dispatch them to specialized sub-agents based on routing logic, with fallback handling when no sub-agent matches.

**Q3:** In the A2A protocol, what is the role of the discovery service?
**Answer:** It allows agents to register their capabilities and enables other agents to find agents that can handle specific tasks, without hardcoding endpoints.

**Q4:** True or False: The Retail Agentic Commerce Blueprint's ACP gives the customer agent authority to finalize transactions.
**Answer:** False. ACP enforces asymmetric trust — the merchant agent retains control over pricing and fulfillment decisions.

**Q5:** An agent system has Agent A → Agent B → Agent C. Agent C begins failing. What failure mode is this, and what is the standard mitigation?
**Answer:** Cascading failure. Mitigation: circuit breakers and timeouts at each agent boundary, plus fallback behaviors.

**Q6:** When should you choose the pipeline pattern over the centralized orchestrator pattern?
**Answer:** When the task naturally decomposes into sequential processing stages where each stage's output is the next stage's input and no parallelism is needed.

**Q7:** What mechanism prevents infinite delegation loops in multi-agent systems?
**Answer:** Maximum delegation depth counters and loop detection via request ID tracking across agent boundaries.

**Q8:** In the Multi-Agent Intelligent Warehouse Blueprint, how do agents share state?
**Answer:** Through a shared data layer (common data store) rather than passing full context in inter-agent messages.

**Q9:** You have a working single-agent system. Response quality is good but latency is high because it makes 5 sequential tool calls. Should you switch to multi-agent?
**Answer:** Only if the tool calls can be parallelized across independent agents. If the calls are dependent (each needs the previous result), multi-agent will not reduce latency. First check if the single agent can make parallel tool calls.

**Q10:** What distinguishes ACP from UCP in the Retail Agentic Commerce Blueprint?
**Answer:** ACP defines how shopping agents interact with a specific merchant system. UCP is a standardization layer enabling cross-platform interoperability so agents from different platforms can transact with any UCP-compliant merchant.

---

## L. Mini Project

**Project: Multi-Agent Research Assistant**

Build a three-agent system: (1) a Search Agent that queries a knowledge base, (2) a Synthesis Agent that combines search results into coherent summaries, (3) a Critique Agent that evaluates the synthesis for accuracy and completeness. Use the centralized orchestrator pattern with a Router Agent coordinating the workflow. The router should first dispatch to Search, pass results to Synthesis, then pass the synthesis to Critique for review. If the Critique agent flags issues, the router should loop back to Search with refined queries. Implement a maximum loop count of 3 to prevent infinite refinement.

---

## M. How This May Appear on the Exam

1. **Architecture selection:** Given a scenario description (e.g., "a system needs to handle customer queries across 4 departments with different data sources"), select the most appropriate multi-agent pattern and justify it.

2. **NAT Router Agent configuration:** Questions about how routing logic works, how sub-agents are registered, and how fallback handling is configured.

3. **A2A protocol mechanics:** Questions about agent discovery, A2A server setup, and the difference between Router Agent dispatch and A2A communication.

4. **Blueprint analysis:** Given a description of the Warehouse or Commerce blueprint, identify which agents are responsible for which functions and explain the coordination pattern used.

5. **Failure diagnosis:** Given a multi-agent system exhibiting a specific symptom (e.g., requests cycling indefinitely, inconsistent state), identify the failure mode and prescribe a fix.

---

## N. Checklist for Mastery

- [ ] I can list 5 specific criteria that justify moving from single-agent to multi-agent architecture.
- [ ] I can draw and explain all 5 coordination patterns (orchestrator, hierarchical, peer-to-peer, pipeline, competitive) with tradeoffs.
- [ ] I can configure a NAT Router Agent with sub-agent registration, routing logic, and fallback handling.
- [ ] I can set up a NAT A2A Server and explain how agent discovery works.
- [ ] I can describe the Multi-Agent Intelligent Warehouse Blueprint architecture and identify at least 3 reusable patterns.
- [ ] I can explain ACP and UCP from the Retail Agentic Commerce Blueprint and describe their trust model.
- [ ] I can choose an appropriate shared state strategy (message passing, shared store, event sourcing, CQRS) for a given scenario.
- [ ] I can diagnose cascading failures, delegation loops, and state synchronization bugs in multi-agent systems.
- [ ] I can implement distributed tracing across a multi-agent system for observability.
- [ ] I can articulate when adding agents is the wrong response and the system needs a smarter single agent instead.
- [ ] I can design conflict resolution strategies for multi-agent systems with potentially contradictory outputs.
- [ ] I can explain the difference between synchronous and asynchronous agent communication and when each is appropriate.
