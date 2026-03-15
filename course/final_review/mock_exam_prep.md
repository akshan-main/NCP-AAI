# Mock Exam Preparation

> **IMPORTANT DISCLAIMER**: NVIDIA does not publish sample exam questions. ALL questions below are INFERRED from the exam domain descriptions, competency statements, and NVIDIA official documentation. They may not reflect actual exam format, difficulty, or question style. Use them as study tools, not as exam previews.

---

## 1. Blueprint-Aligned Revision Checklist

For each exam domain, the specific topics to review before attempting practice questions. Module references point to where the topic is covered in depth.

### Domain 1: Agent Architecture and Design (15%)

1. Seven NAT agent types and when to use each (ReAct, Reasoning, ReWOO, Router, Sequential Executor, Tool Calling, Responses API Agent) — **M2, Section E.1-E.7**
2. Architecture tradeoffs: latency vs. accuracy vs. cost for each agent pattern — **M2, Section E.8**
3. Single-agent vs. multi-agent decision criteria — **M2, Section E.9; M7, Section D**
4. NVIDIA Blueprint reference architectures (AI-Q, Warehouse, Data Flywheel) and which patterns each uses — **M2, Section F**
5. Router pattern: dispatch logic, fallback handling, confidence thresholds — **M2, Section E.4; M7, Section E.2**
6. ReAct vs. ReWOO: interleaved vs. plan-first execution and when each is superior — **M2, Section E.5**
7. Agent loop phases (Perceive, Reason, Act, Observe) and how NAT implements each — **M1, Section E; M2, Section D**

### Domain 2: Agent Development (15%)

1. NAT YAML configuration structure: agent type, model, tools, system prompt, parameters — **M6, Section E.1**
2. NAT Functions, Function Groups, and Per-User Functions — **M5, Section E.3; M6, Section E.2**
3. Tool registry and MCP client integration — **M5, Section E.4; M6, Section E.3**
4. NAT middleware types: caching, defense, logging, red teaming — **M6, Section E.4**
5. NAT plugin system for third-party frameworks (LangChain, LlamaIndex, CrewAI, etc.) — **M6, Section E.5**
6. Prompt engineering: system prompt design, few-shot examples, structured output schemas — **M5, Section E.1-E.2**
7. NAT Parameter Optimizer (Agent Hyperparameter Optimizer) for prompt and config tuning — **M5, Section E.5**
8. NAT authentication mechanisms (API Key, OAuth2, Bearer, HTTP Basic, MCP accounts) — **M6, Section E.6**

### Domain 3: Evaluation and Tuning (13%)

1. NAT Evaluation Framework: RAG eval, red teaming, trajectory eval, custom evaluators — **M8, Section E.1**
2. RAG evaluation metrics: faithfulness, relevance, context precision, context recall — **M8, Section E.2**
3. Retriever evaluation: MRR, NDCG, recall@k, precision@k — **M8, Section E.3**
4. LLM-as-Judge patterns and NAT custom evaluator implementation — **M8, Section E.4**
5. Agent trajectory evaluation: expected vs. actual tool sequences — **M8, Section E.5**
6. NAT Profiler: workflow-to-tool-level token and latency profiling — **M8, Section E.6**
7. NAT Sizing Calculator for resource planning — **M8, Section E.7**
8. Data Flywheel Blueprint for continuous optimization — **M8, Section F**

### Domain 4: Deployment and Scaling (13%)

1. NAT deployment servers: FastAPI, MCP, FastMCP, A2A — **M10, Section E.1**
2. NIM microservices for LLM serving: architecture, API, configuration — **M10, Section E.2**
3. NIM Operator for Kubernetes deployment — **M10, Section E.3**
4. NVIDIA Dynamo for inference acceleration — **M10, Section E.4**
5. Docker Compose for multi-service agent deployment — **M10, Section E.5**
6. Kubernetes/Helm deployment patterns for agent systems — **M10, Section E.6**
7. Horizontal vs. vertical scaling strategies for different agent components — **M10, Section E.7**
8. GPU resource planning and model sizing — **M10, Section E.8**

### Domain 5: Cognition, Planning, and Memory (10%)

1. NAT Reasoning Agent and test-time compute (multi-LLM generation, planning, selection) — **M3, Section E.1**
2. NAT ReWOO Agent: plan generation and plan execution separation — **M3, Section E.2**
3. Reasoning strategies: Chain-of-Thought, Tree-of-Thought, self-reflection — **M3, Section E.3**
4. NAT Memory systems and Object Stores (in-memory, MySQL, Redis, S3) — **M3, Section E.4**
5. NAT Automatic Memory Wrapper for transparent memory management — **M3, Section E.5**
6. Memory architecture patterns: episodic, semantic, procedural — **M3, Section E.6**
7. Planning algorithms and when to separate planning from execution — **M3, Section E.7**

### Domain 6: Knowledge Integration and Data Handling (10%)

1. Seven-stage RAG pipeline and NVIDIA tools for each stage — **M4, Section E.1**
2. NeMo Curator for GPU-accelerated data preparation — **M4, Section E.2**
3. Chunking strategies: fixed-size, recursive, semantic, document-structure-aware — **M4, Section E.3**
4. NIM embedding endpoints and Llama 3.2 NV EmbedQA — **M4, Section E.4**
5. Milvus + cuVS for GPU-accelerated vector search — **M4, Section E.5**
6. Hybrid retrieval: dense + sparse, when each dominates — **M4, Section E.6**
7. Reranking with Llama 3.2 NV RerankQA and cross-encoder architecture — **M4, Section E.7**
8. AI-Q Blueprint component mapping — **M4, Section F**

### Domain 7: NVIDIA Platform Implementation (7%)

1. NAT end-to-end workflow: config → build → deploy → evaluate — **M6, Section D**
2. NIM API: endpoints, request/response format, model selection — **M1, Section E.4; M10, Section E.2**
3. NeMo microservices ecosystem: Customizer, Evaluator, Retriever, Guardrails — **M1, Section E.3**
4. NVIDIA Blueprint implementations and how to use them as starting points — **M2, Section F; M4, Section F**
5. NAT plugin system and framework interoperability — **M6, Section E.5**

### Domain 8: Run, Monitor, and Maintain (5%)

1. NAT observability: Phoenix, Weave, Langfuse, OpenTelemetry integration — **M11, Section E.1**
2. NAT execution tracing: workflow → step → tool level granularity — **M11, Section E.2**
3. Token counting and latency metrics for cost management — **M11, Section E.3**
4. NAT redaction processors for telemetry data — **M11, Section E.4**
5. Dashboard design and alerting strategies for agent systems — **M11, Section E.5**
6. Maintenance runbooks and troubleshooting patterns — **M11, Section E.6**

### Domain 9: Safety, Ethics, and Compliance (5%)

1. NeMo Guardrails: Colang 2.0 syntax, five rail types (input, output, dialog, topical, execution) — **M9, Section E.1**
2. Safety NIMs: Content Safety, Jailbreak Detection, Topic Control — **M9, Section E.2**
3. NeMo Auditor for pre-deployment safety auditing — **M9, Section E.3**
4. NAT defense middleware and red-teaming middleware — **M9, Section E.4**
5. PII detection and redaction (GLiNER, Presidio, PrivateAI integration) — **M9, Section E.5**
6. Execution rails for agentic security (tool call validation) — **M9, Section E.6**
7. Fact-checking rails for hallucination prevention — **M9, Section E.7**

### Domain 10: Human-AI Interaction and Oversight (5%)

1. NAT interactive workflows (WebSocket, bidirectional communication) — **M12, Section E.1**
2. Execution rails for human approval gates — **M12, Section E.2**
3. NAT per-user functions and authentication for personalized agents — **M12, Section E.3**
4. Escalation path design: confidence thresholds, severity routing — **M12, Section E.4**
5. Feedback loop design: logging human decisions, aggregating patterns, retraining — **M12, Section E.5**

