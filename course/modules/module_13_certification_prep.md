# Module 13: Certification Prep and Full-Stack Integration

## A. Module Title

**Certification Prep and Full-Stack Integration**

Primary Exam Domain: All domains (100%)

This is a review and integration module. It introduces no new concepts. Its purpose is to connect the knowledge from Modules 1-12, identify weak areas, and prepare you for exam conditions.

---

## B. Why This Module Matters

The NCP-AAI exam tests integrated understanding, not isolated facts. A question about guardrails may require deployment knowledge. A question about memory may require evaluation knowledge. A question about tool use may require oversight knowledge. If you studied each module in isolation, you can answer questions about that module. If you studied them as an integrated system, you can answer questions that cross module boundaries — and those are the harder questions that separate passing from failing.

This module also addresses a practical reality: the exam has 60-70 questions in 120 minutes. That is approximately 100 seconds per question. You do not have time to reason from first principles on every question. You need pattern recognition — instantly mapping a question to the right concept, the right NVIDIA tool, and the right architectural decision. This module builds that pattern recognition through integration exercises and practice questions.

---

## C. Learning Objectives

1. Complete a full-stack agent system design exercise that touches all 10 exam domains
2. Map every exam domain to specific NVIDIA tools and course modules
3. Identify personal weak areas using a self-assessment checklist and remediate them
4. Answer practice questions under timed conditions (100 seconds per question)
5. Recognize exam question patterns and apply elimination strategies

---

## D. Required Concepts

All content from Modules 1-12. If any module feels unfamiliar, revisit it before proceeding.

---

## E. Core Lesson Content

### Domain-by-Domain Review

**1. Agentic AI Fundamentals (~10%)**
Covered in: Modules 1-2.
Key NVIDIA tools: NAT (NVIDIA Agent Toolkit) as the primary agent framework.
Summary: Understand what makes a system "agentic" (goal-directed, tool-using, autonomous decision-making), the distinction between single-agent and multi-agent architectures, and the spectrum from simple chatbots to fully autonomous agents.
Likely exam angles: (1) Identifying which scenarios require agentic AI vs. traditional ML pipelines, (2) Defining core properties of agentic systems, (3) Distinguishing agent types.

**2. Agent Architectures and Design Patterns (~12%)**
Covered in: Modules 3-5.
Key NVIDIA tools: NAT agent types (SingleAgent, GraphAgent, pipeline patterns), NAT workflow definitions.
Summary: ReAct pattern, tool-augmented generation, multi-agent architectures (supervisor, peer-to-peer, hierarchical), workflow vs. autonomous agents, state management patterns.
Likely exam angles: (1) Selecting the right agent architecture for a scenario, (2) Identifying when multi-agent is needed vs. single-agent, (3) Tradeoffs between workflow rigidity and agent autonomy.

**3. Foundation Models and Prompt Engineering (~10%)**
Covered in: Module 2.
Key NVIDIA tools: NIM for model serving, NeMo framework for model customization.
Summary: How agents use LLMs (planning, reasoning, tool selection), prompt engineering for agents (system prompts, few-shot examples, chain-of-thought), model selection criteria (capability vs. latency vs. cost).
Likely exam angles: (1) Prompt design for tool-calling agents, (2) Model selection tradeoffs, (3) How context window limits affect agent design.

**4. Tool Use and Function Calling (~15%)**
Covered in: Modules 4-5.
Key NVIDIA tools: NAT tool registration, tool schemas (JSON), NAT per-user functions, NeMo Guardrails execution rails for tool validation.
Summary: Tool design principles, function calling protocols, error handling, tool result injection into context, tool composition, API integration patterns.
Likely exam angles: (1) Designing tool schemas for agents, (2) Handling tool failures gracefully, (3) Security considerations for tool execution.

**5. Retrieval-Augmented Generation (RAG) (~12%)**
Covered in: Module 6.
Key NVIDIA tools: NVIDIA RAG pipelines, NeMo Retriever, NIM for embedding models, vector database integration.
Summary: RAG architecture, chunking strategies, embedding models, retrieval methods (dense, sparse, hybrid), reranking, context injection, evaluation of retrieval quality.
Likely exam angles: (1) Choosing chunking strategies for different document types, (2) When to use hybrid retrieval, (3) RAG failure modes and diagnostics.

