# Minimum Real NVIDIA Platform Path

This document defines three accelerated paths to hands-on NVIDIA platform exposure. Each path produces verifiable artifacts — not reading notes, not concept summaries, but actual outputs from real NVIDIA services and tooling.

**Core principle**: Hosted first, local second, self-hosted last.

NVIDIA officially supports trying NIM models through hosted APIs at build.nvidia.com. NeMo Agent Toolkit uses NVIDIA_API_KEY for NVIDIA AI services. NeMo Guardrails works with NVIDIA-hosted models via the same key. Self-hosted NIM requires NGC access, container runtime, and serious GPU hardware — that comes last, if at all.

**Infrastructure honesty**: NVIDIA's Developer Program includes both hosted endpoints (on DGX Cloud) and the ability to download and self-host NIMs on your own infrastructure. These are fundamentally different levels of effort and hardware burden. This document says that plainly at every step.

---

## 3-Day Path: "First Contact"

**Goal**: Prove you can interact with NVIDIA's hosted AI services and run basic NVIDIA tooling locally. Produces 5 artifacts.

### Day 1: NVIDIA Account + Hosted NIM (3-4 hours)

| Step | Task | Hosted/Local | Output |
|------|------|-------------|--------|
| 1 | Create account at build.nvidia.com | Hosted | Account confirmed |
| 2 | Generate NVIDIA_API_KEY | Hosted | Key stored in `~/.bashrc` or `.env` |
| 3 | Explore build.nvidia.com web UI: open a model playground (e.g., meta/llama-3.3-70b-instruct), send 3 prompts, adjust temperature and max_tokens | Hosted | Screenshots of 3 playground interactions with different settings |
| 4 | Make a NIM API call via curl | Hosted | Terminal output: model response + token count |
| 5 | Make the same call via Python (`requests`) | Hosted | Python script + printed response |
| 6 | Make the call via `openai` Python client (NIM is OpenAI-compatible) | Hosted | Python script + printed response |
| 7 | Compare 2 different models on the same prompt (e.g., llama-3.3-70b-instruct vs. mistral-large) | Hosted | Comparison table: response quality, token count, latency |

**Day 1 Artifacts**:
1. `setup_verification.txt` — curl output proving API key works
2. `nim_call.py` — Python script calling NIM endpoint
3. `model_comparison.md` — 2-model comparison table

**Hardware**: CPU laptop. No GPU. No Docker. No containers.

**Cost**: Free tier. All calls use NVIDIA's hosted API credits.

### Day 2: NeMo Agent Toolkit — First Agent (4-5 hours)

| Step | Task | Hosted/Local | Output |
|------|------|-------------|--------|
| 1 | `pip install nvidia-nat` | Local | Version number printed |
| 2 | Write a minimal YAML config: Tool Calling Agent + NVIDIA-hosted LLM + one custom function (`get_current_time`) | Local + Hosted | `workflow.yaml` file |
| 3 | Run the workflow from Python | Local + Hosted | Agent output + execution trace |
| 4 | Modify YAML: switch to ReAct Agent, same function | Local + Hosted | Different execution trace |
| 5 | Modify YAML: switch to Reasoning Agent, same function | Local + Hosted | Third execution trace |
| 6 | Compare all 3 traces: count LLM calls, tokens, latency | Local | Comparison table |
| 7 | Launch NAT console frontend, interact with the agent | Local + Hosted | Screenshot of console session |

**Day 2 Artifacts**:
4. `workflow.yaml` — NAT workflow configuration
5. `agent_comparison.md` — 3-agent-type comparison table with traces

**Hardware**: CPU laptop. NAT runs locally, LLM inference is hosted.

**Cost**: Free tier API calls for LLM inference.

### Day 3: NeMo Guardrails — First Safety Rail (4-5 hours)