---

## 2. Topic Heatmap by Importance

### Tier 1: 30% of Exam Weight — Study First
- **Agent Architecture and Design (15%)**: Agent patterns, NAT agent types, architecture tradeoffs, Blueprints
- **Agent Development (15%)**: NAT YAML, tools, MCP, middleware, plugins, prompt engineering

These two domains alone account for nearly a third of the exam. A student who masters NAT agent types, can write YAML configurations, understands tool registries, and can reason about architecture tradeoffs is well-positioned.

### Tier 2: 26% of Exam Weight — Study Second
- **Evaluation and Tuning (13%)**: NAT eval framework, RAG metrics, trajectory eval, profiler
- **Deployment and Scaling (13%)**: NAT deployment servers, NIM, NIM Operator, Docker Compose, K8s

These domains test whether you can take an agent from development to production. Evaluation knowledge is essential because NVIDIA emphasizes measurable quality. Deployment knowledge is essential because the exam tests NIM and NAT server configuration.

### Tier 3: 20% of Exam Weight — Study Third
- **Cognition, Planning, and Memory (10%)**: Reasoning agents, ReWOO, test-time compute, memory systems
- **Knowledge Integration and Data Handling (10%)**: RAG pipeline, NeMo Retriever, NeMo Curator, embeddings, vector stores

These domains have strong NVIDIA documentation. A student who completed the RAG labs and memory/planning labs should be well-prepared.

### Tier 4: 22% of Exam Weight — Do Not Neglect
- **Platform Implementation (7%)**: NAT workflow, NIM API, NeMo ecosystem
- **Run, Monitor, and Maintain (5%)**: Observability, tracing, telemetry
- **Safety, Ethics, and Compliance (5%)**: Guardrails, Colang, Safety NIMs
- **Human-AI Interaction and Oversight (5%)**: Interactive workflows, approval gates, escalation

Although each domain is only 5-7%, together they constitute over a fifth of the exam. A student who skips these risks losing 12-15 questions. Platform Implementation in particular is woven through the entire course — a student proficient in Tiers 1-3 likely already knows most of it. Safety and Monitoring require targeted study of NeMo Guardrails and NAT observability.

---

## 3. Weak-Area Remediation Plan

If you score poorly on practice questions in a domain, use this targeted remediation plan.

### Domain 1: Agent Architecture (scoring < 60%)

**Root cause**: Likely confusing agent types or unable to match requirements to patterns.
- **Immediate**: Re-read M2, Sections E.1-E.7. Make a comparison table of all 7 NAT agent types with columns: pattern, use case, latency profile, token cost, when NOT to use.
- **Practice**: For each of the 7 agent types, write a one-paragraph scenario where that type is the correct choice and explain why the other 6 are wrong.
- **Lab**: Redo Lab 2 and implement at least 3 different agent types for the same task. Measure latency and token cost differences.
- **Verify**: Can you draw the ReAct loop, Router dispatch, and ReWOO plan-execute cycle from memory?

### Domain 2: Agent Development (scoring < 60%)

**Root cause**: Likely unfamiliar with NAT YAML configuration or tool registration.
- **Immediate**: Re-read M5, Section E.3 (Functions) and M6, Sections E.1-E.3 (YAML, Function Groups, MCP).
- **Practice**: Write NAT YAML configs from scratch for 5 different agent scenarios without looking at examples.
- **Lab**: Redo Lab 5 (tool-using agent) and Lab 6 (NAT pipeline). Focus on YAML configuration and tool registry.
- **Verify**: Can you write a complete NAT YAML configuration from memory that includes agent type, model, tools, and system prompt?

### Domain 3: Evaluation (scoring < 60%)

**Root cause**: Likely unclear on which metrics measure what, or unfamiliar with NAT eval framework API.
- **Immediate**: Re-read M8, Sections E.1-E.5. Make a metrics reference card: metric name, what it measures, how it is computed, what threshold is "good."
- **Practice**: Given a RAG system with faithfulness 0.75 and relevance 0.90, diagnose what is likely wrong (retrieval is good but generation is adding information not in context).
- **Lab**: Redo Lab 8. Run the full evaluation pipeline and interpret every metric.
- **Verify**: Can you explain the difference between context precision and context recall? Between faithfulness and relevance?

### Domain 4: Deployment (scoring < 60%)

**Root cause**: Likely unfamiliar with NAT deployment server types or NIM configuration.
- **Immediate**: Re-read M10, Sections E.1-E.3. Make a table: server type (FastAPI, MCP, FastMCP, A2A), use case, protocol, when to choose.
- **Practice**: Write a Docker Compose file from scratch for a NAT agent with NIM endpoint.
- **Lab**: Redo Lab 10 (NIM deployment). Focus on server configuration and health checks.
- **Verify**: Can you explain the difference between NAT FastAPI Server and NAT A2A Server? When would you use each?

### Domain 5: Cognition/Memory (scoring < 60%)

**Root cause**: Likely confusing memory types or unclear on reasoning agent internals.
- **Immediate**: Re-read M3, Sections E.1-E.5. Make a diagram showing how NAT memory flows: input → memory retrieval → LLM context → response → memory storage.
- **Practice**: For each NAT Object Store (in-memory, MySQL, Redis, S3), identify the use case where that backend is the right choice.
- **Lab**: Redo Lab 3. Implement memory with at least 2 different object stores and compare behavior.
- **Verify**: Can you explain what the Automatic Memory Wrapper does and when you would NOT use it?

### Domain 6: Knowledge/RAG (scoring < 60%)

**Root cause**: Likely unclear on the RAG pipeline stages or NVIDIA-specific tooling.
- **Immediate**: Re-read M4, Sections E.1-E.7. Draw the 7-stage RAG pipeline and label each stage with the NVIDIA tool.
- **Practice**: Given a corpus of API documentation, choose a chunking strategy and justify it. Given a corpus of research papers, choose a different strategy and justify why.
- **Lab**: Redo Lab 4. Focus on comparing chunking strategies and measuring retrieval quality.
- **Verify**: Can you explain why cuVS accelerates Milvus search? What is the difference between HNSW and IVF indexes?

### Domain 7: Platform (scoring < 60%)

**Root cause**: Likely unfamiliar with the NeMo ecosystem beyond the tools used in labs.
- **Immediate**: Re-read M1, Section E.3 (stack overview) and M6, Section D (NAT workflow).
- **Practice**: List all NeMo microservices and state the role of each without looking at notes.
- **Verify**: Can you trace a request from user input through NAT → NIM → response, naming every component touched?

### Domain 8: Monitor (scoring < 60%)

**Root cause**: Likely unfamiliar with NAT observability configuration.
- **Immediate**: Re-read M11, Sections E.1-E.4.
- **Practice**: Write NAT observability configuration for Phoenix export from memory.
- **Lab**: Redo Lab 11. Focus on reading traces and identifying performance bottlenecks from trace data.
- **Verify**: Can you explain the three levels of NAT tracing (workflow, step, tool)?

### Domain 9: Safety (scoring < 60%)

**Root cause**: Likely unfamiliar with Colang 2.0 syntax or unclear on rail types.
- **Immediate**: Re-read M9, Sections E.1-E.2. Write out the 5 rail types with a definition and example of each.
- **Practice**: Write Colang 2.0 flows for input safety, topical control, and execution validation from memory.
- **Lab**: Redo Lab 9. Focus on configuring each rail type and testing with adversarial inputs.
- **Verify**: Can you explain the difference between a dialog rail and a topical rail? Between an input rail and an execution rail?

### Domain 10: Human-AI (scoring < 60%)