**6. Memory and State Management (~8%)**
Covered in: Module 7.
Key NVIDIA tools: NAT memory components (short-term, long-term, entity), NAT state persistence.
Summary: Memory types, context window management, summarization strategies, entity extraction, cross-session persistence, memory in multi-agent systems.
Likely exam angles: (1) Selecting memory type for a scenario, (2) Managing context window overflow, (3) State persistence across agent restarts.

**7. Safety, Guardrails, and Responsible AI (~10%)**
Covered in: Module 8.
Key NVIDIA tools: NeMo Guardrails (Colang 2.0), input/output/dialog/execution rails, topical rails, content moderation.
Summary: Rail types and purposes, Colang configuration, guardrail architecture (input rail → LLM → output rail), jailbreak prevention, content filtering, topic restriction, PII handling.
Likely exam angles: (1) Configuring Colang rails for specific safety requirements, (2) Rail execution order, (3) Balancing safety with usability.

**8. Evaluation and Testing (~10%)**
Covered in: Module 9.
Key NVIDIA tools: NAT evaluation framework, eval datasets, metrics (correctness, faithfulness, relevance), AgentIQ benchmarking.
Summary: What to evaluate (component vs. end-to-end), evaluation metrics for agents, creating eval datasets, regression testing, LLM-as-judge patterns, benchmarking methodology.
Likely exam angles: (1) Selecting appropriate evaluation metrics, (2) Designing evaluation datasets, (3) Interpreting evaluation results.

**9. Deployment and Scaling (~13%)**
Covered in: Module 10.
Key NVIDIA tools: NAT deployment servers (FastAPI, MCP, FastMCP, A2A), NIM microservices, NIM Operator, NVIDIA Dynamo, Helm charts.
Summary: NAT deployment server selection, NAT + NIM architecture separation, async job management, containerization, GPU resource planning, autoscaling for variable-length agents, Kubernetes deployment.
Likely exam angles: (1) Selecting the right deployment server, (2) NAT vs. NIM roles, (3) Scaling strategies for agent workloads.

**10. Run, Monitor, Maintain + Human-AI Oversight (~10% combined)**
Covered in: Modules 11-12.
Key NVIDIA tools: NAT observability (Phoenix/Weave/Langfuse), OpenTelemetry, redaction processors, NAT interactive workflows, NeMo Guardrails execution rails for approval, per-user functions, middleware.
Summary: Agent-specific observability challenges, trace interpretation, debugging failure patterns, monitoring metrics, human oversight spectrum, approval workflows, audit trails, data flywheel.
Likely exam angles: (1) Debugging agent failures from traces, (2) Selecting observability tools, (3) Implementing human approval for high-risk actions.

### Integration Exercise

**Design a complete agent system for an enterprise IT helpdesk.**

Requirements:
- Handles password resets, software installation requests, network troubleshooting, and hardware requests
- Serves 200 employees, peak 50 concurrent users
- Must comply with SOC 2 audit requirements
- Must escalate to human IT staff for actions that modify Active Directory

Walk through every domain:

| Domain | Decision | Justification |
|--------|----------|---------------|
| Architecture | SingleAgent with tool routing | Tasks are well-defined; multi-agent adds unnecessary complexity |
| Foundation Model | 70B instruction-tuned via NIM | Needs strong reasoning for troubleshooting; NIM provides optimized serving |
| Tool Use | 8 tools: password_reset, install_software, check_network, create_ticket, lookup_kb, send_notification, modify_ad, escalate | Each maps to a specific IT action |
| RAG | Knowledge base of IT documentation, chunked by procedure | Agents need to reference SOPs for troubleshooting steps |
| Memory | Short-term (conversation) + entity (user, device) | Track the user's issue context; remember device history |
| Guardrails | Input: topic restriction (IT only), Output: PII filter, Execution: gate modify_ad | Prevent off-topic use; protect PII; require approval for AD changes |
| Evaluation | Weekly eval on 100 resolved tickets, metrics: resolution rate, accuracy, escalation rate | Continuous quality monitoring |
| Deployment | NAT FastAPI Server + NIM (2x H100), K8s with Helm | Standard production deployment with GPU inference |
| Monitoring | Phoenix for tracing, dashboard for resolution rate and escalation rate | Operational visibility |
| Oversight | modify_ad requires manager approval via WebSocket; all actions logged via middleware | SOC 2 compliance; risk-proportional oversight |

### Weak-Area Remediation

Based on source coverage confidence across the course, these three areas are most likely to have gaps:

