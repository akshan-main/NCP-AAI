# Glossary

Every significant term used in this course, organized alphabetically. For each term: a precise definition, NVIDIA-specific context where applicable, and the module where it is first introduced.

---

### A2A (Agent-to-Agent Protocol)

**Definition:** A structured communication protocol for inter-agent messaging that supports typed payloads, task delegation, status tracking, and agent discovery between independently deployed agent services.

**NVIDIA context:** NAT provides an A2A Server deployment option that implements this protocol, enabling multi-agent systems where each agent runs as a separate service.

**First introduced in:** Module 7

---

### Agent

**Definition:** A software system that perceives its environment, reasons about what actions to take, executes those actions via tools or APIs, and observes the results to inform future reasoning. Distinguished from pipelines by its ability to adapt behavior based on intermediate observations.

**NVIDIA context:** In the NVIDIA ecosystem, agents are built using NAT (NeMo Agent Toolkit), which provides seven agent type implementations (ReAct, Reasoning, ReWOO, Router, Sequential Executor, Tool Calling, Responses API Agent).

**First introduced in:** Module 1

---

### Agent Hyperparameter Optimizer

**Definition:** A systematic search tool that optimizes agent configuration parameters (system prompt wording, temperature, top-p, max tokens, few-shot examples, tool descriptions) by running the agent against an evaluation dataset with different parameter combinations and selecting the configuration that maximizes target metrics.

**NVIDIA context:** Built into NAT as the Parameter Optimizer. Replaces manual prompt tuning with data-driven optimization.

**First introduced in:** Module 5

---

### Agent Loop

**Definition:** The fundamental execution cycle of an agentic system, consisting of four phases: Perceive (receive input), Reason (determine next action), Act (execute the action), Observe (process the result). The loop repeats until a termination condition is met.

**NVIDIA context:** NAT agent types implement different variants of this loop — ReAct interleaves reasoning and acting, ReWOO separates planning from execution, Tool Calling runs a single act-observe cycle.

**First introduced in:** Module 1

---

### Agentic AI

**Definition:** AI systems that exhibit agency — the capacity to autonomously perceive, reason, plan, and act in pursuit of goals, adapting their behavior based on environmental feedback rather than following fixed sequences.

**NVIDIA context:** NVIDIA's agentic AI ecosystem spans NIM (inference), NAT (orchestration), NeMo Guardrails (safety), NeMo Evaluator (quality), and NeMo Retriever (knowledge). The NCP-AAI certification tests proficiency across this ecosystem.

**First introduced in:** Module 1

---

### Automatic Memory Wrapper

**Definition:** A NAT component that transparently adds memory capabilities to any agent by intercepting inputs and outputs, automatically storing relevant information in a configured Object Store, and retrieving relevant memories to inject into the agent's context on subsequent calls.

**NVIDIA context:** Part of NAT's memory system. Provides a zero-code way to add memory to agents, though it offers less control than manual memory management.

**First introduced in:** Module 3

---

### Blueprint

**Definition:** A complete, deployable reference architecture published by NVIDIA that demonstrates how to compose multiple NVIDIA tools into a production system for a specific use case.

**NVIDIA context:** Key Blueprints include AI-Q (RAG research assistant), Multi-Agent Warehouse (multi-agent coordination), and Data Flywheel (continuous improvement). Available at build.nvidia.com/blueprints.

**First introduced in:** Module 2

---

### Chain-of-Thought (CoT)

**Definition:** A prompting strategy where the LLM is instructed to produce intermediate reasoning steps before arriving at a final answer, improving accuracy on complex reasoning tasks by making the reasoning process explicit and sequential.

**NVIDIA context:** NAT's Reasoning Agent and ReAct Agent leverage chain-of-thought reasoning as part of their agent loop. Test Time Compute extends this by generating multiple reasoning chains and selecting the best.

**First introduced in:** Module 3

---

### Colang

**Definition:** A domain-specific language developed by NVIDIA for defining guardrail behaviors, conversational flows, and safety policies. Version 2.0 uses a Python-like syntax for defining flows, actions, and event handlers.

**NVIDIA context:** Colang 2.0 is the configuration language for NeMo Guardrails. It defines input rails, output rails, dialog rails, topical rails, and execution rails.

