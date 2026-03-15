# Reading Track

For each module, the official NVIDIA readings that support the lesson content. Organized by required and optional readings, with specific extraction guidance for each.

> **Note on URLs:** All URLs point to official NVIDIA source domains. Where documentation is thin or specific pages may have moved, this is stated explicitly. Always verify URLs are current — NVIDIA documentation restructures periodically.

---

## Module 1: Foundations — What Makes a System Agentic

### Required Readings

**1. NeMo Agent Toolkit Documentation — Getting Started**
- URL: https://docs.nvidia.com/nemo/agent-toolkit/latest/getting-started.html
- What to extract: Installation instructions, first agent configuration, supported agent types overview. Understand the relationship between NAT and the rest of the NeMo ecosystem.
- Why it matters: This is your primary reference for the entire course. Familiarity with the documentation structure saves hours of searching later.

**2. NVIDIA NIM Documentation — Overview**
- URL: https://docs.nvidia.com/nim/
- What to extract: What NIM is, how NIM endpoints work, the API format (OpenAI-compatible), available model families. Understand NIM as the inference layer that all other tools build on.
- Why it matters: Every module uses NIM endpoints. Understanding the NIM API contract (request format, response format, authentication) is prerequisite knowledge.

**3. NVIDIA Agentic AI Certification Page**
- URL: https://www.nvidia.com/en-us/training/certification/agentic-ai-professional/
- What to extract: Exam domains, competency descriptions, recommended training courses. Map these to your study plan.
- Why it matters: This is the authoritative source for what the exam tests. The course is built from this page.

### Optional Readings

**4. NVIDIA Blog — Introduction to Agentic AI**
- URL: https://blogs.nvidia.com (search "agentic AI introduction")
- What to extract: NVIDIA's official framing of what agentic AI means and how their ecosystem supports it. Note the terminology — the exam will use NVIDIA's framing.
- Why it matters: Aligns your mental model with NVIDIA's perspective, which is what the exam tests.

**5. build.nvidia.com — NIM API Playground**
- URL: https://build.nvidia.com
- What to extract: Hands-on experience with NIM API calls. Try at least 3 different models. Note the API format, rate limits, and response structure.
- Why it matters: Lab 1 requires making NIM API calls. Doing exploratory calls first reduces setup friction.

---

## Module 2: Agent Architecture Patterns

### Required Readings

**1. NeMo Agent Toolkit Documentation — Agent Types**
- URL: https://docs.nvidia.com/nemo/agent-toolkit/latest/agents/
- What to extract: Complete descriptions of all seven agent types: ReAct, Reasoning, ReWOO, Router, Sequential Executor, Tool Calling, Responses API Agent. For each, understand: when to use it, YAML configuration syntax, and behavioral characteristics.
- Why it matters: This is the highest-weight exam domain (15%). You must know every agent type, its use case, and its tradeoffs.

**2. NVIDIA AI Blueprints — Catalog**
- URL: https://build.nvidia.com/blueprints
- What to extract: Browse available Blueprints. For AI-Q, Warehouse, and Data Flywheel Blueprints specifically: read the architecture overview, identify which agent patterns each uses, and note the deployment requirements.
- Why it matters: Blueprints are NVIDIA's reference architectures. The exam tests whether you can identify which patterns a Blueprint uses and why.

**3. NeMo Agent Toolkit Documentation — YAML Configuration Reference**
- URL: https://docs.nvidia.com/nemo/agent-toolkit/latest/configuration/
- What to extract: Full YAML configuration schema. Understand every configurable field: agent type, model, tools, system prompt, middleware, parameters. Practice writing configs from scratch.
- Why it matters: Lab 2 requires writing YAML configs for multiple agent types. The exam may test configuration knowledge.

### Optional Readings