**Root cause**: Likely unfamiliar with NAT interactive workflow primitives.
- **Immediate**: Re-read M12, Sections E.1-E.3.
- **Practice**: Design an approval workflow on paper: what triggers escalation, who approves, what happens on timeout.
- **Lab**: Redo Lab 12. Focus on the WebSocket interactive workflow and execution rail approval gate.
- **Verify**: Can you describe how to implement a human approval gate using NeMo Guardrails execution rails?

---

## 4. Thirty Hardest Likely Conceptual Questions

> Distributed proportional to exam weights. Each tests NVIDIA-specific knowledge.

### Architecture (5 Questions)

**Q1.** You need an agent that plans all its tool calls upfront before executing any of them, to minimize total LLM calls. Which NAT agent type should you use, and what is the key tradeoff compared to ReAct?

**A1.** Use the **NAT ReWOO Agent**. ReWOO generates a complete plan (all tool calls and their dependencies) in a single LLM call, then executes the plan without further LLM reasoning between steps. The key tradeoff: ReWOO cannot adapt mid-execution if a tool call returns unexpected results, because the plan was fixed before execution. ReAct, by contrast, reasons after each tool call and can change course, but requires an LLM call per step (higher latency and token cost). Choose ReWOO when the task is predictable and plan changes are rare; choose ReAct when the task requires adaptive reasoning.

---

**Q2.** A team is building a customer support system where user requests fall into 5 distinct categories, each requiring different tools and system prompts. They are debating between (a) a single ReAct agent with all tools registered, or (b) a Router Agent dispatching to 5 specialized Tool Calling Agents. Which is better, and why?

**A2.** The **Router Agent with 5 specialized agents** is better. With a single ReAct agent holding all tools, the tool selection space is large, increasing the probability of incorrect tool selection and increasing prompt size (all tool descriptions must be in context). With specialized agents, each has a focused tool set and optimized system prompt. The Router classifies intent once (cheap, fast) and dispatches to the appropriate specialist. This reduces per-agent complexity, enables independent scaling of each specialist, and makes evaluation easier (you can evaluate each specialist independently). The single ReAct agent is simpler to deploy but degrades as tool count grows.

---

**Q3.** The NVIDIA AI-Q Blueprint uses a specific RAG architecture. Name the embedding model, the reranking model, and the vector store it specifies, and explain why a reranking stage is necessary when you already have a good embedding model.

**A3.** AI-Q Blueprint uses **Llama 3.2 NV EmbedQA** for embedding, **Llama 3.2 NV RerankQA** for reranking, and **Milvus with cuVS** for vector storage and search. Reranking is necessary because embedding-based retrieval (bi-encoder) computes query and document representations independently — fast but less accurate. The reranker is a cross-encoder that processes the query and each candidate document together, capturing fine-grained interactions that bi-encoders miss. This two-stage approach gets the speed of bi-encoder retrieval (search millions of documents) with the accuracy of cross-encoder scoring (rerank top-N candidates).

---

**Q4.** What is the NAT Responses API Agent, and how does it differ from the Tool Calling Agent?

**A4.** The **NAT Responses API Agent** is a NAT agent type that conforms to OpenAI's Responses API specification. It differs from the Tool Calling Agent in its API contract: the Responses API Agent uses the Responses API format (with `response` objects containing items like `function_call`, `message`, etc.) rather than the Chat Completions format used by the Tool Calling Agent. The practical difference is interoperability — the Responses API Agent can be used as a drop-in replacement in systems designed for the OpenAI Responses API. The underlying agent behavior (tool calling, single-step execution) is similar, but the API surface is different.

---

**Q5.** Describe the architectural difference between NAT's Sequential Executor and a Router Agent. Give a scenario where each is the correct choice.

**A5.** The **Sequential Executor** runs a fixed sequence of agents/steps in order — the output of step N is the input to step N+1. There is no decision logic; every step always executes. The **Router Agent** examines the input, classifies it, and dispatches to one (or more) of several possible agents — it makes a routing decision. Use the Sequential Executor when every request must go through the same pipeline (e.g., extract → transform → validate → store). Use the Router Agent when requests have different types requiring different handling (e.g., customer inquiry → FAQ agent vs. complaint → escalation agent vs. order status → lookup agent).

---

### Development (5 Questions)

**Q6.** In NAT YAML configuration, what is a "Function Group" and why would you use Per-User Functions?

**A6.** A **Function Group** in NAT is a named collection of functions (tools) that can be assigned to an agent as a unit. Instead of listing individual functions, you assign a function group, making configuration modular and reusable across agents. **Per-User Functions** are functions that are dynamically available based on the authenticated user. For example, an admin user might have access to `delete_record()` while a regular user does not. Per-User Functions enable role-based tool access without creating separate agent configurations per user role. They are configured via NAT's authentication and function registry system.

---

**Q7.** A developer wants to use their existing LangChain agent with NAT's deployment, evaluation, and guardrails infrastructure. How does NAT support this?

**A7.** NAT provides a **plugin system** that supports third-party framework integration. For LangChain specifically, NAT has a LangChain plugin that wraps a LangChain agent into a NAT-compatible workflow. This means the developer can keep their LangChain agent logic but deploy it via NAT's FastAPI/MCP/A2A servers, evaluate it with NAT's evaluation framework, profile it with NAT's profiler, and protect it with NeMo Guardrails middleware. The plugin handles the translation between LangChain's agent interface and NAT's workflow interface. Similar plugins exist for LlamaIndex, CrewAI, AutoGen, and Semantic Kernel.

---

**Q8.** What is the NAT Agent Hyperparameter Optimizer, and what parameters does it optimize?

**A8.** The **NAT Agent Hyperparameter Optimizer** (also called Parameter Optimizer) is a NAT tool that systematically searches for optimal agent configuration parameters. It optimizes parameters such as: system prompt wording, temperature, top-p, max tokens, number of few-shot examples, tool descriptions, and other YAML-configurable parameters. It works by running the agent against an evaluation dataset with different parameter combinations and measuring quality metrics (e.g., faithfulness, accuracy, task completion rate). It then selects the parameter configuration that maximizes the target metric. This replaces manual prompt tuning with systematic optimization.

---

**Q9.** Explain how NAT's MCP (Model Context Protocol) client support works. What problem does MCP solve that a standard tool registry does not?

**A9.** NAT includes an **MCP client** that can connect to any MCP-compatible server to discover and invoke tools. MCP solves the problem of **dynamic tool discovery across organizational boundaries**. A standard tool registry requires all tools to be registered in the agent's configuration at build time. MCP allows an agent to connect to an MCP server at runtime, discover available tools, and invoke them — even if those tools were added after the agent was deployed. This enables scenarios like: an agent connecting to a company's MCP server to access internal APIs that change frequently, without redeploying the agent each time a new tool is added.

---

**Q10.** What are the four types of NAT middleware, and give a concrete use case for each?

**A10.** The four NAT middleware types are:
1. **Caching middleware**: Caches LLM responses for identical or semantically similar inputs. Use case: a FAQ agent where the same questions recur — caching avoids redundant LLM calls, reducing latency and cost.
2. **Defense middleware**: Adds input validation and attack detection before requests reach the agent. Use case: detecting prompt injection attempts in user messages before they reach the LLM.
3. **Logging middleware**: Captures structured logs of all agent interactions (inputs, outputs, tool calls, timing). Use case: audit trail for compliance — logging every action a financial services agent takes.
4. **Red-teaming middleware**: Automatically generates adversarial inputs to test agent robustness. Use case: pre-deployment testing — running the agent against a library of jailbreak, prompt injection, and edge-case inputs to find vulnerabilities.

---

### Evaluation (4 Questions)