**First introduced in:** Module 9

---

### Content Safety NIM

**Definition:** A NIM microservice that classifies text inputs for harmful content across multiple safety categories (violence, sexual content, illegal activity, etc.) and returns safety scores.

**NVIDIA context:** Deployed as a NIM endpoint and integrated with NeMo Guardrails as an input rail. Part of NVIDIA's Safety NIM family alongside Jailbreak Detection NIM and Topic Control NIM.

**First introduced in:** Module 9

---

### cuVS

**Definition:** CUDA Vector Search — NVIDIA's GPU-accelerated library for approximate nearest neighbor (ANN) search. Provides GPU implementations of index types (HNSW, IVF) that leverage parallelism for high-throughput concurrent vector search.

**NVIDIA context:** Integrated into Milvus to accelerate vector similarity search. Most beneficial under high concurrent query load where CPU-based search becomes a bottleneck.

**First introduced in:** Module 4

---

### Data Flywheel

**Definition:** A continuous improvement cycle where production system outputs and failures are collected, evaluated, curated as training data, used to fine-tune models, and redeployed — creating a virtuous cycle of improving quality.

**NVIDIA context:** NVIDIA publishes a Data Flywheel Blueprint that implements this cycle using NAT (collection), NeMo Evaluator (evaluation), NeMo Curator (curation), and NeMo Customizer (fine-tuning).

**First introduced in:** Module 8

---

### Dialog Rail

**Definition:** A NeMo Guardrails rail type that controls the structure and sequence of conversational interactions — what the bot says when, how it responds to specific interaction patterns, and what conversational flows are allowed.

**NVIDIA context:** Defined in Colang 2.0. Differs from topical rails (which control subject matter) by controlling conversational structure (e.g., "always confirm before proceeding").

**First introduced in:** Module 9

---

### Embedding

**Definition:** A dense vector representation of text (or other data) in a continuous vector space, where semantic similarity between texts corresponds to geometric proximity between their vectors. Used for retrieval, clustering, and similarity search.

**NVIDIA context:** NVIDIA provides embedding models via NIM endpoints (e.g., Llama 3.2 NV EmbedQA). NAT includes embedder components that connect to these NIM endpoints for RAG pipeline construction.

**First introduced in:** Module 4

---

### Execution Rail

**Definition:** A NeMo Guardrails rail type that intercepts and validates tool/action calls before they execute, checking tool name authorization, parameter validity, and contextual appropriateness.

**NVIDIA context:** Critical for agentic security — prevents agents from executing unauthorized or dangerous tool calls even if the LLM generates them. Can also implement human approval gates.

**First introduced in:** Module 9

---

### Faithfulness

**Definition:** A RAG evaluation metric measuring whether every claim in a generated answer is supported by the retrieved passages. High faithfulness means the answer does not contain information beyond what the source documents provide.

**NVIDIA context:** Measured via the NAT evaluation framework using LLM-as-Judge scoring. A key metric for production RAG systems — target threshold typically >= 0.90.

**First introduced in:** Module 8

---

### Function Group

**Definition:** A named collection of functions (tools) in NAT that can be assigned to an agent as a unit, enabling modular and reusable tool configuration.

**NVIDIA context:** Part of NAT's function registry system. Function Groups simplify agent configuration when multiple agents share subsets of tools.

**First introduced in:** Module 5

---

### Guardrail

**Definition:** A runtime safety mechanism that monitors, validates, and potentially modifies AI system inputs, outputs, or actions to enforce safety policies, prevent harmful behavior, and maintain operational boundaries.

**NVIDIA context:** NeMo Guardrails is NVIDIA's guardrail framework, configured via Colang 2.0 with five rail types (input, output, dialog, topical, execution).

**First introduced in:** Module 9

---

### Hallucination

**Definition:** When an LLM generates information that is not supported by its input context or training data, presenting fabricated facts, citations, or claims as though they were real.

**NVIDIA context:** Addressed in the NVIDIA ecosystem through: RAG (grounding in retrieved documents), NeMo Guardrails fact-checking output rail (verifying claims against context), and faithfulness evaluation in the NAT evaluation framework.

**First introduced in:** Module 4

---

### Input Rail