| Step | Task | Hosted/Local | Output |
|------|------|-------------|--------|
| 1 | `pip install nemoguardrails langchain-nvidia-ai-endpoints` | Local | Version number printed |
| 2 | Create config directory: `config.yml` + `topic_rail.co` | Local | Config files |
| 3 | Configure `config.yml` to use NVIDIA-hosted model via NVIDIA_API_KEY | Local + Hosted | Working config |
| 4 | Write Colang 2.0 flow: block off-topic requests (only allow technology questions) | Local | `.co` file |
| 5 | Test with `nemoguardrails chat`: send on-topic message → passes | Local + Hosted | Console output showing pass |
| 6 | Test: send off-topic message → blocked | Local + Hosted | Console output showing block |
| 7 | Write an output rail: check for PII in responses | Local | Updated `.co` file |
| 8 | Test PII rail: query that would produce PII → verify redaction or block | Local + Hosted | Before/after comparison |
| 9 | Launch `nemoguardrails server`, test via HTTP (requires `pip install nemoguardrails[server]`) | Local + Hosted | curl output from guardrails server |

**Day 3 Artifacts**:
6. `guardrails_config/` — Complete config directory (config.yml + .co files)
7. `guardrails_test_results.md` — Before/after comparison: unguarded vs. guarded responses

**Hardware**: CPU laptop. Guardrails library runs locally, LLM inference is hosted.

**Cost**: Free tier API calls.

### 3-Day Path Summary

| Artifact # | Artifact | Proves |
|-----------|----------|--------|
| 1 | `setup_verification.txt` | NVIDIA API key works |
| 2 | `nim_call.py` | Can call NIM from Python |
| 3 | `model_comparison.md` | Understand hosted model behavior |
| 4 | `workflow.yaml` | Can configure NAT workflows |
| 5 | `agent_comparison.md` | Can run and compare agent types |
| 6 | `guardrails_config/` | Can write Colang 2.0 rails |
| 7 | `guardrails_test_results.md` | Safety controls actually work |

**What you touched**: build.nvidia.com (web UI + API), NVIDIA NIM (hosted), NeMo Agent Toolkit (local), NeMo Guardrails (local + hosted model)

**What you did NOT touch**: self-hosted NIM, Docker, Kubernetes, NGC, GPU hardware, NeMo Retriever, Milvus, Blueprints. That's fine. This is first contact.

---

## 7-Day Path: "Working Developer"

**Goal**: Build a functional RAG agent with evaluation and safety on the NVIDIA stack. Produces 14 artifacts.

### Days 1-3: Same as 3-Day Path (Artifacts 1-7)

### Day 4: RAG Pipeline on NVIDIA Stack (5-6 hours)

| Step | Task | Hosted/Local | Output |
|------|------|-------------|--------|
| 1 | Prepare 20-30 documents (e.g., NVIDIA technical blog posts as text files) | Local | Document corpus |
| 2 | Implement chunking: try 2 strategies (fixed 512-token, recursive) | Local | Chunked documents |
| 3 | Generate embeddings via NIM embedding endpoint (e.g., nvidia/nv-embedqa-e5-v5) | Hosted | Embedding vectors |
| 4 | Store in FAISS (local, no GPU needed) | Local | FAISS index file |
| 5 | Implement retrieval: query → embed → search → top-k results | Local + Hosted | Retrieved chunks for test queries |
| 6 | Add reranking via NIM reranking endpoint (e.g., nvidia/nv-rerankqa-mistral-4b-v3) | Hosted | Reranked results |
| 7 | Build generation: retrieved context + query → NIM LLM → answer with citations | Hosted | Generated answers |
| 8 | Test with 10 known-answer queries, manually assess retrieval quality | Local | Evaluation table |

**Day 4 Artifacts**:
8. `rag_pipeline.py` — End-to-end RAG pipeline using NIM endpoints
9. `rag_eval_results.md` — 10-query evaluation with retrieval quality scores

**Hardware**: CPU laptop. Embeddings and reranking use hosted NIM endpoints. FAISS runs locally on CPU. For Milvus + cuVS (GPU-accelerated), Docker is needed — this is optional on Day 4.

### Day 5: NAT Pipeline with Middleware and Evaluation (5-6 hours)

