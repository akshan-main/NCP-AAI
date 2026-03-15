# NCP-AAI Exam Domain Mapping

## Exam Overview (Source: nvidia.com/certification/agentic-ai-professional)

| Field | Value |
|-------|-------|
| Duration | 120 minutes |
| Questions | 60-70 |
| Cost | $200 |
| Delivery | Online, remotely proctored |
| Language | English |
| Validity | 2 years |
| Level | Professional (intermediate) |

## Exam Domains and Course Coverage

### Domain 1: Agent Architecture and Design (15%)

**Competency cluster**: Agent Design and Cognition — architect agents, apply reasoning/planning, manage memory, coordinate multi-agent workflows.

| Coverage Element | Module | Depth | Source Confidence |
|-----------------|--------|-------|-------------------|
| Architecture pattern taxonomy (ReAct, Plan-and-Execute, router, hierarchical) | M2 | Primary | Mixed — standard literature mapped to NAT agent types |
| NAT agent types (ReAct, Reasoning, ReWOO, Router, Sequential, Tool Calling, Responses API) | M2 | Primary | High — NAT v1.5 docs |
| Single-agent vs. multi-agent design decisions | M2, M7 | Primary | Mixed — NVIDIA Blueprints provide reference, design criteria are inferred |
| Reference architectures from NVIDIA Blueprints | M2 | Primary | High — build.nvidia.com/blueprints |
| Multi-agent coordination patterns | M7 | Primary | Mixed — Warehouse Blueprint + A2A protocol documented; coordination patterns inferred |
| Architecture tradeoffs (latency, accuracy, cost, complexity) | M2 | Primary | Inference — NVIDIA does not publish a tradeoff framework |

**Labs**: Lab 2 (implement 3+ NAT agent types), Lab 7 (multi-agent coordination)
**Coverage confidence**: 85% — strong on NVIDIA-specific agent types; weaker on formal architecture decision frameworks which NVIDIA does not document.

---

### Domain 2: Agent Development (15%)

**Competency cluster**: Knowledge Integration and Agent Development — implement retrieval pipelines, handle data, engineer prompts, build multimodal and reliable agents.

| Coverage Element | Module | Depth | Source Confidence |
|-----------------|--------|-------|-------------------|
| NAT YAML configuration and workflow building | M6 | Primary | High — NAT v1.5 docs |
| NAT Functions, Function Groups, Per-User Functions | M5, M6 | Primary | High — NAT v1.5 docs |
| Tool registry and MCP client support | M5, M6 | Primary | High — NAT v1.5 docs |
| NAT middleware (caching, defense, logging, red teaming) | M6 | Primary | High — NAT v1.5 docs |
| NAT plugin system (LangChain, LlamaIndex, CrewAI, AutoGen, Semantic Kernel, etc.) | M6 | Primary | High — NAT v1.5 docs |
| Prompt engineering and NAT Parameter Optimizer | M5 | Primary | High — NAT v1.5 docs |
| NAT authentication (API Key, OAuth2, Bearer, HTTP Basic, MCP accounts) | M6 | Secondary | High — NAT v1.5 docs |
| Structured outputs and function calling | M5 | Primary | Mixed — standard patterns + NAT Tool Calling Agent |

**Labs**: Lab 5 (tool-using agent), Lab 6 (NAT pipeline end-to-end)
**Coverage confidence**: 90% — NAT v1.5 docs are comprehensive for development workflows.

---

### Domain 3: Evaluation and Tuning (13%)

**Competency cluster**: Evaluation, Monitoring, and Maintenance — benchmark/tune performance, monitor live systems, troubleshoot, ensure continuous improvement.