**Definition:** A NeMo Guardrails rail type that processes user input before it reaches the LLM, filtering for harmful content, detecting jailbreak attempts, and enforcing topical boundaries.

**NVIDIA context:** Typically implemented using Content Safety NIM and Jailbreak Detection NIM integrated via Colang 2.0 flows.

**First introduced in:** Module 9

---

### Jailbreak

**Definition:** An adversarial attack that attempts to bypass an LLM's safety constraints or system prompt instructions, causing the model to produce outputs it was designed to refuse (harmful content, system prompt disclosure, role-play as an unrestricted model).

**NVIDIA context:** Detected by Jailbreak Detection NIM and NeMo Guardrails input rails. NAT red-teaming middleware includes standard jailbreak attack libraries for testing.

**First introduced in:** Module 9

---

### Jailbreak Detection NIM

**Definition:** A NIM microservice that classifies text inputs for jailbreak attempts, including prompt injection, role-play attacks, system prompt extraction, and encoding-based bypasses.

**NVIDIA context:** Deployed as a NIM endpoint and integrated with NeMo Guardrails as an input rail. Part of NVIDIA's Safety NIM family.

**First introduced in:** Module 9

---

### LLM-as-Judge

**Definition:** An evaluation pattern where a large language model is used to assess the quality of another model's outputs, scoring dimensions such as relevance, faithfulness, coherence, or task completion using structured rubrics.

**NVIDIA context:** NAT evaluation framework supports LLM-as-Judge via custom evaluators. The judge LLM can be a NIM endpoint, enabling automated quality assessment without human labeling.

**First introduced in:** Module 8

---

### LLMRails

**Definition:** The core NeMo Guardrails Python class that orchestrates the guardrails pipeline — loading configuration, initializing rails, and processing inputs through the configured rail flows.

**NVIDIA context:** The primary API entry point for NeMo Guardrails in Python code. Instantiated with a RailsConfig and used to process messages through the guardrail pipeline.

**First introduced in:** Module 9

---

### MCP (Model Context Protocol)

**Definition:** A protocol for dynamic tool discovery and invocation across organizational boundaries. An MCP server exposes tools with standardized descriptions; an MCP client connects at runtime to discover and call those tools.

**NVIDIA context:** NAT includes an MCP client for connecting to MCP servers and an MCP/FastMCP server for exposing NAT agents as MCP-compatible tool providers.

**First introduced in:** Module 5

---

### Memory

**Definition:** In agentic AI, the ability of an agent to retain and recall information across interactions — including conversation history, user preferences, learned facts, and task context — enabling continuity and personalization.

**NVIDIA context:** NAT provides a memory system with multiple Object Store backends (in-memory, MySQL, Redis, S3) and the Automatic Memory Wrapper for transparent memory management.

**First introduced in:** Module 3

---

### Milvus

**Definition:** An open-source vector database purpose-built for storing, indexing, and searching dense vector embeddings at scale. Supports multiple index types (HNSW, IVF, flat), metadata filtering, and partitioning.

**NVIDIA context:** The vector store used in NVIDIA's AI-Q Blueprint and recommended for production RAG deployments. Integrates with cuVS for GPU-accelerated vector search.

**First introduced in:** Module 4

---

### Middleware

**Definition:** Software components that sit between the client request and the agent's core logic, processing requests and responses in transit. Middleware can add functionality (caching, logging, security) without modifying the agent itself.

**NVIDIA context:** NAT provides four middleware types: caching, defense, logging, and red-teaming. Middleware is configured in the NAT YAML and executes in the request pipeline.

**First introduced in:** Module 6

---

### NAT (NeMo Agent Toolkit)

**Definition:** NVIDIA's comprehensive toolkit for building, deploying, evaluating, and monitoring agentic AI systems. Provides YAML-based agent configuration, multiple agent types, deployment servers, evaluation framework, profiler, middleware, and plugin system.

**NVIDIA context:** The central orchestration tool in the NVIDIA agentic AI stack. Installed via `pip install nvidia-nat`. Current version: v1.5. The base package covers core agent types, YAML configuration, functions, and middleware. Evaluation, profiling, and LangChain integration may require extras — see `platform_setup/feature_to_install_matrix.md` for the full mapping.