| Step | Task | Hosted/Local | Output |
|------|------|-------------|--------|
| 1 | Wrap the RAG pipeline as a NAT function (document_search tool) | Local | NAT function code |
| 2 | Write full NAT YAML config: ReAct Agent + document_search + get_current_time + caching middleware + logging middleware | Local | Complete `pipeline.yaml` |
| 3 | Run the pipeline end-to-end: query → agent reasons → calls tools → answers | Local + Hosted | Execution traces |
| 4 | Write 10 test cases for NAT evaluation framework | Local | Test dataset |
| 5 | Run NAT RAG evaluation on the 10 test cases | Local + Hosted | Evaluation results |
| 6 | Run NAT Profiler on 5 queries: capture token counts and latency per step | Local + Hosted | Profiler output |
| 7 | Identify the slowest step and propose an optimization | Local | Written analysis |

**Day 5 Artifacts**:
10. `pipeline.yaml` — Production NAT config with middleware
11. `eval_results.md` — NAT evaluation output + profiler analysis

### Day 6: Guardrails Integration + Adversarial Testing (4-5 hours)

| Step | Task | Hosted/Local | Output |
|------|------|-------------|--------|
| 1 | Add NeMo Guardrails to the NAT pipeline: configure input rail (content safety) | Local + Hosted | Updated config |
| 2 | Add output rail: hallucination check (verify answer is grounded in retrieved context) | Local | Colang flow |
| 3 | Add execution rail: validate document_search tool inputs | Local | Colang flow |
| 4 | Test with 10 normal queries → all should pass | Local + Hosted | Pass results |
| 5 | Test with 10 adversarial queries (prompt injection, jailbreak, off-topic, PII) → measure block rate | Local + Hosted | Block results |
| 6 | Run NAT red-teaming middleware against the guarded pipeline | Local + Hosted | Red team results |
| 7 | Document: which attacks caught, which slipped through, what to improve | Local | Security assessment |

**Day 6 Artifacts**:
12. `guarded_pipeline_config/` — NAT + Guardrails integrated config
13. `adversarial_test_results.md` — 20-query test results (10 normal + 10 adversarial)

### Day 7: Blueprint Study + Observability (5-6 hours)

| Step | Task | Hosted/Local | Output |
|------|------|-------------|--------|
| 1 | Study the AI-Q Blueprint at build.nvidia.com/blueprints: read the architecture, clone the repo | Local (reading) | Notes |
| 2 | Map your Day 4-6 pipeline against the AI-Q Blueprint architecture: what components match, what differs | Local | Architecture comparison |
| 3 | Identify 3 things the Blueprint does that your pipeline doesn't (e.g., Milvus instead of FAISS, PaddleOCR for document extraction, Nemotron Super 49B for reasoning) | Local | Gap analysis |
| 4 | Install Phoenix: `pip install arize-phoenix` + start server | Local | Phoenix UI running |
| 5 | Configure NAT observability to export to Phoenix | Local | Updated pipeline config |
| 6 | Run 10 queries through the pipeline | Local + Hosted | Traces in Phoenix |
| 7 | Find one trace, annotate: spans, token counts, latency per step | Local | Annotated trace screenshot |
| 8 | Simulate a tool failure, find the error in Phoenix | Local + Hosted | Error trace screenshot |

**Day 7 Artifacts**:
14. `blueprint_comparison.md` — AI-Q Blueprint vs. your pipeline: architecture comparison + gap analysis
15. `observability_traces/` — Phoenix screenshots: normal trace, annotated breakdown, error trace

### 7-Day Path Summary

| Day | Focus | NVIDIA Services Touched | Artifact Count |
|-----|-------|------------------------|----------------|
| 1 | Account + NIM | build.nvidia.com (web + API) | 3 |
| 2 | NAT agents | NAT (local) + NIM (hosted) | 2 |
| 3 | Guardrails | NeMo Guardrails (local) + NIM (hosted) | 2 |
| 4 | RAG pipeline | NIM embedding + reranking + LLM (hosted) | 2 |
| 5 | NAT pipeline + eval | NAT eval + profiler (local) + NIM (hosted) | 2 |
| 6 | Safety integration | NeMo Guardrails + NAT red-teaming (local) + NIM (hosted) | 2 |
| 7 | Blueprint + observability | AI-Q Blueprint (reference) + Phoenix (local) + NIM (hosted) | 2 |
| **Total** | | | **15** |