**4. NVIDIA AI-Q Blueprint — Architecture Deep Dive**
- URL: https://build.nvidia.com/nvidia/aiq
- What to extract: Component diagram, data flow, deployment requirements (2xH100 for AI-Q specifically), and how the Blueprint composes multiple NVIDIA tools into a complete system. Always verify hardware requirements on the specific Blueprint page or repository before attempting full deployment. Requirements vary significantly across Blueprints.
- Why it matters: AI-Q is the most referenced Blueprint in this course. Understanding its architecture provides a concrete anchor for abstract concepts.

---

## Module 3: Reasoning, Planning, and Memory

### Required Readings

**1. NeMo Agent Toolkit Documentation — Reasoning Agent**
- URL: https://docs.nvidia.com/nemo/agent-toolkit/latest/agents/reasoning.html
- What to extract: How the Reasoning Agent implements test-time compute. Understand the multi-generation, planning, and selection phases. Note configuration parameters that control the tradeoff between quality and latency.
- Why it matters: Test-time compute is an NVIDIA-specific concept in the context of NAT. The exam tests whether you understand what it does and when to use it.

**2. NeMo Agent Toolkit Documentation — Memory and Object Stores**
- URL: https://docs.nvidia.com/nemo/agent-toolkit/latest/memory/
- What to extract: All four Object Store backends (in-memory, MySQL, Redis, S3), when to use each, configuration syntax. Understand the Automatic Memory Wrapper — what it does, how to enable it, and when NOT to use it.
- Why it matters: Memory is a core part of the Cognition domain (10% of exam). You must know the Object Store options and their tradeoffs.

**3. NeMo Agent Toolkit Documentation — ReWOO Agent**
- URL: https://docs.nvidia.com/nemo/agent-toolkit/latest/agents/rewoo.html
- What to extract: How ReWOO separates plan generation from plan execution. Understand the plan format, dependency tracking between steps, and the key limitation (no mid-execution adaptation).
- Why it matters: ReWOO vs. ReAct is a fundamental architecture comparison tested in the exam.

### Optional Readings

**4. Original ReAct Paper** (Yao et al., 2023)
- Note: This is not an NVIDIA source. Read for conceptual depth only.
- What to extract: The formal definition of the ReAct pattern and why interleaving reasoning with acting improves performance.
- Why it matters: Deep understanding of ReAct helps you reason about when NAT's ReAct agent is the right choice.

---

## Module 4: Knowledge Integration and RAG Pipelines

### Required Readings

**1. NeMo Retriever Documentation**
- URL: https://docs.nvidia.com/nemo/retriever/
- What to extract: Architecture of the retrieval pipeline, supported embedding models, indexing options, search API. Understand how NeMo Retriever fits into a RAG pipeline.
- Why it matters: Knowledge Integration is 10% of the exam. NeMo Retriever is the primary NVIDIA tool for this domain.

**2. NeMo Curator Documentation**
- URL: https://docs.nvidia.com/nemo/curator/
- What to extract: Available preprocessing operations (deduplication, language filtering, quality scoring), GPU acceleration details, and integration with downstream embedding and indexing.
- Why it matters: Data quality directly determines RAG quality. Understanding NeMo Curator's capabilities is essential for designing production RAG pipelines.

**3. AI-Q Blueprint Documentation**
- URL: https://build.nvidia.com/nvidia/aiq
- What to extract: Specific component choices — Llama 3.2 NV EmbedQA (embedding), Llama 3.2 NV RerankQA (reranking), Milvus + cuVS (vector store), PaddleOCR (document processing). Hardware requirements (2xH100 for AI-Q specifically). Data flow from ingestion to answer generation. Always verify hardware requirements on the specific Blueprint page or repository before attempting full deployment. Requirements vary significantly across Blueprints.
- Why it matters: The AI-Q Blueprint is the definitive NVIDIA reference for RAG architecture. The exam tests component identification and architecture understanding.

