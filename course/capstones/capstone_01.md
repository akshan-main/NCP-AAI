# Capstone 1: Production RAG Agent System

**Builds on:** Labs 4-6 (RAG + agent + orchestration), Labs 8-9 (evaluation + safety)
**Estimated time:** 8-10 hours
**Exam domains exercised:** Knowledge Integration (10%), Agent Development (15%), Evaluation (13%), Safety (5%), Platform (7%)

---

## 1. Problem Statement

Build a production-quality document research assistant that can:

1. **Ingest** a corpus of technical documentation (NVIDIA docs, API references, research papers) through an automated pipeline.
2. **Answer questions** with citations — every claim in the response must reference a specific source passage with document name and section.
3. **Evaluate its own accuracy** using an automated evaluation suite that runs on every pipeline change.
4. **Enforce safety guardrails** that prevent hallucination, block adversarial inputs, redact PII, and validate all tool calls before execution.

The target domain is technical documentation. A realistic corpus is 500-2,000 documents (PDF, Markdown, HTML) totaling 10,000-50,000 pages. The system must answer questions accurately, refuse questions outside its domain, and degrade gracefully when retrieval fails.

This is not a chatbot demo. This is a system that a documentation team could deploy internally and trust to answer questions from engineers who will immediately notice wrong answers.

---

## 2. System Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        PRODUCTION RAG AGENT SYSTEM                       │
│                                                                          │
│  ┌─────────────┐    ┌──────────────────────────────────────────────────┐ │
│  │   Document   │    │              RUNTIME PIPELINE                    │ │
│  │   Ingestion  │    │                                                  │ │
│  │   Pipeline   │    │  ┌──────────┐   ┌───────────┐   ┌───────────┐  │ │
│  │              │    │  │  INPUT    │   │ RETRIEVAL │   │ GENERATION│  │ │
│  │ ┌──────────┐│    │  │  RAILS    │   │           │   │           │  │ │
│  │ │  NeMo    ││    │  │          ├──►│ NAT       ├──►│ NIM LLM   │  │ │
│  │ │  Curator  ││    │  │ Guardrails│  │ Retriever │   │ Endpoint  │  │ │
│  │ │(preproc) ││    │  │ + Safety │   │ + Reranker│   │ (Llama 3) │  │ │
│  │ └────┬─────┘│    │  │   NIM    │   │           │   │           │  │ │
│  │      │      │    │  └──────────┘   └─────┬─────┘   └─────┬─────┘  │ │
│  │ ┌────▼─────┐│    │                       │               │         │ │
│  │ │  NIM     ││    │                  ┌────▼────┐    ┌─────▼──────┐  │ │
│  │ │ Embedding││    │                  │ Milvus  │    │  OUTPUT    │  │ │
│  │ │ Endpoint ││    │                  │ + cuVS  │    │  RAILS     │  │ │
│  │ └────┬─────┘│    │                  │ (Vector │    │            │  │ │
│  │      │      │    │                  │  Store) │    │ Halluc.    │  │ │
│  │ ┌────▼─────┐│    │                  └─────────┘    │ Check +    │  │ │
│  │ │  Milvus  ││    │                                 │ PII Redact │  │ │
│  │ │  Index   ││    │                                 └─────┬──────┘  │ │
│  │ └──────────┘│    │                                       │         │ │
│  └─────────────┘    │                                 ┌─────▼──────┐  │ │
│                     │                                 │  EXECUTION │  │ │
│                     │                                 │  RAILS     │  │ │
│                     │                                 │ (tool call │  │ │
│                     │                                 │ validation)│  │ │
│                     │                                 └─────┬──────┘  │ │
│                     └───────────────────────────────────────┼────────┘ │
│                                                             │          │
│  ┌──────────────────────────────────────────────────────────▼────────┐ │
│  │                    EVALUATION & OBSERVABILITY                      │ │
│  │  NAT Eval Framework │ NAT Profiler │ Phoenix/Langfuse Dashboards  │ │
│  └───────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Data Flow

### Ingestion Flow (Offline)

1. **Document Upload**: Raw documents (PDF, Markdown, HTML) are placed in an input directory or uploaded via API.
2. **Preprocessing (NeMo Curator)**: GPU-accelerated document processing — OCR for scanned PDFs (PaddleOCR), text extraction, deduplication, language filtering, quality scoring.
3. **Chunking**: Adaptive chunking strategy — document-structure-aware chunking for well-structured docs (Markdown headers, API reference sections), recursive character splitting with overlap for unstructured text. Target chunk size: 512 tokens with 64-token overlap.
4. **Embedding (NIM Endpoint)**: Each chunk is embedded using Llama 3.2 NV EmbedQA via a NIM endpoint. Batch processing for efficiency.
5. **Indexing (Milvus + cuVS)**: Embeddings are stored in Milvus with HNSW index. Metadata (document name, section, page number, chunk position) stored alongside vectors for citation generation.