**What you touched**: build.nvidia.com, NIM API (chat + embedding + reranking), NAT (6+ features), NeMo Guardrails (3 rail types), AI-Q Blueprint, Phoenix observability

**What you did NOT touch**: self-hosted NIM, Kubernetes, NIM Operator, Milvus (used FAISS instead), NeMo Customizer, NeMo Curator, multi-agent A2A. Those are Week 2+ topics.

---

## 14-Day Path: "Production-Ready Practitioner"

**Goal**: Build, evaluate, secure, deploy, and monitor a multi-component agent system on the NVIDIA stack. Produces 25+ artifacts. Covers all 8 milestones.

### Days 1-7: Same as 7-Day Path (Artifacts 1-15)

### Day 8: Multi-Agent System (5-6 hours)

| Step | Task | Hosted/Local | Output |
|------|------|-------------|--------|
| 1 | Design 2 specialist agents: (a) RAG agent (from Days 4-6), (b) code analysis agent with code execution sandbox | Local | Agent designs |
| 2 | Configure NAT Router Agent to dispatch between specialists | Local | Router YAML config |
| 3 | Run Router Agent: send a knowledge query → routes to RAG agent | Local + Hosted | Trace showing routing |
| 4 | Send a code question → routes to code agent | Local + Hosted | Trace showing routing |
| 5 | Send an ambiguous query → observe routing decision | Local + Hosted | Trace + analysis |
| 6 | Profile multi-agent overhead: compare Router dispatch latency vs. direct agent call | Local + Hosted | Latency comparison |

**Day 8 Artifacts**:
16. `multi_agent_config/` — Router + 2 specialists YAML configs
17. `routing_analysis.md` — Routing traces + overhead analysis

### Day 9: Comprehensive Evaluation (5-6 hours)

| Step | Task | Hosted/Local | Output |
|------|------|-------------|--------|
| 1 | Create 30+ test cases across both specialist domains | Local | Test dataset |
| 2 | Run NAT trajectory evaluation: verify agents use correct tool sequences | Local + Hosted | Trajectory results |
| 3 | Build a custom LLM-as-Judge evaluator using NAT's custom evaluator path | Local + Hosted | Evaluator code |
| 4 | Run NAT red teaming against the multi-agent system | Local + Hosted | Red team report |
| 5 | Use NAT Sizing Calculator to estimate production resource needs | Local | Sizing report |
| 6 | Build cost/latency/accuracy tradeoff: test with 2 different LLM configurations | Local + Hosted | Tradeoff analysis |

**Day 9 Artifacts**:
18. `evaluation_suite/` — Test cases + custom evaluator + trajectory results
19. `production_readiness.md` — Sizing + tradeoff analysis + red team findings

### Day 10: Full Safety Stack (5-6 hours)

| Step | Task | Hosted/Local | Output |
|------|------|-------------|--------|
| 1 | Configure NeMo Guardrails for the multi-agent system: input rails on Router, execution rails on specialists | Local + Hosted | Multi-layer guardrails config |
| 2 | Add PII detection (Presidio integration) to output rails | Local | PII rail config |
| 3 | Configure topic control: each specialist only answers in its domain | Local + Hosted | Topic rails |
| 4 | Run 30 adversarial tests: prompt injection, cross-agent manipulation, PII extraction, jailbreak | Local + Hosted | Adversarial results |
| 5 | Configure NAT defense middleware on all agents | Local | Middleware config |
| 6 | Document the safety architecture: which rail protects what, coverage gaps | Local | Safety architecture doc |

**Day 10 Artifacts**:
20. `safety_config/` — Multi-agent guardrails configuration
21. `safety_assessment.md` — Adversarial test results + architecture doc + coverage gaps