**First introduced in:** Module 1

---

### NeMo Auditor

**Definition:** An NVIDIA tool for pre-deployment safety auditing of AI systems, performing systematic checks for bias, safety violations, and policy compliance before a system goes into production.

**NVIDIA context:** Positioned as the replacement for the deprecated Safety Blueprint. Used in the pre-deployment phase of the safety lifecycle.

**First introduced in:** Module 9

---

### NeMo Curator

**Definition:** An NVIDIA tool for GPU-accelerated data preparation at scale, providing operations such as fuzzy deduplication, language filtering, quality scoring, text extraction, and data cleaning — all leveraging GPU parallelism for 10-100x speedup over CPU processing.

**NVIDIA context:** Used in the ingestion stage of RAG pipelines. Processes large document corpora (millions of documents) for embedding and indexing.

**First introduced in:** Module 4

---

### NeMo Customizer

**Definition:** An NVIDIA microservice for fine-tuning LLMs using techniques such as supervised fine-tuning (SFT) and Direct Preference Optimization (DPO), enabling model adaptation to specific domains or tasks.

**NVIDIA context:** Used in the Data Flywheel cycle for model improvement. NAT supports finetuning integration with NeMo Customizer and OpenPipe ART.

**First introduced in:** Module 8

---

### NeMo Evaluator

**Definition:** An NVIDIA microservice for systematic evaluation of LLM and agent system quality, providing benchmark evaluation, custom metric computation, and quality reporting.

**NVIDIA context:** Complements the NAT evaluation framework. NeMo Evaluator operates as a standalone microservice; NAT eval framework operates within the NAT workflow.

**First introduced in:** Module 8

---

### NeMo Guardrails

**Definition:** An open-source NVIDIA toolkit for adding programmable guardrails to LLM-powered applications, supporting five rail types (input, output, dialog, topical, execution) configured via the Colang 2.0 language.

**NVIDIA context:** The safety layer in the NVIDIA agentic stack. Integrates with Safety NIMs, PII detection libraries, and NAT middleware. Available in two modes: **library mode** (`pip install nemoguardrails langchain-nvidia-ai-endpoints`, runs on CPU laptop — used in all labs) and **microservice mode** (NGC container deployment, requires Docker, NGC API key, and potentially GPU for Safety NIM models).

**First introduced in:** Module 9

---

### NeMo Retriever

**Definition:** An NVIDIA microservice for building and serving RAG retrieval pipelines, including embedding, indexing, search, and reranking capabilities.

**NVIDIA context:** Provides the retrieval infrastructure for RAG-enabled agents. NAT includes retriever components that connect to NeMo Retriever endpoints.

**First introduced in:** Module 4

---

### NIM (NVIDIA Inference Microservice)

**Definition:** NVIDIA's standardized container format for deploying AI models as microservices with optimized inference performance. Each NIM packages a model with TensorRT-LLM optimization, a standard API, and health monitoring.

**NVIDIA context:** The model serving layer of the NVIDIA stack. NIM endpoints serve LLMs, embedding models, rerankers, and safety models. Accessible via build.nvidia.com for cloud-hosted instances or deployed on-premises.

**First introduced in:** Module 1

---

### NIM Operator

**Definition:** A Kubernetes operator that automates the lifecycle management of NIM deployments, including GPU resource allocation, model caching, health monitoring, and inference-aware scaling.

**NVIDIA context:** The recommended approach for deploying NIM on Kubernetes in production. Eliminates manual GPU configuration and model management.

**First introduced in:** Module 10

---

### NVIDIA Dynamo

**Definition:** An inference acceleration framework that optimizes LLM serving through dynamic batching, prefix caching (KV-cache reuse), disaggregated serving (separating prefill and decode phases), and multi-node inference.

**NVIDIA context:** Integrated with NIM to provide optimized inference for agent systems. Prefix caching is particularly valuable for agents that reuse system prompts across requests.

**First introduced in:** Module 10

---

### Object Store

**Definition:** In NAT, a storage backend for agent memory and state. Provides a unified interface for persisting and retrieving data regardless of the underlying storage technology.

**NVIDIA context:** NAT supports four Object Store backends: in-memory (development), MySQL (queryable persistent storage), Redis (low-latency persistent storage), and S3 (large object archival).