**Q11.** Explain the difference between faithfulness and relevance in RAG evaluation. A system has faithfulness 0.95 and relevance 0.60. What is likely wrong?

**A11.** **Faithfulness** measures whether the generated answer is supported by the retrieved passages (no hallucination). **Relevance** measures whether the answer actually addresses the user's question. A system with high faithfulness (0.95) but low relevance (0.60) is accurately reporting what the retrieved passages say, but the retrieved passages are not relevant to the question. The problem is in the **retrieval stage**, not the generation stage. The retriever is returning passages about the wrong topic, and the LLM faithfully summarizes those irrelevant passages. Fix the retriever (better embeddings, reranking, query reformulation) rather than the generator.

---

**Q12.** What is trajectory evaluation in NAT, and how does it differ from output-only evaluation?

**A12.** **Trajectory evaluation** in NAT assesses the agent's sequence of actions (the trajectory) — which tools were called, in what order, with what parameters — not just the final output. Output-only evaluation checks whether the answer is correct; trajectory evaluation checks whether the agent arrived at the answer through the correct process. This matters because an agent can produce the correct answer through an incorrect process (e.g., calling the wrong tool but getting lucky) or through an inefficient process (calling 10 tools when 3 would suffice). Trajectory evaluation catches process errors that output-only evaluation misses. In NAT, you define expected trajectories for test cases and the framework compares actual agent trajectories against them.

---

**Q13.** How does the NAT Profiler differ from the NAT Evaluation Framework? When would you use each?

**A13.** The **NAT Profiler** measures performance characteristics — token counts, latency at every level (workflow, step, tool), and cost estimates. It answers "how fast and how expensive is this agent?" The **NAT Evaluation Framework** measures quality — faithfulness, relevance, accuracy, trajectory correctness. It answers "how good are this agent's outputs?" Use the Profiler when optimizing for speed and cost (e.g., identifying that the reranking step takes 40% of total latency). Use the Evaluation Framework when optimizing for quality (e.g., measuring whether a new system prompt improves faithfulness). In practice, use both: the Evaluation Framework ensures you are not sacrificing quality, and the Profiler ensures you are not wasting resources.

---

**Q14.** The Data Flywheel Blueprint describes a continuous optimization cycle. What are its stages, and how does it connect evaluation to improvement?

**A14.** The **Data Flywheel Blueprint** implements a cycle of: (1) **Collect** — gather production agent interactions (queries, retrieved contexts, generated answers, user feedback); (2) **Evaluate** — run collected data through the NAT evaluation framework to measure quality metrics and identify failure cases; (3) **Curate** — use failures and low-scoring examples as training data, processed via NeMo Curator; (4) **Fine-tune** — use curated data to fine-tune the model via NeMo Customizer (DPO or supervised fine-tuning); (5) **Deploy** — deploy the improved model and restart the cycle. The key insight is that evaluation is not a one-time activity but a continuous input to improvement. Each cycle improves the model on its actual failure modes rather than generic benchmarks.

---

### Deployment (4 Questions)

**Q15.** NAT provides four deployment server types: FastAPI, MCP, FastMCP, and A2A. When would you choose A2A over FastAPI?

**A15.** Choose **A2A Server** when the agent needs to communicate with other agents in a multi-agent system using the Agent-to-Agent protocol. A2A provides structured inter-agent messaging with typed payloads, task delegation, and status tracking — capabilities that a raw FastAPI endpoint does not provide. Choose **FastAPI Server** when the agent serves external clients (humans, applications) via standard REST/WebSocket API. In a multi-agent deployment, the Router agent might use FastAPI (for external clients) while specialist agents use A2A (for inter-agent communication).

---

**Q16.** What is the NIM Operator, and what problem does it solve that raw Kubernetes YAML does not?

**A16.** The **NIM Operator** is a Kubernetes operator that automates the lifecycle management of NIM microservice deployments. It solves problems that raw K8s YAML cannot easily address: automatic GPU resource discovery and allocation, model caching across pod restarts, health monitoring specific to LLM serving (not just HTTP readiness), automatic scaling based on inference load (not just CPU/memory), and rolling updates that maintain model availability. With raw K8s YAML, you must manually configure GPU tolerations, resource limits, readiness probes tuned for model loading time, and PersistentVolumeClaims for model caching. The NIM Operator encapsulates NVIDIA's operational knowledge of how to run inference workloads on Kubernetes.

---

**Q17.** Describe how NVIDIA Dynamo accelerates inference for agent systems. What specific optimizations does it provide?

**A17.** **NVIDIA Dynamo** is an inference acceleration framework that optimizes the LLM serving pipeline. For agent systems, its key optimizations include: (1) **Dynamic batching** — groups incoming requests into batches to maximize GPU utilization, critical when multiple agents share a NIM endpoint; (2) **Prefix caching** — caches KV-cache entries for common prompt prefixes (e.g., system prompts shared across requests), avoiding redundant computation; (3) **Disaggregated serving** — separates the prefill (processing the prompt) and decode (generating tokens) phases, allowing them to run on different hardware optimized for each; (4) **Multi-node inference** — distributes large models across multiple GPUs/nodes with optimized tensor parallelism. For agent systems, prefix caching is particularly valuable because agents reuse the same system prompt for every request.

---

**Q18.** A team wants to deploy a NAT agent with NIM endpoints. Their LLM model requires 2xH100 GPUs. They need to support 50 concurrent users. Describe the minimum deployment architecture.

**A18.** Minimum architecture: (1) **NAT FastAPI Server** — 2-3 replicas behind a load balancer for connection handling (lightweight, CPU-only); (2) **NIM LLM endpoint** — at least 2 instances, each on 2xH100 (4xH100 total), to handle 50 concurrent users (one NIM instance handles ~20-25 concurrent requests for a 70B model); (3) **Redis** — single instance for state/caching; (4) **NeMo Guardrails** — co-located with the NAT server or as a sidecar. If using RAG, add: (5) **NIM Embedding endpoint** — 1 instance on 1xGPU; (6) **Milvus** — single node with cuVS. Total GPU requirement: 4xH100 for LLM + 1xGPU for embeddings. Use the **NAT Sizing Calculator** to validate these estimates against the specific model and workload profile.

---

### Cognition/Memory (3 Questions)

**Q19.** Explain NAT's Test Time Compute feature. How does it differ from standard single-pass generation?

**A19.** NAT's **Test Time Compute** (implemented in the Reasoning Agent) uses multiple LLM calls at inference time to improve answer quality. Instead of a single generation pass, it: (1) **Generates** multiple candidate responses (potentially using different prompts or temperatures); (2) **Plans** by evaluating which candidate best addresses the query; (3) **Selects** the best candidate based on quality criteria. This trades compute (more LLM calls) for quality (better answers). It differs from standard generation in that it introduces a deliberation step — the system reasons about its own outputs before committing to one. The tradeoff is latency and cost: test-time compute may use 3-5x more tokens than single-pass generation but can significantly improve accuracy on complex reasoning tasks.

---

**Q20.** NAT supports four Object Store backends: in-memory, MySQL, Redis, and S3. For each, give the scenario where it is the correct choice for agent memory.

**A20.**
- **In-memory**: Development and testing, or agents with short-lived sessions that do not need persistence. Fast but volatile — memory is lost when the process restarts. Correct for: prototyping, unit tests, and stateless agents.
- **Redis**: Production agents with session memory that need low-latency reads/writes and moderate persistence (Redis can persist to disk). Correct for: conversational agents where memory must survive container restarts but does not need long-term archival.
- **MySQL**: Production agents with long-term memory that must be queryable, auditable, and backed up. Correct for: compliance-sensitive applications where every agent memory must be stored permanently and be searchable.
- **S3**: Agents that store large memory artifacts (documents, images, analysis reports) that do not need low-latency random access. Correct for: archival of agent session histories, storing large context documents that the agent references occasionally.