### Day 11: Deployment (5-6 hours)

| Step | Task | Hosted/Local | Output |
|------|------|-------------|--------|
| 1 | Package the Router Agent as a NAT FastAPI Server | Local | Server code + config |
| 2 | Write Dockerfile for the agent service | Local | Dockerfile |
| 3 | Write Docker Compose: agent service + Redis (memory) + FAISS/Milvus (RAG) | Local | docker-compose.yml |
| 4 | `docker compose up` — verify all services start | Local (Docker) | Running containers |
| 5 | Test via HTTP: send queries to the deployed agent endpoint | Local | curl/httpx test results |
| 6 | Add health checks and readiness probes | Local | Updated Dockerfile/Compose |
| 7 | Load test: 5 concurrent users with a simple script | Local | Latency distribution |

**Day 11 Artifacts**:
22. `deployment/` — Dockerfile + docker-compose.yml + health check config
23. `load_test_results.md` — Latency distribution under concurrent load

**Hardware**: Docker required. No GPU needed (LLM inference still uses hosted NIM). For self-hosted NIM in the Docker Compose, GPU required — this is OPTIONAL and clearly marked.

### Day 12: Observability at Scale (4-5 hours)

| Step | Task | Hosted/Local | Output |
|------|------|-------------|--------|
| 1 | Configure NAT + NeMo Guardrails to export traces to Phoenix | Local | Observability config |
| 2 | Run 30 queries through the deployed system | Local + Hosted | 30 traces in Phoenix |
| 3 | Build a token cost dashboard: tokens per request, by agent, by tool | Local | Dashboard screenshot |
| 4 | Build a latency dashboard: p50/p95/p99, by component | Local | Dashboard screenshot |
| 5 | Configure PII redaction in telemetry | Local | Redaction config |
| 6 | Simulate 3 failures (LLM timeout, tool error, guardrail false positive), debug from traces | Local + Hosted | Debug writeup for each |
| 7 | Write 5 alerting rules | Local | Alerting rules doc |
| 8 | Write a 1-page production runbook: "How to investigate an agent failure" | Local | Runbook |

**Day 12 Artifacts**:
24. `observability/` — Dashboard screenshots + redaction config + alerting rules
25. `runbook.md` — Production failure investigation runbook

### Day 13: Human Oversight + Capstone Polish (5-6 hours)

| Step | Task | Hosted/Local | Output |
|------|------|-------------|--------|
| 1 | Add execution rail: human approval required before code execution tool runs | Local + Hosted | Colang execution rail |
| 2 | Configure NAT interactive workflow: WebSocket pause for approval | Local | Interactive config |
| 3 | Test: query triggers code execution → agent pauses → approve → executes | Local + Hosted | Approval flow output |
| 4 | Test: reject the approval → verify code does NOT execute | Local + Hosted | Rejection flow output |
| 5 | Configure logging middleware for audit trail | Local | Middleware config |
| 6 | Verify audit completeness: reconstruct full decision chain from logs | Local | Audit trail printout |
| 7 | Polish capstone: ensure eval results are documented, safety controls verified, deployment working, observability configured | Local | Capstone README |

**Day 13 Artifacts**:
26. `human_oversight/` — Approval rail + interactive config + audit trail
27. `capstone_README.md` — Complete capstone documentation

### Day 14: Integration Review + Exam Prep (4-5 hours)

| Step | Task | Hosted/Local | Output |
|------|------|-------------|--------|
| 1 | Run the quality gate checklist (quality_gate.md): verify all items are YES | N/A | Completed checklist |
| 2 | Verify all 8 milestones are complete with artifacts | N/A | Milestone verification |
| 3 | Complete the Module 13 mini mock exam (20 questions) | N/A | Score + weak areas |
| 4 | For each weak area: review the relevant module's "How this may appear on the exam" section | N/A | Study notes |
| 5 | Complete 10 of the hardest conceptual questions from mock_exam_prep.md | N/A | Score + explanations |
| 6 | Practice oral explanations: pick 3 exam domains at random, explain each for 3 minutes | N/A | Self-assessment |