**First introduced in:** Module 3

---

### OpenTelemetry

**Definition:** An open-source observability framework providing standardized APIs, SDKs, and protocols for collecting distributed traces, metrics, and logs from software systems.

**NVIDIA context:** NAT exports telemetry data via OpenTelemetry to observability backends (Phoenix, Weave, Langfuse). Provides workflow-to-tool-level tracing granularity for agent systems.

**First introduced in:** Module 11

---

### Output Rail

**Definition:** A NeMo Guardrails rail type that processes LLM output before it reaches the user, checking for hallucination, PII leakage, format compliance, and other output quality requirements.

**NVIDIA context:** Implements fact-checking rails (verify claims against context), PII redaction (mask sensitive data), and custom output validation via Colang 2.0 flows.

**First introduced in:** Module 9

---

### Parameter Optimization

**Definition:** The systematic process of searching for optimal agent configuration parameters (prompt wording, temperature, sampling parameters, tool descriptions) by evaluating multiple configurations against quality metrics and selecting the best.

**NVIDIA context:** Implemented in NAT as the Agent Hyperparameter Optimizer. Automates what is otherwise manual prompt engineering.

**First introduced in:** Module 5

---

### PII (Personally Identifiable Information)

**Definition:** Information that can identify a specific individual, including names, email addresses, phone numbers, social security numbers, addresses, and other personal identifiers.

**NVIDIA context:** NeMo Guardrails provides PII detection and redaction via integration with GLiNER, Presidio, and PrivateAI. NAT redaction processors can also mask PII in telemetry data.

**First introduced in:** Module 9

---

### Profiler

**Definition:** A tool that measures performance characteristics of a system — execution time, resource usage, throughput, and bottleneck identification — at multiple levels of granularity.

**NVIDIA context:** The NAT Profiler measures token counts, latency, and cost at workflow, step, and tool levels. Used to identify performance bottlenecks and optimize agent configurations.

**First introduced in:** Module 8

---

### RAG (Retrieval-Augmented Generation)

**Definition:** An architecture pattern that grounds LLM generation in retrieved information by first searching a knowledge base for relevant documents, then providing those documents as context to the LLM for answer generation.

**NVIDIA context:** NVIDIA provides end-to-end RAG tooling: NeMo Curator (data prep), NIM embedding endpoints, Milvus + cuVS (vector search), NIM reranker, and NIM LLM. The AI-Q Blueprint is a complete RAG reference architecture.

**First introduced in:** Module 4

---

### RailsConfig

**Definition:** The Python configuration object in NeMo Guardrails that specifies models, rails, prompts, and Colang flows. Used to instantiate the LLMRails class.

**NVIDIA context:** Can be loaded from YAML configuration files or constructed programmatically. Defines which models to use, which rail flows to enable, and guardrail behavior parameters.

**First introduced in:** Module 9

---

### ReAct (Reasoning + Acting)

**Definition:** An agent pattern that interleaves reasoning (thinking about what to do) with acting (executing tools) and observing (processing results) in a loop. After each action, the agent reasons about the result before deciding the next action.

**NVIDIA context:** One of NAT's seven agent types. Best for tasks requiring adaptive multi-step reasoning where the plan may change based on intermediate results. Higher token cost than single-step patterns.

**First introduced in:** Module 2

---

### Reasoning Agent

**Definition:** A NAT agent type that implements test-time compute — generating multiple candidate responses, evaluating them, and selecting the best one. Trades additional LLM calls for higher answer quality on complex reasoning tasks.

**NVIDIA context:** NAT's implementation of test-time compute. Uses multi-LLM generation, planning, and selection phases.

**First introduced in:** Module 3

---

### Redaction Processor

**Definition:** A component that detects and masks sensitive information (PII, secrets, credentials) in text before it is stored, transmitted, or displayed.

**NVIDIA context:** Used in two contexts: (1) NeMo Guardrails output rail for PII redaction in agent responses; (2) NAT telemetry redaction processor for masking sensitive data in observability traces.

**First introduced in:** Module 9

---

### Reranking