---

**Q21.** What is the NAT Automatic Memory Wrapper, and when should you NOT use it?

**A21.** The **Automatic Memory Wrapper** in NAT transparently adds memory capabilities to any agent without modifying the agent's logic. It intercepts agent inputs and outputs, automatically storing relevant information in the configured Object Store, and retrieves relevant memories to inject into the agent's context on subsequent calls. You should NOT use it when: (1) the agent's memory requirements are highly specific (e.g., only certain types of information should be remembered, or memory should be structured in a particular way); (2) memory injection might conflict with the agent's context window budget (the wrapper adds retrieved memories to the prompt, which may push out important context); (3) you need fine-grained control over what is stored vs. forgotten (the wrapper's automatic decisions may not match your retention policy); (4) the agent is stateless by design and adding memory would introduce unwanted state dependencies.

---

### Knowledge (3 Questions)

**Q22.** NeMo Curator provides GPU-accelerated data preparation. Name three specific preprocessing operations it supports and explain why GPU acceleration matters for each.

**A22.** Three NeMo Curator operations:
1. **Fuzzy deduplication** (using MinHash LSH): Identifies near-duplicate documents across large corpora. GPU acceleration matters because pairwise similarity computation scales quadratically — at 1M+ documents, CPU-based dedup takes days while GPU acceleration reduces it to hours.
2. **Language identification and filtering**: Classifies document language and filters to target languages. GPU acceleration matters because running language classification models on millions of documents benefits from batch GPU inference.
3. **Quality scoring**: Assigns quality scores to documents using classifier models (e.g., filtering low-quality web scrapes). GPU acceleration matters because the quality classifier is a neural model that processes every document in the corpus — batch GPU inference is orders of magnitude faster than sequential CPU inference.

GPU acceleration transforms these from overnight batch jobs into interactive workflows, enabling iterative data quality improvement.

---

**Q23.** Explain the difference between dense retrieval, sparse retrieval, and hybrid retrieval. When does sparse retrieval outperform dense?

**A23.** **Dense retrieval** converts queries and documents to dense vectors (embeddings) and finds nearest neighbors in vector space. It captures semantic similarity (synonyms, paraphrases). **Sparse retrieval** (e.g., BM25) uses term-frequency-based scoring on token-level inverted indexes. It captures exact keyword matches. **Hybrid retrieval** combines both, typically by running both retrievers and merging results (e.g., reciprocal rank fusion). Sparse retrieval outperforms dense when: (1) queries contain rare technical terms, identifiers, or codes (e.g., "error code NV-4231") that the embedding model has not learned good representations for; (2) queries are very short (1-2 words) where there is insufficient semantic signal for embedding similarity; (3) the document corpus has specialized vocabulary not well-represented in the embedding model's training data. In technical documentation domains, hybrid retrieval typically outperforms either alone.

---

**Q24.** What role does cuVS play in the Milvus vector store, and how does it differ from CPU-based index types?

**A24.** **cuVS** (CUDA Vector Search) is NVIDIA's GPU-accelerated library for approximate nearest neighbor (ANN) search. In Milvus, cuVS replaces CPU-based ANN algorithms (like CPU HNSW or IVF) with GPU-accelerated implementations. The difference: CPU-based HNSW search processes one query at a time through graph traversal — fast for single queries but does not scale linearly with concurrent queries. cuVS leverages GPU parallelism to process many queries simultaneously and accelerates the distance computation step (computing thousands of cosine similarities in parallel). The practical impact is higher throughput under concurrent load: when 100 users search simultaneously, cuVS maintains low latency while CPU HNSW queues requests. cuVS is most beneficial for high-concurrency production workloads; for development with single-digit QPS, CPU HNSW is sufficient.

---

### Platform (2 Questions)

**Q25.** Describe the end-to-end NAT workflow from configuration to production deployment. What are the four major phases?

**A25.** The four major phases of the NAT workflow:
1. **Configure**: Define the agent in YAML — agent type, model endpoint, tools (function registry), system prompt, middleware (caching, defense, logging), guardrails integration. This is the declarative specification of what the agent does.
2. **Build**: Use NAT to instantiate the agent from configuration. Register functions, connect to NIM endpoints, initialize middleware. Test locally with NAT's built-in testing utilities. Iterate on configuration using the Agent Hyperparameter Optimizer.
3. **Evaluate**: Run the NAT evaluation framework — RAG eval, trajectory eval, red teaming, profiling. Use the Sizing Calculator to estimate production resource requirements. Iterate until quality and performance thresholds are met.
4. **Deploy**: Choose a deployment server (FastAPI, MCP, A2A) based on the use case. Containerize. Deploy to Kubernetes via NIM Operator or Docker Compose. Configure observability export. Monitor via Phoenix/Langfuse dashboards.

---

**Q26.** How does the NAT plugin system enable interoperability with third-party frameworks? Name at least four supported frameworks.

**A26.** The NAT plugin system provides adapter layers that translate between third-party agent framework interfaces and NAT's internal workflow representation. This means agents built with other frameworks can leverage NAT's deployment servers, evaluation framework, profiler, middleware, and guardrails without rewriting agent logic. Supported frameworks include: **LangChain**, **LlamaIndex**, **CrewAI**, **AutoGen**, **Semantic Kernel**, and others. Each plugin wraps the framework's agent execution into a NAT-compatible workflow, enabling the agent to be deployed via NAT FastAPI/MCP/A2A servers, evaluated with NAT eval, profiled with NAT profiler, and protected by NeMo Guardrails — regardless of which framework was used to build the agent logic.

---

### Monitor (1 Question)

**Q27.** NAT supports multiple observability backends: Phoenix, Weave, Langfuse, and raw OpenTelemetry. What tracing granularity does NAT provide, and what is a NAT redaction processor?

**A27.** NAT provides **three levels of tracing granularity**: (1) **Workflow level** — the entire agent request, including total latency, total token count, and final status; (2) **Step level** — each reasoning step within the workflow (e.g., each ReAct iteration), including per-step latency and tokens; (3) **Tool level** — each individual tool call, including tool name, parameters, response, and execution time. This hierarchical tracing lets you drill from "this request took 8 seconds" to "the third tool call (database query) took 5 seconds." A **NAT redaction processor** is a telemetry post-processor that removes or masks sensitive data from traces before they are exported to the observability backend. For example, it can redact API keys, user PII, or proprietary content from trace data so that observability dashboards do not expose sensitive information.

---

### Safety (2 Questions)

**Q28.** Name the five rail types in NeMo Guardrails and explain the difference between a dialog rail and a topical rail.

**A28.** The five rail types are:
1. **Input rails** — process user input before it reaches the LLM (e.g., content safety, jailbreak detection)
2. **Output rails** — process LLM output before it reaches the user (e.g., hallucination check, PII redaction)
3. **Dialog rails** — control the conversational flow and structure (e.g., requiring certain exchanges before granting access to certain capabilities)
4. **Topical rails** — restrict the conversation to specific topics (e.g., preventing the agent from discussing politics)
5. **Execution rails** — validate and control tool/action execution (e.g., requiring approval before executing a database write)

The difference between dialog and topical: a **dialog rail** controls the structure and sequence of the conversation (what the bot says when, how it responds to certain interaction patterns), while a **topical rail** controls the subject matter (what topics the bot will and will not engage with). A dialog rail might enforce "always ask for confirmation before proceeding"; a topical rail might enforce "never discuss competitors."

---

**Q29.** How do NeMo Guardrails execution rails provide agentic security? Give a specific example of a dangerous agent action that execution rails would prevent.

**A29.** **Execution rails** intercept tool calls before they execute, validating the tool name, parameters, and context against configured policies. This provides agentic security by ensuring the agent cannot perform unauthorized or dangerous actions even if the LLM generates a malicious or erroneous tool call. Specific example: an agent has a `delete_record(record_id)` tool. Without execution rails, a prompt injection attack could cause the LLM to generate `delete_record("*")` to delete all records. An execution rail can: (1) validate that `record_id` matches an expected format (UUID, not wildcard); (2) check that the authenticated user has permission to delete the specified record; (3) require human approval for delete operations. The execution rail fires between the LLM generating the tool call and the tool actually executing, providing a security boundary.

---

### Human-AI (1 Question)

**Q30.** How does NAT support interactive agent workflows, and how would you implement a human approval gate using NAT and NeMo Guardrails together?

**A30.** NAT supports interactive workflows via **WebSocket-based bidirectional communication** through its FastAPI Server. This allows the agent to send intermediate results to the user and receive feedback before proceeding, rather than running autonomously to completion. To implement a human approval gate: (1) Configure the agent with a tool that represents the action requiring approval (e.g., `publish_document`); (2) Write a NeMo Guardrails **execution rail** in Colang 2.0 that intercepts calls to `publish_document`; (3) In the execution rail, the flow pauses execution, sends the pending action details to the user via WebSocket, and waits for a human response; (4) If the human approves, the execution rail allows the tool call to proceed; if the human rejects, the rail blocks the call and returns rejection feedback to the agent. This combines NAT's interactive infrastructure with Guardrails' policy enforcement to create a governed approval workflow.

---

## 5. Fifteen Scenario-Based Questions

**S1.** A financial services company wants to build an agent that answers questions about regulatory documents. The documents are updated quarterly. The agent must never hallucinate — every claim must be traceable to a specific document. Which NVIDIA architecture and tools should they use?

**Answer:** Use a **RAG architecture** based on the AI-Q Blueprint. NeMo Curator for document preprocessing (quarterly batch processing of updated documents). NIM embedding endpoint (Llama 3.2 NV EmbedQA) for embeddings. Milvus with cuVS for vector storage. NIM reranker (Llama 3.2 NV RerankQA) for high-precision retrieval. NIM LLM with a citation-requiring system prompt for generation. NeMo Guardrails with **fact-checking output rail** to verify every claim against retrieved passages and **topical rail** to restrict to regulatory topics only. NAT evaluation framework with faithfulness threshold >= 0.95. The quarterly update cycle means the ingestion pipeline runs as a batch job, not a streaming pipeline.

---

**S2.** A development team has built a LangChain agent and wants to add NVIDIA evaluation, guardrails, and production deployment without rewriting the agent. What is the fastest path?

**Answer:** Use the **NAT LangChain plugin** to wrap the existing LangChain agent into a NAT workflow. This gives them: (1) NAT evaluation framework to measure quality; (2) NeMo Guardrails integration via NAT middleware; (3) NAT FastAPI Server for production deployment; (4) NAT Profiler for performance analysis; (5) NAT observability for monitoring. No agent logic rewrite required — the plugin adapts the LangChain interface to NAT's workflow interface.

---

**S3.** An agent system is experiencing 12-second p95 latency. The NAT Profiler shows: retrieval 0.5s, reranking 0.8s, LLM generation 9s, guardrail checks 1.5s. Where should the team focus optimization?

**Answer:** Focus on **LLM generation (9s)**, which accounts for 75% of total latency. Options: (1) Switch to a smaller NIM model if quality allows (e.g., 8B instead of 70B); (2) Enable NVIDIA Dynamo prefix caching to avoid recomputing system prompt tokens; (3) Reduce output token limit if the current limit is excessive; (4) Add more NIM replicas if the 9s is due to queuing under load. Guardrail checks at 1.5s are a secondary target — check if safety NIMs (Content Safety, Jailbreak Detection) can be parallelized rather than sequential. Do NOT optimize retrieval or reranking — they are already fast.

---

**S4.** A team needs to deploy an agent that serves 500 concurrent users with a 70B parameter model. They have a budget of 8xH100 GPUs. Is this feasible?

**Answer:** Use the **NAT Sizing Calculator** to validate, but rough estimation: a 70B model on 2xH100 handles approximately 20-40 concurrent requests. With 8xH100, you can run 4 NIM instances, handling 80-160 concurrent users. 500 concurrent users is NOT feasible with 8xH100 for a 70B model. Options: (1) Use a smaller model (8B or 13B) that serves more concurrent requests per GPU — 8xH100 could handle 500+ concurrent users with an 8B model; (2) Increase GPU budget to 24-32xH100 for the 70B model; (3) Implement aggressive caching (NAT caching middleware) to reduce effective concurrent LLM requests; (4) Implement request queuing with appropriate timeout handling.

---

**S5.** A team wants to evaluate their RAG agent but does not have a labeled test dataset. What is the most practical approach using NVIDIA tools?

**Answer:** Use a **multi-step approach**: (1) Generate synthetic question-answer pairs from the corpus using an LLM — give the LLM a passage and ask it to generate questions that the passage answers. This creates a "synthetic test set." (2) Use **NAT evaluation framework with LLM-as-Judge** for metrics that do not require ground truth labels — faithfulness (does the answer match the retrieved context?) and relevance (does the answer address the question?) can be evaluated without human labels. (3) For retriever evaluation, use the synthetic Q&A pairs as pseudo-ground-truth for recall@k measurement. (4) Over time, collect real user queries and have domain experts label a subset for higher-quality evaluation. This bootstraps evaluation without the cold-start problem of needing a labeled dataset upfront.

---

**S6.** A healthcare company wants to build an agent that answers questions about medical procedures but must never provide medical advice. How should guardrails be configured?

**Answer:** Configure NeMo Guardrails with: (1) **Topical rail** — define "medical procedure information" as in-scope and "medical advice, diagnosis, treatment recommendation" as out-of-scope using Colang 2.0. (2) **Output rail** — add a custom output rail that checks whether the response contains directive language ("you should," "I recommend," "take this medication") and blocks or reformulates it to informational language. (3) **Input rail** — Content Safety NIM to block harmful medical queries. (4) **Dialog rail** — enforce a disclaimer ("This information is for educational purposes only, not medical advice") at the start of every conversation and periodically within long conversations. (5) **Execution rail** — if the agent has tools, ensure no tool can prescribe, diagnose, or recommend treatment.

---

**S7.** A team is choosing between ReAct and ReWOO for a research agent that searches multiple databases. The databases are unreliable — about 10% of queries fail. Which agent type is better?

**Answer:** **ReAct** is better for this scenario. ReWOO generates the entire plan upfront and executes it — if a database query fails, the plan breaks and there is no mechanism to adapt. ReAct reasons after each step, so when a database query fails, the agent can: (1) retry the query; (2) reformulate the query; (3) search a different database; (4) adjust its plan based on partial results. The 10% failure rate means approximately 1 in 10 tool calls will fail, which is too frequent for a rigid plan. The tradeoff is that ReAct uses more LLM calls (one per step), but the reliability improvement justifies the cost.

---

**S8.** A company has 5 million documents to index for RAG. Their current CPU-based preprocessing pipeline takes 72 hours. How can NVIDIA tools reduce this?

**Answer:** Use **NeMo Curator** for GPU-accelerated preprocessing. NeMo Curator provides GPU-accelerated: deduplication (fuzzy dedup via MinHash LSH on GPU), language filtering, quality scoring, text extraction, and cleaning. Typical speedup is 10-100x over CPU processing, reducing 72 hours to 1-7 hours. For embedding, use **NIM embedding endpoints** with batch processing — GPU inference for embeddings is far faster than CPU. For indexing, use **Milvus with cuVS** for GPU-accelerated index building. The combination of NeMo Curator (preprocessing) + NIM (embedding) + Milvus/cuVS (indexing) creates an end-to-end GPU-accelerated ingestion pipeline.

---

**S9.** A multi-agent system has a Router Agent that dispatches to 4 specialist agents. The team discovers that 90% of requests go to only one specialist. What should they do?

**Answer:** This is a scaling and architecture question. Actions: (1) **Scale the popular specialist** — give it more replicas while keeping other specialists at minimum replicas. This is the advantage of service separation via NAT A2A — each agent scales independently. (2) **Verify the router is correct** — use NAT trajectory evaluation to confirm the 90/10 split reflects actual request distribution, not a routing bug. Test with labeled requests to measure routing accuracy. (3) **Consider eliminating the router** — if 90% of requests go to one specialist, the routing overhead may not be justified. Consider making the popular specialist the default handler with a lightweight classifier that redirects the 10% of edge cases. (4) **Use NAT Profiler** to measure routing overhead — if the router adds significant latency for 90% of requests that all go to the same place, optimizing the routing decision (caching, simpler classifier) is worthwhile.

---

**S10.** A team needs to implement PII redaction in their agent's outputs. NeMo Guardrails supports multiple PII detection backends. Which should they choose?

**Answer:** NeMo Guardrails integrates with three PII detection backends: **GLiNER**, **Presidio**, and **PrivateAI**. The choice depends on requirements: (1) **Presidio** (Microsoft) — best for general-purpose PII detection with strong multi-language support, rule-based + ML hybrid approach, fully open source. Choose for most production deployments. (2) **GLiNER** — NER-based detection, good for custom entity types beyond standard PII categories. Choose when you need to detect domain-specific sensitive entities (e.g., internal project code names). (3) **PrivateAI** — cloud-based service with highest accuracy for complex PII patterns but adds external API dependency. Choose when accuracy is paramount and cloud dependency is acceptable. For most NVIDIA-stack deployments, **Presidio** is the pragmatic default — it runs locally, has broad entity coverage, and integrates cleanly with the NeMo Guardrails redaction processor.

---

**S11.** A team deployed their agent last month. Usage data shows that 15% of user queries result in "I don't know" responses. The team suspects the knowledge base covers those topics. How should they diagnose and fix this?

**Answer:** Diagnostic steps using NVIDIA tools: (1) **Collect the failing queries** from NAT observability logs — identify the 15% of queries producing "I don't know." (2) **Run retriever evaluation** — for each failing query, check what the retriever returned. If retrieval returns empty or irrelevant results, the problem is retrieval (embedding quality, index coverage, or chunking). (3) **Check chunking** — if relevant documents exist but are not retrieved, the chunking strategy may have split the answer across chunks. Try larger chunks or document-structure-aware chunking. (4) **Check embedding coverage** — if the query uses terminology different from the documents, the embedding model may not bridge the semantic gap. Consider hybrid retrieval (add BM25 for keyword matching). (5) **Run NAT evaluation framework** on the failing queries to measure context recall — this tells you whether the right information was retrievable. (6) Feed fixed examples into the **Data Flywheel** cycle for continuous improvement.

---

**S12.** A team wants to deploy agents with NIM on Kubernetes but does not want to manually manage GPU allocation, model caching, and health monitoring. What should they use?

**Answer:** Use the **NIM Operator for Kubernetes**. The NIM Operator automates: GPU resource discovery and allocation (no manual toleration/affinity configuration), model caching via PersistentVolumeClaims (models persist across pod restarts, avoiding re-download), health monitoring specific to LLM serving (model loading status, inference readiness, not just TCP health), and scaling based on inference-specific metrics (queue depth, GPU utilization). Deploy the NIM Operator to the cluster, then define NIM deployments as custom resources — the operator handles the operational complexity.

---

**S13.** An agent occasionally generates tool calls with invalid parameters, causing downstream service errors. How should the team prevent this?

**Answer:** Implement **NeMo Guardrails execution rails**. Configure an execution rail in Colang 2.0 that intercepts every tool call before execution and validates: (1) The tool name is in the authorized tool list; (2) All required parameters are present; (3) Parameter values match expected types and formats (e.g., IDs match UUID pattern, dates are valid); (4) Parameter values are within acceptable ranges. Additionally, use **NAT's structured output support** to constrain the LLM's tool call generation — define JSON schemas for each tool's parameters so the LLM generates well-formed calls. The combination of structured outputs (generation-time constraint) and execution rails (pre-execution validation) provides defense in depth.

---

**S14.** A team is building a conversational agent that needs to remember user preferences across sessions (preferred language, communication style, previous topics discussed). Which NAT memory configuration should they use?

**Answer:** Use **NAT Memory with Redis Object Store** and the **Automatic Memory Wrapper**. Redis provides: persistence across sessions (survives container restarts), low-latency reads (< 1ms for memory retrieval), and TTL support (can expire old memories). The Automatic Memory Wrapper handles the mechanics — it stores relevant interaction details after each conversation turn and retrieves relevant memories to inject into context at the start of each turn. Configure the wrapper to prioritize: user preferences (highest retention), recent conversation context (medium retention), and older topics (lowest retention, subject to TTL expiration). If the team needs queryable audit trails of all memory operations, add MySQL as a secondary store for logging.

---

**S15.** A team has a working agent but wants to systematically improve its performance. They do not know whether to optimize the prompt, the retrieval, or the model. What NVIDIA tools help them decide?

**Answer:** Use a three-step diagnostic: (1) **NAT Profiler** — identify where time and tokens are being spent. If retrieval is 60% of latency, optimize retrieval first. If generation dominates, optimize the model or prompt. (2) **NAT Evaluation Framework** — measure faithfulness, relevance, context precision, context recall. Low context recall = retrieval problem. Low faithfulness with good context = generation/prompt problem. (3) Based on findings: if retrieval needs improvement, try better chunking, hybrid retrieval, or a different reranker. If generation needs improvement, use the **NAT Agent Hyperparameter Optimizer** to systematically test prompt variants and temperature settings. If model quality is the bottleneck, consider the **Data Flywheel Blueprint** — collect failure cases, curate them, and fine-tune with NeMo Customizer. The key principle: diagnose with Profiler + Evaluation Framework before optimizing anything.

---

## 6. Ten Architecture Comparison Questions

**A1.** You are designing a RAG system. Compare: (a) embedding-only retrieval (single-stage) vs. (b) embedding + reranking (two-stage). When is the extra reranking stage NOT worth the latency cost?

**Answer:** Two-stage (embedding + reranking) is better in almost all production scenarios. The reranker (cross-encoder) captures query-document interactions that the embedding model (bi-encoder) misses, typically improving precision by 10-20%. However, reranking is NOT worth it when: (1) the corpus is very small (< 1,000 documents) and the embedding model already achieves near-perfect recall — the reranker has little to improve; (2) latency budget is extremely tight (< 500ms total) and the reranker adds 200-400ms; (3) all queries are simple keyword lookups where semantic similarity is not needed (rare in production). For the NCP-AAI exam: the AI-Q Blueprint uses two-stage retrieval, establishing it as NVIDIA's recommended approach.

---

**A2.** Compare: (a) a single NAT Tool Calling Agent with 20 tools vs. (b) a NAT Router Agent dispatching to 4 specialized agents, each with 5 tools. Which is better, and at what tool count does the switch become necessary?

**Answer:** The Router architecture (b) is better at 20 tools. Empirically, LLM tool selection accuracy degrades as tool count increases — at 10+ tools, the LLM may select the wrong tool 15-25% of the time because tool descriptions compete for attention in the context window. The Router pattern reduces this by having the Router make a coarse classification (4 categories), then each specialist selects from only 5 tools. The switch becomes necessary at roughly **8-12 tools**, depending on tool description similarity. If tools are very distinct (e.g., "send email" vs. "query database"), a single agent handles more tools. If tools are similar (e.g., 5 different search APIs), the single agent confuses them earlier. Additional benefit of the Router: specialists can have optimized system prompts, and each can be evaluated and scaled independently.

---

**A3.** Compare: (a) deploying NIM models on bare-metal Kubernetes with manual YAML vs. (b) using the NIM Operator. When would you choose bare-metal over the Operator?

**Answer:** The NIM Operator is better for nearly all production deployments — it automates GPU allocation, model caching, health monitoring, and scaling. Choose bare-metal K8s YAML when: (1) the K8s cluster has a custom GPU scheduler (e.g., a multi-tenant cluster with a proprietary resource allocator) that conflicts with the NIM Operator's GPU management; (2) the team needs to deploy models not yet supported by the NIM Operator; (3) the organization has strict compliance requirements that prohibit third-party operators from managing GPU resources; (4) the deployment is a one-off research experiment where operational automation is not needed. For NCP-AAI: know that the NIM Operator is the recommended approach and understand what it automates.

---

**A4.** Compare: (a) NAT caching middleware vs. (b) no caching with more NIM replicas. When is caching more cost-effective than scaling?

**Answer:** Caching is more cost-effective when the query distribution has high repetition — a small set of queries accounts for a large fraction of traffic (Zipf distribution). If 30% of queries are repeats, caching eliminates 30% of LLM calls, equivalent to removing the need for 30% of NIM replicas. Each NIM replica for a 70B model requires 2xH100 (~$30K/year in cloud cost), so caching that saves one replica is extremely cost-effective. Caching is NOT effective when: (1) queries are highly unique (research tools, creative writing); (2) answers must be dynamic (incorporating real-time data); (3) the agent's context includes per-user state that makes "identical queries" rare. Use semantic caching (provided by NAT caching middleware) rather than exact-match caching to increase hit rates.

---

**A5.** Compare: (a) CPU-based Milvus (HNSW index) vs. (b) GPU-accelerated Milvus (cuVS). When does the GPU cost pay for itself?

**Answer:** cuVS pays for itself when concurrent query load is high. CPU HNSW handles single-digit QPS efficiently (< 10ms per query) and requires no GPU budget. At 50-100+ concurrent queries, CPU HNSW queues requests and latency degrades linearly, while cuVS maintains low latency through GPU parallelism. The GPU cost pays for itself when: (1) concurrent search QPS > 50 and latency SLA is < 50ms; (2) the index is very large (100M+ vectors) where even single queries are slower on CPU; (3) the deployment already has GPU infrastructure for NIM, so the marginal cost of allocating one GPU to Milvus is lower than adding CPU nodes. For development, testing, and low-traffic production, CPU HNSW is sufficient. For high-traffic production with latency requirements, cuVS is the right choice.

---

**A6.** Compare: (a) NeMo Guardrails with Safety NIMs (Content Safety, Jailbreak Detection) vs. (b) NeMo Guardrails with Colang-only rules (no Safety NIMs). When are Safety NIMs necessary?

**Answer:** Colang-only rules are pattern-based — they match user inputs against defined patterns and templates. They work well for topical control, dialog flow enforcement, and simple input filtering. Safety NIMs are ML models trained specifically for content safety and jailbreak detection — they catch attacks that pattern-matching cannot (novel phrasings, encoding tricks, multi-turn manipulation). Safety NIMs are necessary when: (1) the agent faces adversarial users (public-facing systems, systems where prompt injection is a real threat); (2) compliance requires state-of-the-art safety measures; (3) the attack surface is broad (the agent handles free-form text from untrusted users). Colang-only is sufficient when: (1) the agent is internal-only with trusted users; (2) the agent handles structured inputs with limited free-text; (3) cost constraints prevent deploying additional NIM endpoints. For production public-facing agents, Safety NIMs + Colang rules (defense in depth) is the recommended approach.

---

**A7.** Compare: (a) NAT FastAPI Server for all agents in a multi-agent system vs. (b) NAT FastAPI Server for the gateway + NAT A2A Server for inter-agent communication. Why not use FastAPI for everything?

**Answer:** FastAPI provides REST/WebSocket endpoints designed for external clients (humans, applications). A2A provides a structured agent-to-agent protocol with typed messages, task delegation, status tracking, and agent discovery. While you CAN have agents call each other's REST APIs, you lose: (1) **Typed inter-agent contracts** — A2A defines message schemas for agent communication; REST requires custom schema design per endpoint; (2) **Task lifecycle management** — A2A tracks task status (pending, running, complete, failed) natively; REST requires custom implementation; (3) **Agent discovery** — A2A supports dynamic agent registration and discovery; REST requires hardcoded URLs; (4) **Trace correlation** — A2A automatically propagates trace context; REST requires manual header propagation. Use FastAPI for the gateway (external clients speak HTTP) and A2A for internal agent communication (agents speak a protocol designed for agents).

---

**A8.** Compare: (a) a single large NIM model (70B) for all agents vs. (b) different NIM models per agent (8B for simple tasks, 70B for complex tasks). When is the mixed approach better?

**Answer:** Mixed models are better when agents have different complexity requirements. A Router Agent that classifies requests needs only an 8B model (classification is a simple task). A Code Analysis Agent that identifies subtle security vulnerabilities benefits from a 70B model (nuanced reasoning). Using 70B for the Router wastes GPU and adds latency. Benefits of mixed: (1) **Cost reduction** — 8B model uses ~10x fewer GPU resources than 70B; (2) **Latency reduction** — 8B model generates tokens faster; (3) **Scaling flexibility** — can add 8B replicas cheaply for the high-traffic Router while keeping fewer expensive 70B replicas for complex agents. Mixed is NOT better when: (1) all agents perform comparably complex tasks; (2) managing multiple NIM endpoints adds operational complexity the team cannot handle; (3) prompt tuning per model creates maintenance burden. Use NAT Profiler to identify which agents actually need 70B quality.

---

**A9.** Compare: (a) synchronous agent execution (user waits for complete response) vs. (b) NAT interactive workflow (WebSocket, streaming intermediate results). When is synchronous sufficient?

**Answer:** Synchronous is sufficient when: (1) total latency is under 5 seconds — users tolerate a brief wait; (2) intermediate results are not meaningful (e.g., a simple Q&A agent where only the final answer matters); (3) the client is another system (API-to-API) that processes the final result programmatically. Interactive/WebSocket is necessary when: (1) total latency exceeds 10 seconds — users need feedback that the system is working (progress indicators, intermediate results); (2) the task involves human-in-the-loop decisions mid-execution (approval gates); (3) the agent performs multi-step work where intermediate results are valuable (showing retrieved documents before generating the answer); (4) the system needs bidirectional communication (user can cancel, redirect, or provide additional input mid-execution). For most production multi-step agents, interactive workflows provide a better user experience.

---

**A10.** Compare: (a) running evaluation only at deployment time vs. (b) implementing the Data Flywheel Blueprint for continuous evaluation. When is deployment-time evaluation sufficient?

**Answer:** Deployment-time-only evaluation is sufficient when: (1) the knowledge base is static (never updated); (2) the user population and query distribution are stable; (3) the LLM model is fixed (no fine-tuning or model swaps); (4) the system is internal with limited usage where manual monitoring catches issues. Continuous evaluation (Data Flywheel) is necessary when: (1) the knowledge base is updated regularly (new documents change what the agent should know); (2) query patterns shift over time (new topics, new user populations); (3) the team plans to fine-tune or swap models (need to measure impact); (4) the system is customer-facing with high stakes (wrong answers have business impact); (5) the team wants to systematically improve quality over time rather than maintaining a static baseline. For most production systems, continuous evaluation catches drift that deployment-time evaluation cannot detect.