**Day 14 Artifacts**:
28. `quality_gate_completed.md` — All checkboxes verified
29. `exam_prep_scores.md` — Mock exam scores + weak area remediation notes

### 14-Day Path Summary

| Day | Focus | New NVIDIA Interaction | Cumulative Artifacts |
|-----|-------|----------------------|---------------------|
| 1 | Account + NIM | build.nvidia.com, NIM API | 3 |
| 2 | NAT agents | NAT (3 agent types) | 5 |
| 3 | Guardrails | NeMo Guardrails, Colang 2.0 | 7 |
| 4 | RAG | NIM embedding + reranking | 9 |
| 5 | NAT pipeline + eval | NAT eval + profiler + middleware | 11 |
| 6 | Safety | Guardrails integration + red teaming | 13 |
| 7 | Blueprint + observability | AI-Q Blueprint, Phoenix | 15 |
| 8 | Multi-agent | NAT Router Agent | 17 |
| 9 | Evaluation at scale | Trajectory eval, custom evaluators, Sizing Calculator | 19 |
| 10 | Full safety | Multi-layer guardrails, PII, topic control | 21 |
| 11 | Deployment | NAT FastAPI Server, Docker Compose | 23 |
| 12 | Observability at scale | Distributed tracing, dashboards, alerting | 25 |
| 13 | Human oversight | Execution rails, interactive workflows, audit | 27 |
| 14 | Integration + exam prep | Quality gate verification | 29 |

---

## What Requires What — Hardware Honesty Table

| Activity | CPU Laptop | Docker Required | GPU Recommended | GPU Required |
|----------|-----------|-----------------|-----------------|--------------|
| build.nvidia.com account + API key | Yes | No | No | No |
| NIM API calls (hosted) | Yes | No | No | No |
| NAT installation and workflows | Yes | No | No | No |
| NeMo Guardrails (library mode — all labs use this) | Yes | No | No | No |
| Phoenix observability | Yes | No | No | No |
| FAISS vector store | Yes | No | No | No |
| Redis for NAT memory | Yes | Yes | No | No |
| Milvus vector store | Yes | Yes | No (cuVS needs GPU) | No |
| Docker Compose deployment | Yes | Yes | No | No |
| Self-hosted NIM (small model, 7-13B) | No | Yes | No | Yes (1x A100 40GB min) |
| Self-hosted NIM (production model, 70B) | No | Yes | No | Yes (2x H100 80GB min) |
| Full AI-Q Blueprint deployment | No | Yes | No | Yes (2x H100 80GB min for AI-Q; verify requirements on specific Blueprint page — they vary significantly across Blueprints) |
| NIM Operator / Kubernetes | No | Yes | No | Yes (GPU cluster) |
| NeMo Customizer (fine-tuning) | No | Yes | No | Yes (multi-GPU) |
| NeMo Curator (GPU data prep) | No | Yes | No | Yes |

**Bottom line**: Days 1-10 of the 14-day path work on a CPU laptop with an internet connection. Days 11-12 need Docker. Days 13-14 need nothing extra. Self-hosted NIM and full Blueprint deployment are clearly marked optional throughout.

---

## Milestone Mapping to Paths

| Milestone | 3-Day | 7-Day | 14-Day |
|-----------|-------|-------|--------|
| 1: First NIM call | Day 1 | Day 1 | Day 1 |
| 2: First NAT workflow | Day 2 | Day 2 | Day 2 |
| 3: Agent pattern comparison | Day 2 | Day 2 | Day 2 |
| 4: RAG on NVIDIA stack | — | Day 4 | Day 4 |
| 5: Guardrails in action | Day 3 | Day 6 | Day 6 |
| 6: Blueprint reproduction | — | Day 7 | Day 7 |
| 7: Observable agent system | — | Day 7 | Day 12 |
| 8: Integrated capstone | — | — | Day 13 |

The 3-day path hits 3 of 8 milestones. The 7-day path hits 6 of 8. The 14-day path hits all 8.