### Query Flow (Runtime)

1. **User Query Received**: Query arrives via NAT FastAPI Server endpoint.
2. **Input Rails (NeMo Guardrails)**:
   - Content Safety NIM checks for harmful content.
   - Jailbreak Detection NIM screens for prompt injection attempts.
   - Topical rail verifies query is within the documentation domain.
3. **Query Embedding**: Query is embedded using the same NIM embedding endpoint.
4. **Retrieval (NAT Retriever)**: Top-20 candidates retrieved from Milvus via dense similarity search. Optional: hybrid retrieval with BM25 sparse index for keyword-heavy queries.
5. **Reranking (NIM Reranker)**: Llama 3.2 NV RerankQA reranks the 20 candidates. Top-5 passages are selected as context.
6. **Generation (NIM LLM)**: Llama 3 (or comparable model) generates an answer with explicit citation instructions in the system prompt. The prompt requires the model to cite specific passages by document name and section.
7. **Output Rails (NeMo Guardrails)**:
   - Hallucination check: fact-checking rail verifies claims against retrieved passages.
   - PII Redaction: redaction processor scans output for PII patterns and masks them.
8. **Execution Rails**: If the agent invokes any tools (e.g., follow-up retrieval, calculation), execution rails validate the tool call parameters before execution.
9. **Response Delivery**: Final answer with citations returned to the user.

---

## 4. Components

| Component | NVIDIA Tool | Role | Why This Choice |
|-----------|-------------|------|-----------------|
| Document preprocessing | NeMo Curator | GPU-accelerated cleaning, dedup, quality filtering | Handles scale (millions of docs); GPU acceleration is 10-100x faster than CPU-based preprocessing |
| OCR | PaddleOCR (via AI-Q Blueprint) | Extract text from scanned PDFs | Proven integration in NVIDIA's reference architecture |
| Embedding | NIM Embedding Endpoint (Llama 3.2 NV EmbedQA) | Convert text to dense vectors | Purpose-built for retrieval; optimized for NIM serving with low latency |
| Vector store | Milvus + cuVS | Store and search embeddings | cuVS provides GPU-accelerated similarity search; Milvus is production-grade with filtering, partitioning |
| Retrieval | NAT Retriever | Orchestrate retrieval pipeline | Integrates with NAT agent workflow; supports hybrid retrieval configuration |
| Reranking | NIM Reranker (Llama 3.2 NV RerankQA) | Cross-encoder reranking of candidates | Cross-encoder accuracy far exceeds bi-encoder for final ranking; NIM serving keeps latency low |
| LLM generation | NIM LLM Endpoint (Llama 3) | Generate cited answers | NIM provides optimized inference with TensorRT-LLM; consistent API across model sizes |
| Input guardrails | NeMo Guardrails + Content Safety NIM + Jailbreak Detection NIM | Block harmful/adversarial inputs | Multi-layered defense: Colang rules for domain control, NIMs for content safety. Library mode (`pip install nemoguardrails langchain-nvidia-ai-endpoints`) is sufficient for development; microservice mode (NGC container) is for production deployments. |
| Output guardrails | NeMo Guardrails (fact-checking rail + redaction processor) | Prevent hallucination and PII leakage | Configurable via Colang 2.0; integrates PII detection (GLiNER/Presidio) |
| Execution guardrails | NeMo Guardrails (execution rails) | Validate tool calls before execution | Prevents agent from executing unauthorized operations |
| Agent orchestration | NeMo Agent Toolkit (NAT) | Coordinate the full pipeline | YAML-configurable; supports tool registry, middleware, deployment servers |
| Evaluation | NAT Evaluation Framework | Automated quality assessment | Built-in RAG metrics, red teaming, custom evaluators |
| Observability | NAT Observability (Phoenix/Langfuse export) | Runtime monitoring and tracing | OpenTelemetry-based; traces from workflow to tool level |
| API server | NAT FastAPI Server | Serve the agent as an API | Production-ready server with authentication, middleware support |

---

## 5. Evaluation Plan

### 5.1 Retriever Evaluation