| Coverage Element | Module | Depth | Source Confidence |
|-----------------|--------|-------|-------------------|
| NAT Evaluation Framework (RAG eval, red teaming, trajectory eval, custom evaluators) | M8 | Primary | High — NAT v1.5 docs |
| NAT Profiler (workflow-to-tool-level token/latency profiling) | M8 | Primary | High — NAT v1.5 docs |
| NAT Sizing Calculator | M8 | Secondary | High — NAT v1.5 docs |
| Retriever evaluation metrics (MRR, NDCG, recall@k) | M8 | Primary | Mixed — standard IR metrics applied to NeMo Retriever |
| RAG evaluation metrics (faithfulness, relevance, context precision/recall) | M8 | Primary | Mixed — RAGAS-style metrics implemented via NAT eval framework |
| LLM-as-Judge patterns | M8 | Primary | Mixed — standard pattern, NAT custom evaluator path |
| Agent trajectory evaluation | M8 | Primary | High — NAT v1.5 docs confirm trajectory evaluation |
| Benchmark vs. custom evaluation | M8 | Primary | Mixed — NAT dataset loading documented; benchmark selection inferred |
| Online monitoring vs. offline evaluation | M8, M11 | Primary | Mixed — NAT profiler for offline; online patterns inferred |
| NAT Finetuning (DPO with NeMo Customizer, OpenPipe ART) | M8 | Secondary | High — NAT v1.5 docs |
| NeMo Evaluator microservice | M8 | Secondary | Medium — documented at high level in NeMo docs |
| Data Flywheel Blueprint (continuous optimization) | M8 | Secondary | Medium — Blueprint exists, implementation details may be thin |
| Cost/latency/accuracy tradeoff framework | M8 | Primary | Inference — no NVIDIA framework; course provides decision matrix |

**Labs**: Lab 8 (evaluation pipeline), Lab 9 (safety evaluation overlaps)
**Coverage confidence**: 80% — NAT eval framework is well-documented; NeMo Evaluator microservice is thinner. Online monitoring methodology is inferred.

---

### Domain 4: Deployment and Scaling (13%)

**Competency cluster**: NVIDIA Platform Implementation and Deployment — use NVIDIA tools for inference optimization, production-scale deployment, workflow management.

| Coverage Element | Module | Depth | Source Confidence |
|-----------------|--------|-------|-------------------|
| NAT deployment servers (FastAPI, MCP, FastMCP, A2A) | M10 | Primary | High — NAT v1.5 docs |
| NIM microservices for model serving | M10 | Primary | High — NeMo docs |
| NIM Operator for Kubernetes | M10 | Primary | High — NeMo docs |
| NVIDIA Dynamo for inference acceleration | M10 | Secondary | Medium — referenced in NAT docs but agent-specific usage is thin |
| NeMo Export and Deploy | M10 | Secondary | Medium — NeMo docs |
| Containerization and Docker Compose | M10 | Primary | High — AI-Q Blueprint provides reference (2xH100 documented for AI-Q specifically; always verify requirements on the specific Blueprint page as they vary significantly across Blueprints) |
| Kubernetes/Helm deployment patterns | M10 | Primary | Mixed — NIM Operator documented; K8s patterns for agents are operational inference |
| Scaling strategies (horizontal/vertical for agent systems) | M10 | Primary | Inference — NVIDIA does not publish agent-specific scaling patterns |
| GPU resource planning | M10 | Secondary | Medium — AI-Q Blueprint documents requirements; general planning inferred |
| Latency optimization | M10 | Primary | Mixed — Dynamo documented; agent-level latency optimization partially inferred |

**Labs**: Lab 10 (NIM deployment), Lab 11 (monitoring for deployed systems)
**Coverage confidence**: 78% — NAT deployment servers and NIM are well-documented. Agent-specific scaling patterns and Dynamo usage for agents require inference.

---

### Domain 5: Cognition, Planning, and Memory (10%)

**Competency cluster**: Agent Design and Cognition.

| Coverage Element | Module | Depth | Source Confidence |
|-----------------|--------|-------|-------------------|
| NAT Reasoning Agent | M3 | Primary | High — NAT v1.5 docs |
| NAT ReWOO Agent | M3 | Primary | High — NAT v1.5 docs |
| NAT Test Time Compute (multi-LLM generation, planning, selection) | M3 | Primary | Medium — documented but internal algorithms not specified |
| NAT Memory systems | M3 | Primary | High — NAT v1.5 docs |
| NAT Object Stores (in-memory, MySQL, Redis, S3) | M3 | Primary | High — NAT v1.5 docs |
| NAT Automatic Memory Wrapper | M3 | Primary | High — NAT v1.5 docs |
| Reasoning strategies (CoT, ToT, self-reflection) | M3 | Primary | Inference — standard literature; NAT implements these via agent types |
| Planning algorithms | M3 | Primary | Inference — NAT's planning is documented at feature level, not algorithm level |
| Memory architecture patterns (episodic, semantic, procedural) | M3 | Primary | Inference — NAT provides backends; architecture patterns from literature |