**Definition:** A second-stage retrieval step that uses a cross-encoder model to re-score and reorder candidate documents retrieved by the first-stage retriever, capturing fine-grained query-document interactions that bi-encoder embeddings miss.

**NVIDIA context:** NVIDIA provides the Llama 3.2 NV RerankQA model via NIM endpoints. Used in the AI-Q Blueprint RAG pipeline between retrieval and generation.

**First introduced in:** Module 4

---

### ReWOO (Reasoning Without Observation)

**Definition:** An agent pattern that generates a complete execution plan (all tool calls and their dependencies) in a single LLM call, then executes the plan without further LLM reasoning between steps. Minimizes total LLM calls but cannot adapt mid-execution.

**NVIDIA context:** One of NAT's seven agent types. Best for predictable tasks where the plan is unlikely to change. Lower token cost than ReAct but less adaptive.

**First introduced in:** Module 2

---

### Router Agent

**Definition:** An agent pattern that examines the input, classifies it into one of several categories, and dispatches it to the appropriate specialized agent or handler. Does not perform the task itself — it routes to the agent best equipped for the task.

**NVIDIA context:** One of NAT's seven agent types. Used in multi-agent systems as the entry point that dispatches to specialist agents. Can route based on intent classification, keyword matching, or LLM-based classification.

**First introduced in:** Module 2

---

### Safe Synthesizer

**Definition:** An NVIDIA tool (Early Access) for generating safe synthetic data for LLM training and evaluation, ensuring generated data does not contain harmful content, bias, or policy violations.

**NVIDIA context:** Early Access status — documented but not production-ready. Relevant for generating safety evaluation datasets.

**First introduced in:** Module 9

---

### Sequential Executor

**Definition:** A NAT agent type that runs a fixed sequence of steps in order, where the output of each step is the input to the next. No routing decision — every step always executes.

**NVIDIA context:** One of NAT's seven agent types. Best for deterministic pipelines where every request follows the same processing stages (e.g., extract → transform → validate → store).

**First introduced in:** Module 2

---

### Sizing Calculator

**Definition:** A NAT tool that estimates the compute resources (GPU count, memory, replicas) required to serve an agent system at a specified throughput and latency target.

**NVIDIA context:** Part of NAT's planning tools. Used before deployment to estimate infrastructure requirements based on model size, expected concurrency, and latency SLA.

**First introduced in:** Module 8

---

### Test Time Compute

**Definition:** The practice of spending additional compute at inference time (multiple LLM calls, deliberation, self-verification) to improve output quality, trading latency and cost for accuracy.

**NVIDIA context:** Implemented in NAT's Reasoning Agent, which generates multiple candidates, plans evaluation, and selects the best output.

**First introduced in:** Module 3

---

### Tool Calling Agent

**Definition:** A NAT agent type that makes a single tool selection and execution per turn. The LLM decides which tool to call and with what parameters, the tool executes, and the result is returned. No multi-step reasoning loop.

**NVIDIA context:** One of NAT's seven agent types. Best for simple, well-defined tasks where the correct tool is clear from the input. Lowest latency and token cost among agent types.

**First introduced in:** Module 2

---

### Topic Control NIM

**Definition:** A NIM microservice that classifies whether a text input falls within a set of allowed topics, enabling automated enforcement of conversational boundaries.

**NVIDIA context:** Deployed as a NIM endpoint and integrated with NeMo Guardrails as an input or topical rail. Part of NVIDIA's Safety NIM family.

**First introduced in:** Module 9

---

### Topical Rail

**Definition:** A NeMo Guardrails rail type that restricts the conversation to specific allowed topics and blocks off-topic requests, preventing the agent from engaging with subjects outside its intended domain.

**NVIDIA context:** Defined in Colang 2.0 with topic definitions and off-topic handling flows. Can be augmented with Topic Control NIM for ML-based topic classification.

**First introduced in:** Module 9

---

### Trajectory Evaluation

**Definition:** An evaluation method that assesses the sequence of actions an agent took (tools called, parameters used, order of operations), not just the final output. Catches process errors, inefficiencies, and incorrect reasoning paths that produce correct answers by coincidence.

**NVIDIA context:** Part of the NAT evaluation framework. Test cases define expected trajectories, and the framework compares actual agent trajectories against expectations.

**First introduced in:** Module 8