| Metric | Target | How Measured |
|--------|--------|-------------|
| Recall@5 | >= 0.85 | NAT eval framework with a curated Q&A test set (100+ question-passage pairs). For each question, check if the correct passage appears in the top-5 retrieved results. |
| Precision@5 | >= 0.70 | Of the 5 retrieved passages, what fraction is relevant to the query. Measured via LLM-as-Judge relevance scoring. |
| MRR (Mean Reciprocal Rank) | >= 0.80 | Reciprocal rank of the first relevant passage, averaged across test set. |
| Latency (retrieval only) | p95 < 500ms | NAT Profiler measurement of retrieval step in isolation. |

**Test set construction**: Manually curate 100-200 question-passage pairs from the corpus. Include easy questions (answer in a single passage), hard questions (answer requires synthesizing 2-3 passages), and adversarial questions (similar-sounding but different topics). Use NAT eval framework's dataset loading to manage the test set.

### 5.2 RAG (End-to-End) Evaluation

| Metric | Target | How Measured |
|--------|--------|-------------|
| Faithfulness | >= 0.90 | NAT eval framework: does every claim in the answer have support in the retrieved passages? LLM-as-Judge scoring. |
| Answer Relevance | >= 0.85 | NAT eval framework: does the answer address the question? LLM-as-Judge scoring. |
| Context Precision | >= 0.75 | Are the retrieved passages actually useful for answering? Measures retrieval quality from the generation perspective. |
| Context Recall | >= 0.80 | Does the retrieved context contain all the information needed for a complete answer? |
| Citation Accuracy | >= 0.90 | Custom evaluator: for each citation in the answer, verify it points to a real passage that supports the claim. |

### 5.3 Adversarial/Safety Evaluation

| Metric | Target | How Measured |
|--------|--------|-------------|
| Jailbreak resistance | Survive 95% of attacks | NAT red-teaming middleware with standard attack library (prompt injection, role-play, encoding attacks). |
| Off-topic rejection | >= 95% correct rejection | Test with 50+ off-topic queries; measure how many are correctly refused by topical rails. |
| PII redaction | 100% redaction of test PII | Inject known PII patterns into test documents; verify redaction processor catches all. |
| Hallucination rate | < 5% | Generate answers for 100 questions with known correct answers; measure hallucination via fact-checking rail. |

### 5.4 Latency Evaluation

| Metric | Target | How Measured |
|--------|--------|-------------|
| End-to-end p50 | < 3 seconds | NAT Profiler across full pipeline. |
| End-to-end p95 | < 5 seconds | NAT Profiler, simple single-retrieval queries. |
| End-to-end p95 (complex) | < 10 seconds | Multi-step retrieval queries requiring follow-up searches. |

### 5.5 Production-Readiness Criteria

The system is production-ready when ALL of the following are met:
- All retriever metrics at target thresholds
- All RAG metrics at target thresholds
- Adversarial survival rate >= 95%
- p95 latency within budget
- Evaluation suite runs in CI/CD on every pipeline change
- No regressions on any metric for 3 consecutive evaluation runs

---

## 6. Safety and Governance

### 6.1 Input Rails

| Rail | Implementation | Protects Against |
|------|---------------|-----------------|
| Content Safety | Content Safety NIM via NeMo Guardrails input rail | Harmful, violent, sexual, or illegal content in user queries |
| Jailbreak Detection | Jailbreak Detection NIM via NeMo Guardrails input rail | Prompt injection, role-play attacks, system prompt extraction, encoding-based bypasses |
| Topical Control | Colang 2.0 topical rail + Topic Control NIM | Off-topic queries that waste compute and could produce hallucinated answers |
| Input length validation | NeMo Guardrails dialog rail | Excessively long inputs designed to overwhelm context window |

**Configuration example (Colang 2.0)**:
```colang
define user ask about documentation
  "What does the API endpoint for model serving do?"
  "How do I configure the retriever pipeline?"
  "Explain the deployment architecture."

define user ask off topic
  "What's the weather today?"
  "Write me a poem."
  "Tell me a joke."

define flow handle off topic
  user ask off topic
  bot refuse off topic
  bot say "I can only answer questions about the technical documentation in my knowledge base."
```

### 6.2 Output Rails

| Rail | Implementation | Protects Against |
|------|---------------|-----------------|
| Hallucination Check | NeMo Guardrails fact-checking rail | Claims not supported by retrieved passages; fabricated citations |
| PII Redaction | NeMo Guardrails redaction processor (GLiNER/Presidio) | Leaking personally identifiable information from source documents |
| Response format validation | Custom output rail | Malformed responses that lack citations or exceed length limits |