**Labs**: Lab 3 (planning + memory integration)
**Coverage confidence**: 75% — NAT memory/reasoning tools documented; planning internals and memory architecture patterns require inference.

---

### Domain 6: Knowledge Integration and Data Handling (10%)

**Competency cluster**: Knowledge Integration and Agent Development.

| Coverage Element | Module | Depth | Source Confidence |
|-----------------|--------|-------|-------------------|
| NeMo Retriever (RAG pipeline microservice) | M4 | Primary | High — NeMo docs |
| NeMo Curator (GPU-accelerated data prep) | M4 | Primary | High — NeMo docs |
| NAT retrievers and embedders | M4 | Primary | High — NAT v1.5 docs |
| AI-Q Blueprint RAG architecture | M4 | Primary | High — build.nvidia.com/nvidia/aiq |
| Embedding models (Llama 3.2 NV EmbedQA) | M4 | Primary | High — AI-Q Blueprint |
| Reranking (Llama 3.2 NV RerankQA) | M4 | Primary | High — AI-Q Blueprint |
| Vector stores (Milvus + cuVS) | M4 | Primary | High — AI-Q Blueprint |
| Chunking strategies | M4 | Primary | Mixed — standard techniques; applied to NeMo Retriever context |
| Document processing (PaddleOCR) | M4 | Secondary | High — AI-Q Blueprint |

**Labs**: Lab 4 (RAG pipeline with NeMo Retriever)
**Coverage confidence**: 90% — strongest NVIDIA documentation coverage. AI-Q Blueprint provides concrete reference architecture.

---

### Domain 7: NVIDIA Platform Implementation (7%)

**Competency cluster**: NVIDIA Platform Implementation and Deployment.

| Coverage Element | Module | Depth | Source Confidence |
|-----------------|--------|-------|-------------------|
| NAT end-to-end workflow | M6 | Primary | High — NAT v1.5 docs |
| NIM API usage | M1, M10 | Primary | High — build.nvidia.com |
| NeMo microservices (Customizer, Evaluator, Retriever, Guardrails) | M4, M8, M9 | Distributed | High — NeMo docs |
| NVIDIA Blueprint implementation | M2, M4, M7 | Distributed | High — build.nvidia.com/blueprints |
| NAT plugin system for third-party frameworks | M6 | Primary | High — NAT v1.5 docs |

**Labs**: Lab 6 (NAT pipeline), Lab 10 (NIM deployment)
**Coverage confidence**: 90% — NVIDIA platform is distributed across all modules with direct documentation.

---

### Domain 8: Run, Monitor, and Maintain (5%)

| Coverage Element | Module | Depth | Source Confidence |
|-----------------|--------|-------|-------------------|
| NAT observability (Phoenix, Weave, Langfuse, OpenTelemetry) | M11 | Primary | High — NAT v1.5 docs |
| NAT execution tracing (workflow → tool level) | M11 | Primary | High — NAT v1.5 docs |
| Token counting and latency metrics | M11 | Primary | High — NAT v1.5 docs |
| NAT redaction processors | M11 | Secondary | High — NAT v1.5 docs |
| Custom telemetry exporters | M11 | Secondary | High — NAT v1.5 docs |
| NeMo Guardrails structured logging and tracing | M11 | Secondary | High — Guardrails docs |
| Dashboard design and alerting | M11 | Primary | Inference — NAT exports telemetry; dashboard specifics are operational |
| Maintenance runbooks and troubleshooting | M11 | Primary | Inference — operational best practice |

**Labs**: Lab 11 (monitoring + observability)
**Coverage confidence**: 75% — NAT telemetry export is well-documented; monitoring dashboard design and maintenance practices are inferred.

---

### Domain 9: Safety, Ethics, and Compliance (5%)