**Human-AI Oversight (65% source confidence)**: This domain relies most heavily on inferred patterns. Focus your study on the NVIDIA-documented primitives: WebSocket interactive workflows, execution rails for approval, per-user functions, and middleware. For design patterns (escalation paths, approval queues, feedback loops), understand the concepts but know they are general practice, not NVIDIA-prescribed.

**Deployment and Scaling (75% source confidence)**: The NAT deployment servers and NIM are well-documented. NVIDIA Dynamo's agent-specific documentation is thin. Focus on the NAT + NIM architecture separation, deployment server selection, and async job management — these are concrete and testable.

**Evaluation Methodology (80% source confidence)**: NAT provides an evaluation framework, but specific metric implementations may evolve. Focus on what to evaluate (component vs. end-to-end), which metrics matter for which scenarios, and how evaluation connects to the data flywheel.

### Exam Strategy

**Time management**: 120 minutes / 65 questions (midpoint estimate) = ~110 seconds per question.
- First pass: answer every question you are confident about. Flag uncertain ones.
- Second pass: return to flagged questions with remaining time.
- Do not spend more than 3 minutes on any single question on first pass.

**Elimination strategy**: For "which NVIDIA tool" questions:
- Eliminate tools from the wrong layer (NIM is inference, not orchestration; NAT is orchestration, not inference)
- Eliminate tools from the wrong lifecycle phase (evaluation tools are not deployment tools)
- If two options seem equally valid, prefer the one that is more specifically designed for the scenario

**"Which NVIDIA tool" question pattern**: These questions give a scenario and ask which NVIDIA tool or component is appropriate. Key mappings to memorize:

| Scenario | Tool |
|----------|------|
| Agent orchestration and workflow | NAT |
| LLM inference serving | NIM |
| K8s-automated NIM deployment | NIM Operator |
| Safety rails and content filtering | NeMo Guardrails |
| Model optimization for inference | NVIDIA Dynamo / TensorRT-LLM |
| Agent-as-MCP-tool | NAT MCP Server |
| Multi-agent distributed deployment | NAT A2A Server |
| Human approval for tool calls | NeMo Guardrails execution rails |
| Trace visualization | Phoenix / Weave / Langfuse |
| Sensitive data in telemetry | NAT redaction processors |

### Blueprint-Aligned Revision

Review each NVIDIA Blueprint architecture you encountered in the course:
- **AI-Q Blueprint**: Multi-agent enterprise Q&A. Key takeaway: 2x H100 minimum, Docker Compose reference, multi-model orchestration pattern. Always verify hardware requirements on the specific Blueprint page or repository before attempting full deployment. Requirements vary significantly across Blueprints.
- Verify you can trace a request through the Blueprint: user query → NAT orchestration → tool calls → NIM inference → guardrail check → response.

### Topic Heatmap

```
Domain Weight and NVIDIA-Specificity
========================================================
                        Low NVIDIA-Specific    High NVIDIA-Specific
                        Content                Content
High Weight (>10%)  |   RAG (12%)             Tool Use (15%)
                    |                          Architecture (12%)
                    |                          Deployment (13%)
                    |
Medium Weight       |   Fundamentals (10%)     Safety/Guardrails (10%)
(8-10%)             |   Foundation Models(10%) Evaluation (10%)
                    |   Memory (8%)
                    |
Low Weight (<8%)    |   Human Oversight (5%)
                    |   Monitoring (5%)
========================================================
```

Prioritize study time on high-weight, high-NVIDIA-specificity topics: Tool Use, Architecture, Deployment, and Safety/Guardrails. These have the most questions and the most NVIDIA-specific content that you cannot learn from general AI knowledge.

---

## F. Terminology Box

No new terms are introduced in this module. Refer to terminology boxes in Modules 1-12 for comprehensive definitions.

---

## G. Common Misconceptions

1. **"I only need to know NVIDIA tools to pass."** The exam tests architectural reasoning, not just tool naming. You must understand when and why to use each tool, not just what it is.

2. **"The low-weight domains (5% each) aren't worth studying."** Two 5% domains = 10% of the exam. That is 6-7 questions. Missing all of them can be the difference between passing and failing.

3. **"I can rely on general AI/ML knowledge for agent-specific questions."** Agent systems have specific architectural patterns (ReAct, tool augmentation, multi-agent coordination) that differ from traditional ML systems. General knowledge is necessary but not sufficient.