### 6.3 Execution Rails

| Rail | Implementation | Protects Against |
|------|---------------|-----------------|
| Tool call validation | NeMo Guardrails execution rail | Agent attempting to call unauthorized tools or passing dangerous parameters |
| Retrieval scope enforcement | Custom execution rail | Retrieval queries that attempt to access partitions outside the authorized document set |

### 6.4 Guardrail Configuration (RailsConfig)

```yaml
models:
  - type: main
    engine: nim
    model: meta/llama-3.1-70b-instruct

rails:
  input:
    flows:
      - content safety check
      - jailbreak detection check
      - topic check
  output:
    flows:
      - fact checking
      - pii redaction
      - response format check
  execution:
    flows:
      - tool call validation
```

---

## 7. Deployment Plan

### Docker Compose Architecture

```yaml
services:
  # Agent API Server
  agent-server:
    image: agent-rag:latest
    build: .
    ports:
      - "8000:8000"
    environment:
      - NIM_LLM_ENDPOINT=http://nim-llm:8000
      - NIM_EMBED_ENDPOINT=http://nim-embed:8000
      - MILVUS_HOST=milvus
      - REDIS_HOST=redis
    depends_on:
      - nim-llm
      - nim-embed
      - milvus
      - redis

  # NIM LLM Endpoint
  nim-llm:
    image: nvcr.io/nim/meta/llama-3.1-70b-instruct:latest
    ports:
      - "8001:8000"
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 2
              capabilities: [gpu]

  # NIM Embedding Endpoint
  nim-embed:
    image: nvcr.io/nim/nvidia/llama-3.2-nv-embedqa-1b-v2:latest
    ports:
      - "8002:8000"
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]

  # Vector Store
  milvus:
    image: milvusdb/milvus:latest
    ports:
      - "19530:19530"
    volumes:
      - milvus-data:/var/lib/milvus

  # State/Cache Store
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  # Observability
  phoenix:
    image: arizephoenix/phoenix:latest
    ports:
      - "6006:6006"

volumes:
  milvus-data:
```

### Deployment Steps

1. Build the agent image with NAT configuration, guardrails config, and application code.
2. Pull NIM container images from NVIDIA NGC.
3. Start infrastructure services (Milvus, Redis, Phoenix).
4. Start NIM endpoints and wait for health checks.
5. Start the agent server and verify end-to-end health.
6. Run the ingestion pipeline to populate Milvus.
7. Run the evaluation suite to verify production-readiness thresholds.

---

## 8. Observability and Monitoring

### NAT Observability Configuration

NAT exports OpenTelemetry traces to observability backends. Configure export to Phoenix or Langfuse.

**Key traces captured**:
- Workflow-level: total request duration, token count, model used
- Step-level: retrieval latency, reranking latency, generation latency, guardrail check duration
- Tool-level: individual tool call duration, parameters, results

### Dashboards

| Dashboard | Metrics | Purpose |
|-----------|---------|---------|
| Request Overview | Request rate, p50/p95/p99 latency, error rate | Operational health at a glance |
| Retrieval Quality | Average retrieval latency, cache hit rate, empty retrieval rate | Detect retrieval degradation before it affects answers |
| Generation Quality | Average generation latency, token usage per request, guardrail trigger rate | Monitor cost and quality |
| Safety | Jailbreak attempt rate, off-topic rejection rate, PII redaction trigger rate | Security posture |
| Evaluation Trends | Faithfulness/relevance scores over time, regression alerts | Quality trends across deployments |

### Alerting Rules

- **p95 latency > 8 seconds for 5 minutes**: Page on-call. Likely NIM endpoint saturation.
- **Error rate > 5% for 2 minutes**: Page on-call. Check NIM health, Milvus connectivity.
- **Empty retrieval rate > 15%**: Alert. Likely index corruption or embedding model drift.
- **Guardrail trigger rate spikes > 3x baseline**: Alert. Possible adversarial attack or upstream data issue.

---

## 9. Scaling Concerns

### At 100 Concurrent Users

**What breaks first**: NIM LLM endpoint. A 70B model on 2xH100 handles roughly 20-40 concurrent inference requests depending on sequence length. Always verify hardware requirements on the specific Blueprint page or repository before attempting full deployment. Requirements vary significantly across Blueprints. At 100 users, the request queue grows and p95 latency degrades.