**4. NVIDIA Embedding Models on build.nvidia.com**
- URL: https://build.nvidia.com (search for embedding models)
- What to extract: Available embedding models, dimensionality, max sequence length, and API usage. Test at least one embedding call.
- Why it matters: Lab 4 requires embedding API calls. Understanding model specifications helps with chunking decisions (chunk size must fit within the model's max sequence length).

### Optional Readings

**5. Milvus Documentation — GPU Index (cuVS)**
- URL: https://milvus.io/docs (search for GPU index)
- What to extract: How cuVS integrates with Milvus, which index types support GPU acceleration, and performance benchmarks.
- Why it matters: Understanding cuVS helps answer questions about when GPU-accelerated vector search is worth the cost.

---

## Module 5: Agent Development — Tools and Prompt Engineering

### Required Readings

**1. NeMo Agent Toolkit Documentation — Functions and Tool Registry**
- URL: https://docs.nvidia.com/nemo/agent-toolkit/latest/tools/
- What to extract: How to define functions, register them in the tool registry, create Function Groups, and configure Per-User Functions. Understand the function signature format that NAT expects.
- Why it matters: Tool registration is a core development skill tested in the Agent Development domain (15%).

**2. NeMo Agent Toolkit Documentation — MCP Client**
- URL: https://docs.nvidia.com/nemo/agent-toolkit/latest/mcp/
- What to extract: How NAT connects to MCP servers, discovers tools, and invokes them. Configuration for MCP client in YAML.
- Why it matters: MCP support is a distinguishing NAT feature. Understanding dynamic tool discovery is important for the exam.

**3. NeMo Agent Toolkit Documentation — Agent Hyperparameter Optimizer**
- URL: https://docs.nvidia.com/nemo/agent-toolkit/latest/optimization/
- What to extract: What parameters can be optimized, how the search works, how to define the evaluation metric, and how to interpret results.
- Why it matters: Parameter optimization is the NVIDIA-recommended approach to prompt engineering. Knowing this tool distinguishes you from candidates who only know manual prompt tuning.

> **Note:** Official documentation on the Agent Hyperparameter Optimizer may be thinner than other NAT features. If the linked page is sparse, search the NAT GitHub repository for examples and the NAT API reference for the optimizer class.

### Optional Readings

**4. NVIDIA Blog — Structured Outputs and Function Calling**
- URL: https://blogs.nvidia.com (search for "function calling" or "structured outputs")
- What to extract: NVIDIA's approach to structured outputs, JSON schema enforcement, and how it integrates with tool calling agents.
- Why it matters: Structured outputs reduce tool call errors — understanding why helps with architecture decisions.

---

## Module 6: NeMo Agent Toolkit and Orchestration

### Required Readings

**1. NeMo Agent Toolkit Documentation — Full Reference**
- URL: https://docs.nvidia.com/nemo/agent-toolkit/latest/
- What to extract: End-to-end workflow: configuration → build → deploy → evaluate. Middleware system (caching, defense, logging, red teaming). Plugin system (LangChain, LlamaIndex, CrewAI, AutoGen, Semantic Kernel). Authentication mechanisms (API Key, OAuth2, Bearer, HTTP Basic, MCP accounts).
- Why it matters: This module requires comprehensive NAT knowledge. The full reference is your primary study material.

**2. NeMo Agent Toolkit Documentation — Middleware**
- URL: https://docs.nvidia.com/nemo/agent-toolkit/latest/middleware/
- What to extract: Each middleware type, configuration syntax, execution order, and how middleware interacts with the agent pipeline.
- Why it matters: Middleware is a key NAT concept — it adds capabilities (caching, security, logging) without modifying agent logic.

**3. NeMo Agent Toolkit Documentation — Plugin System**
- URL: https://docs.nvidia.com/nemo/agent-toolkit/latest/plugins/
- What to extract: Supported frameworks, how plugins wrap third-party agents into NAT workflows, and limitations of the plugin approach.
- Why it matters: The plugin system is NAT's interoperability story. Understanding it helps answer "how do I use NVIDIA tools with my existing agent?" — a common exam scenario.

### Optional Readings

**4. NeMo Agent Toolkit GitHub Repository**
- URL: https://github.com/NVIDIA/NeMo-Agent-Toolkit (verify current URL — may be under a different organization)
- What to extract: Example configurations, integration tests, and real YAML files. Reading actual configs is more informative than documentation for many students.
- Why it matters: Documentation describes the happy path; source code shows edge cases and actual usage patterns.

> **Note:** The NAT GitHub repository URL may differ from the above. Check NVIDIA's official GitHub organizations (NVIDIA, NVIDIA-NeMo, NVIDIA-AI-Blueprints) for the current location.

---

## Module 7: Multi-Agent Systems

### Required Readings

**1. NeMo Agent Toolkit Documentation — Router Agent**
- URL: https://docs.nvidia.com/nemo/agent-toolkit/latest/agents/router.html
- What to extract: How the Router Agent classifies and dispatches requests. Configuration for routing rules, confidence thresholds, and fallback behavior.
- Why it matters: The Router Agent is the coordination primitive for multi-agent systems in NAT.

**2. NeMo Agent Toolkit Documentation — A2A Server**
- URL: https://docs.nvidia.com/nemo/agent-toolkit/latest/deployment/a2a.html
- What to extract: A2A protocol structure, message format, task lifecycle, agent discovery. How to deploy agents as A2A services.
- Why it matters: A2A is what makes multi-agent systems real distributed systems rather than functions calling functions in the same process.

**3. NVIDIA Multi-Agent Warehouse Blueprint**
- URL: https://build.nvidia.com/blueprints (search for warehouse or multi-agent)
- What to extract: Architecture for multi-agent coordination, how agents communicate, how tasks are distributed, and what infrastructure is shared.
- Why it matters: This is NVIDIA's reference multi-agent architecture. The exam tests understanding of multi-agent coordination patterns.

> **Note:** Documentation on the Warehouse Blueprint may be limited to the Blueprint landing page and README. If detailed architecture documentation is thin, focus on the Blueprint's source code and Docker Compose files for concrete implementation details.

### Optional Readings

**4. Google A2A Protocol Specification**
- Note: Not an NVIDIA source. Read for protocol-level understanding.
- What to extract: The formal A2A protocol specification, message types, and lifecycle. Helps understand what NAT's A2A Server implements.
- Why it matters: Deep protocol understanding helps debug A2A communication issues.

---

## Module 8: Evaluation and Tuning

### Required Readings

**1. NeMo Agent Toolkit Documentation — Evaluation Framework**
- URL: https://docs.nvidia.com/nemo/agent-toolkit/latest/evaluation/
- What to extract: All evaluation types: RAG evaluation (faithfulness, relevance, context metrics), red teaming, trajectory evaluation, custom evaluators. Dataset loading. Evaluation configuration.
- Why it matters: Evaluation is 13% of the exam. The NAT eval framework is the primary tool.

**2. NeMo Agent Toolkit Documentation — Profiler**
- URL: https://docs.nvidia.com/nemo/agent-toolkit/latest/profiler/
- What to extract: Profiler configuration, what it measures (token counts, latency at workflow/step/tool level), how to interpret profiler output, and how to use it for optimization decisions.
- Why it matters: The Profiler is distinct from the Evaluation Framework — knowing the difference is testable.

**3. NeMo Agent Toolkit Documentation — Sizing Calculator**
- URL: https://docs.nvidia.com/nemo/agent-toolkit/latest/sizing/
- What to extract: How to estimate resource requirements (GPU count, memory, replicas) for a given workload profile. Input parameters and output recommendations.
- Why it matters: Resource planning is part of the Deployment domain but overlaps with Evaluation. Understanding the Sizing Calculator connects evaluation results to deployment decisions.

**4. NVIDIA Data Flywheel Blueprint**
- URL: https://build.nvidia.com/blueprints (search for data flywheel)
- What to extract: The continuous improvement cycle: collect → evaluate → curate → fine-tune → deploy. How evaluation feeds into the improvement process.
- Why it matters: The Data Flywheel is NVIDIA's approach to continuous optimization. Understanding it connects evaluation to ongoing quality improvement.

> **Note:** Data Flywheel Blueprint documentation may be limited. If the Blueprint page is sparse, the concept is well-described in NVIDIA blog posts. Search blogs.nvidia.com for "data flywheel."

### Optional Readings

**5. NVIDIA DLI Course — Evaluating RAG and Semantic Search Systems**
- URL: https://learn.nvidia.com (search for the course title)
- What to extract: Practical evaluation methodology, specific metric implementations, and hands-on evaluation exercises.
- Why it matters: Supplements the course's evaluation module with additional hands-on practice.

---

## Module 9: Safety, Guardrails, and Governance

### Required Readings

**1. NeMo Guardrails Documentation**
- URL: https://docs.nvidia.com/nemo/guardrails/
- What to extract: Complete Colang 2.0 syntax. All five rail types with examples. LLMRails API. RailsConfig structure. Integration with Safety NIMs. PII detection integration (GLiNER, Presidio, PrivateAI). Execution rails for tool call validation.
- Why it matters: NeMo Guardrails is the safety layer for the entire NVIDIA agentic stack. This documentation is comprehensive and is the primary source for the Safety domain (5%).

**2. NeMo Guardrails GitHub Repository**
- URL: https://github.com/NVIDIA/NeMo-Guardrails
- What to extract: Example configurations for each rail type, integration examples with NIM, and the Colang 2.0 language specification.
- Why it matters: The GitHub repo has more examples than the documentation. Reading real Colang files is the fastest way to learn the syntax.

**3. NVIDIA Blog — Safety NIMs (Content Safety, Jailbreak Detection, Topic Control)**
- URL: https://blogs.nvidia.com (search for "safety NIM" or "content safety NIM")
- What to extract: Architecture of each Safety NIM, what it classifies, how it integrates with NeMo Guardrails, and deployment requirements.
- Why it matters: Safety NIMs are the ML layer of NVIDIA's safety story. Understanding their capabilities helps design multi-layered guardrails.

**4. NeMo Agent Toolkit Documentation — Defense and Red-Teaming Middleware**
- URL: https://docs.nvidia.com/nemo/agent-toolkit/latest/middleware/ (defense and red-teaming sections)
- What to extract: How NAT defense middleware adds attack detection, and how red-teaming middleware automates adversarial testing.
- Why it matters: These middleware types connect NAT's agent pipeline to NeMo Guardrails' safety capabilities.

### Optional Readings

**5. NeMo Auditor Documentation**
- URL: https://docs.nvidia.com/nemo/auditor/ (verify current URL)
- What to extract: Pre-deployment auditing capabilities, how it replaces the deprecated Safety Blueprint, and integration with the deployment workflow.
- Why it matters: NeMo Auditor is referenced in the course but documentation may be thin. Read what is available and note the gaps.

> **Note:** NeMo Auditor is relatively new, and documentation may be limited. If the URL above does not resolve, search docs.nvidia.com for "NeMo Auditor." The course covers it at the level available in official docs.

---

## Module 10: Deployment and Scaling

### Required Readings

**1. NeMo Agent Toolkit Documentation — Deployment Servers**
- URL: https://docs.nvidia.com/nemo/agent-toolkit/latest/deployment/
- What to extract: All four server types (FastAPI, MCP, FastMCP, A2A). Configuration for each. When to use each. Health check configuration. Authentication setup.
- Why it matters: Deployment is 13% of the exam. Knowing which server type to use for a given scenario is directly testable.

**2. NVIDIA NIM Documentation — Deployment Guide**
- URL: https://docs.nvidia.com/nim/deployment/
- What to extract: How to deploy NIM containers, GPU requirements per model size, TensorRT-LLM optimization, model caching, and health monitoring.
- Why it matters: NIM is the inference layer for all agent systems. Understanding deployment configuration is essential.

**3. NIM Operator Documentation**
- URL: https://docs.nvidia.com/nim/operator/
- What to extract: How to install the NIM Operator, define NIM deployments as custom resources, configure GPU allocation, and set up auto-scaling.
- Why it matters: NIM Operator is the recommended production deployment approach for Kubernetes. The exam tests awareness of its capabilities.

**4. NVIDIA Dynamo Documentation**
- URL: https://docs.nvidia.com/dynamo/ (verify current URL)
- What to extract: Dynamic batching, prefix caching, disaggregated serving, and multi-node inference capabilities. Understand which optimizations apply to agent workloads.
- Why it matters: Dynamo is NVIDIA's inference optimization layer. Understanding its features helps answer questions about latency optimization.

> **Note:** Dynamo documentation for agent-specific usage may be thin. The core Dynamo features (batching, prefix caching) are well-documented, but how they apply specifically to agent workloads may require inference from general documentation.

### Optional Readings

**5. NVIDIA DLI Course — Deploying RAG Pipelines for Production at Scale**
- URL: https://learn.nvidia.com (search for the course title)
- What to extract: Production deployment patterns, scaling strategies, and hands-on deployment exercises with NIM.
- Why it matters: Supplements the course with NVIDIA's own production deployment training.

---

## Module 11: Observability, Monitoring, and Maintenance

### Required Readings

**1. NeMo Agent Toolkit Documentation — Observability**
- URL: https://docs.nvidia.com/nemo/agent-toolkit/latest/observability/
- What to extract: Supported observability backends (Phoenix, Weave, Langfuse, OpenTelemetry). Tracing granularity (workflow, step, tool). Configuration syntax. Token counting and latency metric collection. Redaction processors.
- Why it matters: This is the primary reference for the Monitor domain (5%). The exam tests whether you know NAT's tracing capabilities and observability configuration.

**2. NeMo Agent Toolkit Documentation — Redaction Processors**
- URL: https://docs.nvidia.com/nemo/agent-toolkit/latest/observability/ (redaction section)
- What to extract: How redaction processors work, what they mask, and how to configure custom redaction rules for telemetry data.
- Why it matters: Redaction processors are a specific NAT feature for handling sensitive data in traces — a niche but testable topic.

### Optional Readings

**3. OpenTelemetry Documentation — Python SDK**
- URL: https://opentelemetry.io/docs/languages/python/
- Note: Not an NVIDIA source. Read for protocol understanding.
- What to extract: How OpenTelemetry traces, spans, and exporters work. This helps you understand what NAT is doing under the hood.
- Why it matters: NAT uses OpenTelemetry as its telemetry foundation. Understanding the protocol helps debug trace issues.

**4. Arize Phoenix Documentation**
- URL: https://docs.arize.com/phoenix
- Note: Not an NVIDIA source. Read for backend understanding.
- What to extract: How Phoenix ingests and visualizes OpenTelemetry traces. Dashboard capabilities for LLM traces.
- Why it matters: Phoenix is the most commonly referenced observability backend in NAT documentation.

> **Note:** NVIDIA's documentation on monitoring dashboards and alerting strategies for agent systems is thin. The course supplements official docs with operational best practices derived from general observability literature applied to agent-specific concerns.

---

## Module 12: Human-AI Interaction and Oversight

### Required Readings

**1. NeMo Agent Toolkit Documentation — Interactive Workflows**
- URL: https://docs.nvidia.com/nemo/agent-toolkit/latest/deployment/ (interactive/WebSocket section)
- What to extract: How NAT supports WebSocket-based bidirectional communication. How to send intermediate results and receive human feedback during agent execution.
- Why it matters: Interactive workflows are NAT's primitive for human-in-the-loop design.

**2. NeMo Guardrails Documentation — Execution Rails**
- URL: https://docs.nvidia.com/nemo/guardrails/ (execution rails section)
- What to extract: How execution rails intercept tool calls and can pause for human approval. Colang 2.0 syntax for approval flows.
- Why it matters: Execution rails are the mechanism for implementing human approval gates — the core HITL pattern.

**3. NeMo Agent Toolkit Documentation — Per-User Functions and Authentication**
- URL: https://docs.nvidia.com/nemo/agent-toolkit/latest/authentication/
- What to extract: How to configure per-user tool access based on authentication. OAuth2, API Key, Bearer, HTTP Basic, MCP account configuration.
- Why it matters: Per-user functions enable role-based access control in agent systems, which is part of the governance story.

### Optional Readings

**4. NVIDIA Blog — Human-AI Collaboration**
- URL: https://blogs.nvidia.com (search "human AI collaboration" or "human in the loop")
- What to extract: NVIDIA's perspective on human oversight for AI systems. Note any specific patterns or frameworks they recommend.
- Why it matters: The Human-AI domain is the weakest in NVIDIA-specific documentation. Blog posts may provide additional NVIDIA perspective that supplements the official docs.

> **Note:** Human-AI Interaction has the thinnest NVIDIA documentation of any exam domain. NVIDIA provides implementation primitives (interactive workflows, execution rails, per-user functions) but does not publish design patterns for HITL systems. The course bridges this gap by constructing patterns from these primitives, but students should be aware that exam questions in this domain may test primitive knowledge rather than pattern knowledge.

---

## Module 13: Certification Prep and Integration

### Required Readings

**1. All previously assigned required readings** — revisit any that were not fully absorbed.

**2. NVIDIA NCP-AAI Certification Page (revisit)**
- URL: https://www.nvidia.com/en-us/training/certification/agentic-ai-professional/
- What to extract: Re-read the exam domains and competency statements. For each competency, verify you can explain it using NVIDIA-specific terminology and tools.
- Why it matters: This is your final alignment check. The exam tests what this page describes.

**3. NVIDIA NeMo Agent Toolkit v1.5 Release Notes**
- URL: https://docs.nvidia.com/nemo/agent-toolkit/latest/release-notes.html
- What to extract: New features in the current version, deprecated features, and known limitations. The exam may test current capabilities.
- Why it matters: Knowing what is new and what has changed helps avoid answering based on outdated information.

### Optional Readings

**4. NVIDIA DLI Recommended Courses (all five listed in the Learning Path)**
- URLs: Available at https://learn.nvidia.com
- Courses: Building RAG Agents With LLMs, Evaluating RAG and Semantic Search Systems, Building Agentic AI Applications With LLMs, Adding New Knowledge to LLMs, Deploying RAG Pipelines for Production at Scale
- What to extract: Any hands-on exercises or concepts not covered in this course. These are the courses NVIDIA recommends for exam preparation.
- Why it matters: These are NVIDIA's own training materials for the certification. They may cover topics or perspectives this course does not.

**5. NVIDIA Technical Blog — Agentic AI Posts**
- URL: https://blogs.nvidia.com (search "agentic AI" and filter to last 6 months)
- What to extract: New announcements, product updates, and conceptual frameworks published since this course was written.
- Why it matters: The NVIDIA ecosystem evolves. Recent blog posts may cover new tools or changes that affect exam content.

---

## Reading Prioritization Guide

If you have limited reading time, prioritize in this order:

1. **NAT Documentation — Agent Types** (Module 2) — highest exam weight, most NVIDIA-specific
2. **NAT Documentation — Full Reference** (Module 6) — broadest coverage of development skills
3. **NAT Documentation — Evaluation Framework** (Module 8) — third-highest exam weight
4. **NeMo Guardrails Documentation** (Module 9) — comprehensive, well-written, testable
5. **NAT Documentation — Deployment Servers** (Module 10) — directly testable deployment knowledge
6. **AI-Q Blueprint Documentation** (Module 4) — concrete reference architecture for RAG
7. **NAT Documentation — Memory and Object Stores** (Module 3) — memory systems are testable
8. **NAT Documentation — Observability** (Module 11) — niche but directly testable
9. **NIM Documentation** (Module 1, 10) — inference layer knowledge
10. **Everything else** — diminishing returns, but still valuable

Total estimated reading time: 25-35 hours for all required readings, 10-15 hours for optional readings.