4. **"Longer answers are better on the exam."** The exam is multiple choice. There are no points for elaboration. Speed and accuracy matter. Practice identifying the key discriminator in each question.

5. **"If I know the theory, I'll recognize the right answer."** Many questions present scenarios where multiple answers seem plausible. The discriminator is often which tool is specifically designed for the scenario vs. which could technically work. Practice with scenario-based questions.

---

## H. Failure Modes / Anti-Patterns

1. **Studying modules in isolation**: The exam tests integration. After reviewing each module, practice connecting it to adjacent modules.

2. **Memorizing tool names without understanding roles**: Know what each tool does, when to use it, and what it does NOT do. The exam tests selection, not recall.

3. **Skipping hands-on labs**: Practical experience creates durable memory. If you only read the content, you will struggle with scenario-based questions that require applied understanding.

4. **Ignoring the inference disclaimers**: Some course content is inferred from general practice. The exam tests NVIDIA-documented features. When in doubt, choose the answer that aligns with a specific NVIDIA primitive over a general best practice.

5. **Cramming the night before**: The certification covers 10 domains. Spaced review over 2-3 weeks is significantly more effective than a single marathon session.

---

## I. Hands-On Lab

**Lab 13: Full-Stack Agent System Design Review**

Take the IT helpdesk agent design from the integration exercise in Section E. Verify each decision by referencing the relevant module. For each domain, write one paragraph explaining: (a) what you chose, (b) why, (c) what alternative you considered and rejected, and (d) which NVIDIA tool implements your choice. The goal is to produce a complete design document that demonstrates integrated understanding.

---

## J. Stretch Lab

**Stretch Lab 13: Timed Mock Exam Simulation**

Take the 20-question mini mock exam (Section K) under timed conditions: 33 minutes, no notes. Score yourself. For each incorrect answer, trace back to the relevant module and re-study that section. Then take the exam again after 48 hours and compare scores.

---

## K. Review Quiz — Mini Mock Exam (20 Questions)

**DISCLAIMER**: All practice questions below are inferred from exam domain descriptions and competency statements. NVIDIA does not publish sample questions. These questions are designed to approximate exam style and difficulty but are not official practice materials.

**Q1.** An agent needs to search a knowledge base, synthesize results, and send an email to the user. The email tool occasionally fails due to SMTP server issues. What should the agent architecture include?
- A) Retry logic in the email tool with exponential backoff
- B) A separate email-sending agent
- C) Removing the email tool and asking the user to send it themselves
- D) Ignoring the failure and reporting success

**Answer: A** — Tool-level retry with backoff is the standard pattern for transient failures. A separate agent is over-engineering for a single tool failure.

**Q2.** Which NAT deployment server should you use when your agent must be callable as a tool by other MCP-compatible agents?
- A) FastAPI Server
- B) A2A Server
- C) MCP Server
- D) Console Frontend

**Answer: C** — MCP Server exposes the agent as an MCP-compatible tool.

**Q3.** An agent's context window is filling up after 8 tool calls. What memory strategy addresses this?
- A) Switch to a model with a larger context window
- B) Implement conversation summarization to compress earlier context
- C) Remove all tool results from context
- D) Restart the agent after every 5 tool calls

**Answer: B** — Summarization compresses earlier context while preserving key information. Larger windows delay the problem without solving it.

**Q4.** What is the primary purpose of NeMo Guardrails output rails?
- A) To speed up LLM inference
- B) To filter or modify the LLM's response before it reaches the user
- C) To manage GPU memory allocation
- D) To route requests to different NIM instances

**Answer: B** — Output rails validate and filter LLM responses for safety, accuracy, and policy compliance.

**Q5.** In the NAT + NIM architecture, which component handles agent orchestration logic?
- A) NIM
- B) NAT
- C) NVIDIA Dynamo
- D) NIM Operator

**Answer: B** — NAT handles orchestration (agent logic, tool routing, state). NIM handles inference.

**Q6.** An evaluation shows that an agent correctly answers 90% of questions but the answers contain hallucinated citations 15% of the time. Which metric captures this issue?
- A) Correctness
- B) Faithfulness
- C) Relevance
- D) Latency

**Answer: B** — Faithfulness measures whether the response is grounded in the retrieved/provided context. Hallucinated citations indicate low faithfulness.

**Q7.** Which observability platform integrated with NAT is open-source and self-hosted?
- A) Weave
- B) Langfuse (cloud only)
- C) Phoenix
- D) Datadog