**Solution**: Scale NIM LLM horizontally — add more NIM replicas behind a load balancer. Each replica needs its own GPU allocation.

**Second bottleneck**: Milvus search latency under concurrent load. cuVS helps, but 100 concurrent vector searches may saturate a single Milvus node.

**Solution**: Milvus supports horizontal scaling via sharding. Partition the collection by document category for more efficient searches.

### At 1,000 Concurrent Users

**Additional breaks**:
- **Redis**: Session state and caching may need Redis Cluster.
- **NAT FastAPI Server**: Single server process cannot handle 1,000 concurrent WebSocket connections. Deploy multiple replicas with a reverse proxy.
- **Embedding NIM**: Embedding queries for 1,000 users simultaneously. Batch processing helps but may need multiple embedding NIM replicas.
- **Guardrail NIMs**: Content Safety and Jailbreak Detection NIMs become bottlenecks since every request passes through them.

**Scaling priority order**:
1. NIM LLM replicas (highest latency contributor)
2. NAT FastAPI Server replicas (connection handling)
3. Guardrail NIM replicas (throughput gate)
4. Milvus sharding (search throughput)
5. Embedding NIM replicas (embedding throughput)
6. Redis Cluster (state management)

### What to Scale First

Always scale the component with the highest per-request latency contribution first. Use NAT Profiler to identify the bottleneck before scaling. Do not guess.

---

## 10. What Makes This Weak vs. Strong

### Weak Submission Characteristics

- **No evaluation**: Ships without any automated quality measurement. No faithfulness scores, no retrieval metrics, no adversarial testing. "It seemed to work when I tried it" is not evaluation.
- **No guardrails**: No input filtering, no output checking, no execution rails. The first jailbreak attempt succeeds. PII in source docs leaks to users.
- **Single chunking strategy**: Fixed 512-token chunks for all document types. API references, narrative docs, and code samples all get the same treatment. Results in poor retrieval for structured content.
- **No monitoring**: No traces, no dashboards, no alerts. When the system degrades, no one knows until users complain.
- **Hardcoded prompts**: System prompt is a string literal in the code. No iteration, no testing of prompt variants, no parameter optimization.
- **No citation verification**: System claims to cite sources but citations are not validated against actual retrieved passages.
- **Deployment is "run the script"**: No Docker Compose, no health checks, no restart policies. System dies and stays dead.

### Strong Submission Characteristics

- **Comprehensive evaluation suite**: Retriever eval, RAG eval, adversarial eval, latency eval. All metrics at target thresholds. Runs in CI/CD.
- **Multi-layered guardrails**: Input rails (content safety + jailbreak + topical), output rails (hallucination check + PII redaction), execution rails (tool validation). Each layer configured with specific Colang 2.0 flows.
- **Adaptive chunking**: Document-structure-aware chunking for structured docs, recursive splitting for unstructured. Chunk size tuned via retrieval eval metrics.
- **Full observability**: OpenTelemetry traces exported to Phoenix/Langfuse. Dashboards for operational health, retrieval quality, safety posture. Alerting rules for degradation.
- **Parameter-optimized prompts**: System prompt developed through systematic testing. NAT Agent Hyperparameter Optimizer used to tune prompt parameters. Multiple prompt variants evaluated against the test set.
- **Documented failure modes**: README describes known failure modes (e.g., "system returns low-confidence answers for questions spanning multiple documents"), mitigation strategies, and escalation procedures.
- **Production-grade deployment**: Docker Compose with health checks, restart policies, resource limits, volume persistence. Deployment runbook included.
- **Evaluation-driven development**: Every pipeline change is validated against the evaluation suite before deployment. Regression on any metric blocks deployment.

---

## 11. Deliverables Checklist

- [ ] Source code for the complete agent system (NAT YAML config, Guardrails config, application code)
- [ ] Docker Compose file that brings up all services
- [ ] Ingestion pipeline script that processes a sample corpus
- [ ] Evaluation suite with test dataset and scoring scripts
- [ ] Evaluation report showing all metrics at target thresholds
- [ ] Observability configuration (Phoenix/Langfuse export, dashboard definitions)
- [ ] Architecture diagram (can be the ASCII diagram above, refined)
- [ ] README documenting: setup instructions, architecture decisions, evaluation results, known limitations, failure modes
- [ ] 5-minute demo video or live demonstration showing: a correct answer with citations, a jailbreak attempt being blocked, an off-topic query being refused, the monitoring dashboard