| Coverage Element | Module | Depth | Source Confidence |
|-----------------|--------|-------|-------------------|
| NeMo Guardrails (Colang 2.0, 5 rail types, LLMRails API, CLI, deployment) | M9 | Primary | High — Guardrails docs + GitHub |
| Content Safety NIM | M9 | Primary | High — NVIDIA blog |
| Jailbreak Detection NIM | M9 | Primary | High — NVIDIA blog |
| Topic Control NIM | M9 | Primary | High — NVIDIA blog |
| NeMo Auditor (pre-deployment auditing) | M9 | Primary | Medium — documented as replacement for deprecated Safety Blueprint |
| NAT defense middleware | M9 | Primary | High — NAT v1.5 docs |
| NAT red-teaming middleware | M9 | Primary | High — NAT v1.5 docs |
| PII detection (GLiNER, Presidio, PrivateAI) | M9 | Secondary | High — Guardrails docs |
| Agentic security (tool call validation via execution rails) | M9 | Primary | High — Guardrails docs |
| Fact-checking rails | M9 | Secondary | High — Guardrails docs |
| 4-stage safety lifecycle (conceptual frame only) | M9 | Framing | Medium — deprecated Safety Blueprint used as lens, not implementation |
| Safe Synthesizer | M9 | Optional/Advanced | Low — Early Access, documented but not production-ready |
| Ethics and compliance frameworks | M9, M12 | Light | Inference — policy/regulatory, not NVIDIA-documented |

**Labs**: Lab 9 (guardrails configuration + safety testing)
**Coverage confidence**: 88% — NeMo Guardrails is extensively documented. NeMo Auditor is newer and documentation may be thinner. Ethics/compliance are inherently non-NVIDIA.

---

### Domain 10: Human-AI Interaction and Oversight (5%)

| Coverage Element | Module | Depth | Source Confidence |
|-----------------|--------|-------|-------------------|
| NAT interactive workflows (WebSocket, bidirectional) | M12 | Primary | High — NAT v1.5 docs |
| NeMo Guardrails execution rails (tool call validation/approval) | M12 | Primary | High — Guardrails docs |
| NAT per-user functions and authentication | M12 | Secondary | High — NAT v1.5 docs |
| NAT middleware for audit trail | M12 | Secondary | High — NAT v1.5 docs |
| Human-in-the-loop design patterns | M12 | Primary | Inference — NVIDIA provides primitives, not patterns |
| Escalation path design | M12 | Primary | Inference — operational best practice |
| Feedback loop design | M12 | Primary | Inference — operational best practice |
| Approval workflow architecture | M12 | Primary | Inference — built on NAT primitives |

**Labs**: Lab 12 (human-in-the-loop integration)
**Coverage confidence**: 65% — NVIDIA provides implementation primitives (interactive workflows, execution rails, auth). Design patterns for HITL are entirely inferred. This is the weakest domain for NVIDIA-specific content. The course compensates by anchoring every pattern to an NVIDIA primitive.

---

## Coverage Confidence Summary

| Domain | Weight | Coverage Confidence | Risk Level |
|--------|--------|-------------------|------------|
| Agent Architecture and Design | 15% | 85% | Low |
| Agent Development | 15% | 90% | Low |
| Evaluation and Tuning | 13% | 80% | Medium |
| Deployment and Scaling | 13% | 78% | Medium |
| Cognition, Planning, and Memory | 10% | 75% | Medium |
| Knowledge Integration | 10% | 90% | Low |
| NVIDIA Platform Implementation | 7% | 90% | Low |
| Run, Monitor, and Maintain | 5% | 75% | Medium |
| Safety, Ethics, and Compliance | 5% | 88% | Low |
| Human-AI Interaction | 5% | 65% | High |

**Weighted average coverage confidence**: ~83%

**Interpretation**: The course has strong coverage (85%+) for 62% of exam weight (Architecture, Development, Knowledge, Platform, Safety). Medium coverage (75-80%) for 31% (Evaluation, Deployment, Cognition, Monitoring). The weakest area (Human-AI at 5%) is low-weight. The primary risk areas for a student are Deployment scaling patterns and Evaluation methodology, where NVIDIA documentation is thinner.