**Answer: C** — Phoenix (Arize) is open-source and self-hosted. Langfuse is also open-source but the question specifies both properties; Phoenix is the canonical NAT integration for self-hosted.

**Q8.** A production agent occasionally enters an infinite loop, calling the same tool repeatedly. Which monitoring metric would detect this?
- A) Token cost per request
- B) Average steps per execution (sudden spike)
- C) HTTP error rate
- D) CPU utilization

**Answer: B** — A sudden increase in steps per execution indicates looping behavior. Token cost would also increase but is a secondary indicator.

**Q9.** An agent handles both password resets (low risk) and Active Directory modifications (high risk). What is the appropriate oversight architecture?
- A) Human approval for all actions
- B) No oversight — the agent is trained to be careful
- C) Automated execution for password resets; human approval via execution rail for AD modifications
- D) Log everything but approve nothing

**Answer: C** — Risk-proportional oversight: automate low-risk actions, gate high-risk actions on human approval using NeMo Guardrails execution rails.

**Q10.** What is the minimum GPU configuration documented for the AI-Q Blueprint?
- A) 1x A100 40GB
- B) 2x H100 80GB
- C) 4x A100 40GB
- D) 1x H100 80GB

**Answer: B** — The AI-Q Blueprint documents 2x H100 80GB minimum (or 3x A100 80GB alternative). Always verify hardware requirements on the specific Blueprint page or repository before attempting full deployment. Requirements vary significantly across Blueprints.

**Q11.** An agent uses RAG to answer questions about company policies. Users report outdated answers. What is the most likely cause?
- A) The LLM is outdated
- B) The RAG index has not been refreshed with recent policy updates
- C) The embedding model is too small
- D) The chunking strategy is wrong

**Answer: B** — Stale RAG indexes are the most common cause of outdated answers. Index freshness maintenance is an operational requirement.

**Q12.** Which NAT feature ensures that telemetry data does not contain personal information?
- A) Per-user functions
- B) Redaction processors
- C) Execution rails
- D) Middleware logging

**Answer: B** — NAT redaction processors strip sensitive data from telemetry before export.

**Q13.** A multi-agent system has a supervisor agent and three specialist agents. The supervisor routes tasks to specialists. What NAT deployment pattern supports this?
- A) All agents in one FastAPI Server
- B) Each agent as a separate A2A Server
- C) Each agent as a separate MCP Server
- D) All agents in one Console Frontend

**Answer: B** — A2A Server supports distributed multi-agent execution where each agent is an independent service.

**Q14.** What is the purpose of NAT's `nat_test_llm`?
- A) To benchmark LLM performance
- B) To provide a deterministic mock LLM for replay testing and debugging
- C) To test LLM fine-tuning quality
- D) To serve as a backup LLM when NIM is unavailable

**Answer: B** — `nat_test_llm` replaces the real LLM with a deterministic mock for reproducible debugging.

**Q15.** An agent's p99 latency is 45 seconds but p50 is 3 seconds. What does this indicate?
- A) The system is broken
- B) A small percentage of requests trigger complex, multi-step agent executions
- C) The LLM is consistently slow
- D) Network latency is unstable

**Answer: B** — High p99 with low p50 indicates a heavy tail — most requests are fast, but some complex queries take much longer. This is normal for agents but must be managed.

**Q16.** Which Colang construct in NeMo Guardrails would restrict an agent to only discuss IT support topics?
- A) Output rail
- B) Topical rail (dialog rail)
- C) Execution rail
- D) Input sanitization

**Answer: B** — Topical rails (a type of dialog rail) restrict the conversation to specified topics.

**Q17.** Why is request-count-based autoscaling insufficient for agent endpoints?
- A) Agents don't receive requests
- B) Agent requests have variable resource consumption; one complex request may use 100x the resources of a simple one
- C) Kubernetes doesn't support request-count scaling
- D) Agents only use CPU, not GPU

**Answer: B** — Variable-length execution means request count does not correlate with resource consumption. Scale on token throughput or GPU queue depth instead.

**Q18.** What does the data flywheel concept describe?
- A) A hardware component in GPU clusters
- B) A continuous cycle of collecting production data, evaluating quality, improving the system, and redeploying
- C) A data compression technique
- D) A method for rotating encryption keys

**Answer: B** — The data flywheel is the continuous improvement loop: collect → evaluate → improve → deploy → repeat.

**Q19.** An execution rail intercepts a tool call and determines it requires human approval. Using NAT, how is the approval request delivered to the human?
- A) Email notification
- B) WebSocket interactive workflow sends a real-time message to the connected client
- C) The agent writes the request to a database
- D) The agent terminates and a new agent is created after approval

**Answer: B** — NAT WebSocket interactive workflows provide bidirectional real-time communication, enabling the agent to pause and request approval from the connected human.

**Q20.** You are designing an evaluation dataset for an enterprise Q&A agent. The dataset should test whether the agent correctly refuses to answer questions outside its domain. What type of test cases should be included?
- A) Only in-domain questions with correct answers
- B) In-domain questions with correct answers AND out-of-domain questions where the expected behavior is refusal
- C) Only adversarial jailbreak attempts
- D) Random questions from the internet

**Answer: B** — A comprehensive eval dataset includes both positive cases (correct answers) and negative cases (correct refusals for out-of-domain queries).

---

## L. Mini Project

**Full-Stack Design Document**

Produce a complete design document for an agent system of your choice (e.g., customer support, code review, research assistant). The document must include:

1. Architecture diagram with all components labeled
2. Agent type selection with justification
3. Tool inventory with schemas
4. RAG configuration (if applicable) with chunking and retrieval strategy
5. Memory strategy with type selection and overflow handling
6. Guardrails configuration: input, output, dialog, and execution rails
7. Evaluation plan: metrics, dataset design, frequency
8. Deployment plan: NAT server type, NIM configuration, GPU estimate, scaling strategy
9. Monitoring plan: key metrics, alerting rules, debugging procedures
10. Oversight plan: risk classification of each tool, oversight level, approval workflows

Each section must reference the specific NVIDIA tool that implements it and justify the choice over alternatives.

---

## M. How This May Appear on the Exam

1. **Cross-domain integration questions**: "An agent uses RAG for knowledge retrieval and has a guardrail that blocks responses not grounded in retrieved documents. The guardrail is blocking correct responses. What is the most likely issue?" (Answer: the guardrail faithfulness threshold is too strict, or the retrieval step is returning insufficient context.)

2. **NVIDIA tool mapping under pressure**: Rapid-fire "which tool" questions that test your ability to instantly map scenarios to tools. Memorize the tool-scenario table in this module.

3. **Architecture tradeoff questions**: "Given [constraints], which architecture is most appropriate?" These require integrated understanding of agent types, deployment patterns, and resource requirements.

4. **Process questions**: "After deploying an agent to production, what is the recommended approach for continuous improvement?" (Answer: data flywheel — collect production data, evaluate, improve, redeploy.)

5. **"What would you do FIRST" questions**: "A production agent's error rate increases from 2% to 12%. What is your first step?" (Answer: examine traces to identify the error type and source, not immediately change the prompt or model.)

---

## N. Checklist for Mastery (All Domains)

**Fundamentals**
- [ ] I can define what makes a system "agentic" and distinguish agents from chatbots and pipelines

**Architecture**
- [ ] I can select the appropriate NAT agent type for a given scenario and justify the choice

**Foundation Models**
- [ ] I can explain how agents use LLMs for planning, reasoning, and tool selection

**Tool Use**
- [ ] I can design tool schemas, implement error handling, and configure per-user function isolation

**RAG**
- [ ] I can design a RAG pipeline with appropriate chunking, retrieval, and reranking strategies

**Memory**
- [ ] I can select memory types, manage context overflow, and configure cross-session persistence

**Safety and Guardrails**
- [ ] I can configure NeMo Guardrails with input, output, dialog, and execution rails using Colang

**Evaluation**
- [ ] I can design evaluation datasets, select metrics, and interpret results for agent quality assessment

**Deployment**
- [ ] I can select NAT deployment servers, configure NIM, estimate GPU requirements, and design scaling strategies

**Monitoring**
- [ ] I can configure observability, read traces, debug failures, and design monitoring dashboards

**Human Oversight**
- [ ] I can implement risk-proportional oversight using NAT interactive workflows, execution rails, and per-user functions

**Integration**
- [ ] I can design a complete agent system that addresses all 10 domains with justified NVIDIA tool selections
- [ ] I can answer exam-style questions in under 100 seconds with high accuracy
- [ ] I can distinguish between NVIDIA-documented features and inferred best practices
