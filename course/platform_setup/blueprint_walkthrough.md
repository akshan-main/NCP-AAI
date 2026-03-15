# NVIDIA Blueprint Walkthrough — AI-Q Research Assistant

## Hardware / Environment Requirements

| Requirement | Details |
|---|---|
| Full Blueprint deployment | 2x H100 80GB or 3x A100 80GB (Docker Compose), Kubernetes for production. Always verify hardware requirements on the specific Blueprint page or repository before attempting full deployment. Requirements vary significantly across Blueprints. |
| Study and partial reproduction | CPU laptop + NVIDIA_API_KEY (no GPU required) |
| Internet required? | Yes |
| Python version | 3.9+ |
| Platform | macOS, Linux, Windows with WSL |

**The majority of this walkthrough is designed to work on a CPU laptop with only an NVIDIA API key.** Full deployment is optional and requires significant GPU hardware. The learning value is in understanding the architecture, not in running every container.

---

## Prerequisites

- Completed: [NVIDIA Account and API Key Setup](build_nvidia_account_and_api_key.md)
- Completed: [First NIM API Call](first_nim_api_call.md)
- `NVIDIA_API_KEY` environment variable is set and verified
- Git installed
- Python 3.9+ with pip
- Docker installed (optional, for studying the compose file; not required for the partial reproduction)
- Install dependencies for partial reproduction:

```bash
pip install openai faiss-cpu numpy langchain-nvidia-ai-endpoints langchain langchain-community
```

---

## Setup Steps

### Step 1: Navigate to the Blueprint Catalog

Open your browser and go to:

```
https://build.nvidia.com/blueprints
```

Find the **AI-Q Research Assistant** blueprint. Click on it to view its detail page.

The Blueprints catalog provides reference architectures — complete, end-to-end AI applications designed by NVIDIA. They are not tutorials; they are production-grade systems you can study, adapt, and deploy.

### Step 2: Read the Architecture Overview

On the blueprint detail page, study the architecture diagram. Identify all components:

| Component | Role | NVIDIA-specific? |
|---|---|---|
| LLM (e.g., Llama 3.3 70B) | Generates responses, reasons over retrieved context | Served via NIM |
| Embedding model (e.g., NV-EmbedQA) | Converts text to vectors for semantic search | Served via NIM |
| Reranker (e.g., NV-RerankQA) | Re-scores retrieved documents for relevance | Served via NIM |
| Vector store (Milvus) | Stores and retrieves document embeddings | Open source (not NVIDIA-specific) |
| OCR / Document parser | Extracts text from PDFs and images | NeMo Retriever or open source alternatives |
| Agent toolkit | Orchestrates the multi-step RAG pipeline | NeMo Agent Toolkit |

Key architectural insights:
- The LLM, embedding model, and reranker are all served as NIM microservices.
- Milvus is an open-source vector database — it could be swapped for FAISS, Pinecone, Weaviate, etc.
- The agent orchestrates: query -> embed -> retrieve -> rerank -> generate.

### Step 3: Clone the Blueprint Repo

```bash
git clone https://github.com/NVIDIA-AI-Blueprints/aiq-research-assistant.git
cd aiq-research-assistant
```

Explore the directory structure:

```bash
ls -la
```

Key files to examine:
- `README.md` — Setup instructions and hardware requirements
- `docker-compose.yml` (or equivalent) — Service definitions
- Configuration files for each service
- Sample data and test scripts

### Step 4: Read the README and Identify Hardware Requirements

Read the README carefully. Look for:

1. **Hardware requirements**: The AI-Q Blueprint specifies GPU requirements for running the full stack locally. Typical requirements are 2x H100 80GB or 3x A100 80GB, because the Blueprint runs multiple NIM containers simultaneously (LLM + embedding + reranker). Always verify hardware requirements on the specific Blueprint page or repository before attempting full deployment. Requirements vary significantly across Blueprints.

2. **Software requirements**: Docker, NVIDIA Container Toolkit, NGC credentials.

3. **Quickstart instructions**: These assume you have the hardware. If you do not, that is fine — the next steps are designed for you.

Document what you find:

```
Hardware requirements found in README:
- GPU: ___________
- RAM: ___________
- Storage: ___________
- Docker version: ___________
- NVIDIA driver version: ___________
```

### Step 5: Study the Docker Compose File

Even if you cannot run the containers, the Docker Compose file is the most informative artifact in any Blueprint.

Open the Docker Compose file and map each service:

```bash
# Read the compose file (look for docker-compose.yml, docker-compose.yaml, or compose.yaml)
```

For each service, document:

| Service Name | Container Image | Ports | GPU Allocation | Purpose |
|---|---|---|---|---|
| llm | nvcr.io/nvidia/nim/... | 8000 | 2x H100 | LLM inference |
| embedding | nvcr.io/nvidia/nim/... | 8001 | 1x GPU | Text embedding |
| reranker | nvcr.io/nvidia/nim/... | 8002 | 1x GPU | Document reranking |
| milvus | milvusdb/milvus:... | 19530 | None (CPU) | Vector storage |
| agent | custom image | 8080 | None (CPU) | Orchestration |

Key things to note:
- Which services require GPUs (the `deploy.resources.reservations.devices` section).
- Which services are NVIDIA container images (`nvcr.io/nvidia/...`) vs. open source.
- How services communicate (shared Docker network, port mappings).
- Environment variables and configuration volumes.

### Step 6: Full Deployment (If Hardware Is Available)

**This step is optional.** Only proceed if you have the required GPU hardware.

Follow the README quickstart:

1. Log in to NGC: `docker login nvcr.io`
2. Set environment variables (NGC API key, model names)
3. Run: `docker compose up -d`
4. Wait for all services to become healthy: `docker compose ps`
5. Test the endpoint with the provided sample queries
6. Monitor GPU usage: `nvidia-smi`

### Step 7: Study Without Full Hardware (Most Learners)

If you do NOT have 2x H100 (which applies to most learners), your approach is:

**A. Identify what you CAN reproduce using hosted APIs:**

| Component | Full Blueprint (self-hosted) | Your reproduction (hosted API) |
|---|---|---|
| LLM inference | NIM container on H100 | NIM hosted API at integrate.api.nvidia.com |
| Embedding | NIM container on GPU | NIM hosted API (embedding endpoint) |
| Reranking | NIM container on GPU | NIM hosted API (reranking endpoint) |
| Vector store | Milvus container | FAISS (local, CPU, no container needed) |
| OCR/parsing | NeMo Retriever container | PyPDF or similar (local, CPU) |
| Agent orchestration | Custom container | Python script (local, CPU) |

**B. Identify what you CANNOT reproduce and why:**

- **Production-grade latency**: Self-hosted NIM on H100 achieves lower latency than hosted APIs. You cannot reproduce this without the hardware.
- **Data isolation**: Self-hosted means your data never leaves your infrastructure. Hosted APIs send data to NVIDIA's cloud.
- **Milvus at scale**: FAISS works for small datasets but Milvus supports distributed, production-scale vector storage.
- **NeMo Retriever OCR pipeline**: The full document processing pipeline (OCR, table extraction, layout analysis) requires the NeMo Retriever microservice. You can approximate with open-source tools but will lose accuracy on complex documents.

### Step 8: Reproduce a Minimal RAG Pipeline Using Hosted APIs

This is the core hands-on exercise. You will build a simplified version of the AI-Q RAG pipeline using only hosted NIM endpoints and local CPU-based components.

**Create the file `aiq_mini_rag.py`:**

```python
"""
Minimal reproduction of the AI-Q Research Assistant RAG pipeline.
Uses: NIM hosted embedding + FAISS (local) + NIM hosted LLM.
Runs on: CPU laptop with NVIDIA_API_KEY.
"""

import os
import numpy as np
import faiss
from openai import OpenAI

# --- Configuration ---
NVIDIA_API_KEY = os.environ["NVIDIA_API_KEY"]
EMBED_MODEL = "nvidia/nv-embedqa-e5-v5"
LLM_MODEL = "meta/llama-3.3-70b-instruct"
EMBED_URL = "https://integrate.api.nvidia.com/v1"
LLM_URL = "https://integrate.api.nvidia.com/v1"

# --- Sample Documents (in a real system, these come from a document store) ---
documents = [
    "NVIDIA NIM is a set of optimized inference microservices for deploying AI models "
    "with high performance. NIM supports GPU-accelerated inference for LLMs, embedding "
    "models, and other AI workloads.",

    "NVIDIA Blueprints are reference architectures for building AI applications. "
    "The AI-Q Research Assistant Blueprint demonstrates a RAG pipeline with LLM, "
    "embedding, reranking, and vector search components.",

    "Retrieval-Augmented Generation (RAG) combines a retriever that finds relevant "
    "documents with a generator (LLM) that produces answers based on the retrieved "
    "context. This reduces hallucination and grounds responses in facts.",

    "Milvus is an open-source vector database designed for similarity search on "
    "dense vectors. It supports billion-scale vector search with low latency and "
    "is commonly used in RAG pipelines.",

    "NeMo Guardrails is a toolkit for adding safety controls to LLM applications. "
    "It supports input rails (filtering user messages), output rails (filtering "
    "model responses), and topical rails (keeping conversations on track).",
]

# --- Step 1: Embed Documents ---
print("Step 1: Embedding documents with NIM embedding endpoint...")

embed_client = OpenAI(base_url=EMBED_URL, api_key=NVIDIA_API_KEY)

def get_embeddings(texts: list[str]) -> np.ndarray:
    """Get embeddings from NIM embedding endpoint."""
    response = embed_client.embeddings.create(
        model=EMBED_MODEL,
        input=texts,
    )
    return np.array([item.embedding for item in response.data], dtype=np.float32)

doc_embeddings = get_embeddings(documents)
print(f"  Embedded {len(documents)} documents, dimension: {doc_embeddings.shape[1]}")

# --- Step 2: Build FAISS Index ---
print("Step 2: Building FAISS index (local, CPU)...")

dimension = doc_embeddings.shape[1]
index = faiss.IndexFlatIP(dimension)  # Inner product (cosine similarity for normalized vectors)

# Normalize embeddings for cosine similarity
faiss.normalize_L2(doc_embeddings)
index.add(doc_embeddings)

print(f"  FAISS index built with {index.ntotal} vectors")

# --- Step 3: Query Embedding and Retrieval ---
query = "How does RAG work and what are its benefits?"
print(f"\nStep 3: Processing query: '{query}'")

query_embedding = get_embeddings([query])
faiss.normalize_L2(query_embedding)

k = 3  # Retrieve top 3 documents
scores, indices = index.search(query_embedding, k)

print(f"  Retrieved top {k} documents:")
retrieved_docs = []
for rank, (score, idx) in enumerate(zip(scores[0], indices[0])):
    print(f"    Rank {rank + 1} (score: {score:.4f}): {documents[idx][:80]}...")
    retrieved_docs.append(documents[idx])

# --- Step 4: Generate Response with LLM ---
print(f"\nStep 4: Generating response with {LLM_MODEL}...")

llm_client = OpenAI(base_url=LLM_URL, api_key=NVIDIA_API_KEY)

context = "\n\n".join(
    f"[Document {i+1}]: {doc}" for i, doc in enumerate(retrieved_docs)
)

response = llm_client.chat.completions.create(
    model=LLM_MODEL,
    messages=[
        {
            "role": "system",
            "content": (
                "You are a helpful research assistant. Answer the user's question "
                "based ONLY on the provided context documents. If the context doesn't "
                "contain enough information, say so. Cite which document(s) you used."
            ),
        },
        {
            "role": "user",
            "content": f"Context:\n{context}\n\nQuestion: {query}",
        },
    ],
    max_tokens=300,
    temperature=0.3,
)

answer = response.choices[0].message.content
print(f"\nFinal Answer:\n{answer}")
print(f"\nTokens used: {response.usage.total_tokens}")

# --- Step 5: Architecture Comparison ---
print("\n" + "=" * 60)
print("ARCHITECTURE COMPARISON")
print("=" * 60)
print(f"""
{'Component':<25} {'AI-Q Blueprint':<30} {'This Reproduction':<30}
{'-'*85}
{'LLM':<25} {'NIM on H100 (self-hosted)':<30} {'NIM hosted API':<30}
{'Embedding':<25} {'NIM on GPU (self-hosted)':<30} {'NIM hosted API':<30}
{'Reranker':<25} {'NIM on GPU (self-hosted)':<30} {'Not included (simplified)':<30}
{'Vector Store':<25} {'Milvus (containerized)':<30} {'FAISS (local, in-memory)':<30}
{'Document Parsing':<25} {'NeMo Retriever':<30} {'Hardcoded strings':<30}
{'Orchestration':<25} {'NeMo Agent Toolkit':<30} {'Python script':<30}
{'Hardware Required':<25} {'2xH100 80GB':<30} {'CPU laptop':<30}
""")
```

Run it:

```bash
python aiq_mini_rag.py
```

### Step 9: Document Your Analysis

Create a structured comparison capturing what you learned. This is the deliverable for this walkthrough.

**Template (fill this in):**

```
# AI-Q Blueprint Analysis

## Components I reused from the Blueprint
- [ ] NIM LLM endpoint (same model, same API format)
- [ ] NIM embedding endpoint (same model, same API format)
- [ ] RAG pipeline pattern (embed -> retrieve -> generate)

## Components I adapted
- [ ] Vector store: Milvus -> FAISS (same concept, different scale)
- [ ] Document parsing: NeMo Retriever -> hardcoded sample docs
- [ ] Orchestration: NeMo Agent Toolkit -> Python script

## Components I could NOT reproduce and why
- [ ] Reranker: would require rerank NIM endpoint (available as hosted API, omitted for simplicity)
- [ ] Production latency: hosted APIs add network latency vs. self-hosted on H100
- [ ] Data privacy: hosted APIs send data to NVIDIA cloud
- [ ] Scale: FAISS is in-memory, Milvus supports distributed billion-vector search

## Key architectural decisions I learned from studying the Blueprint
- The separation of embedding, retrieval, and generation into independent microservices
- The use of a reranker as a second-stage filter to improve precision
- The role of the agent as orchestrator rather than monolith
- How Docker Compose maps the logical architecture to running services

## What I would need to deploy the full Blueprint
- Hardware: ___________
- Software: ___________
- Estimated cost: ___________
```

---

## Verification Checklist

Confirm each of the following before proceeding:

- [ ] You can navigate to the AI-Q Blueprint on build.nvidia.com/blueprints
- [ ] You have cloned the Blueprint repo and examined the directory structure
- [ ] You have read the Docker Compose file and can map each service to its architectural role
- [ ] You have identified which components require GPUs and which do not
- [ ] The `aiq_mini_rag.py` script runs successfully on your CPU laptop
- [ ] The script embeds documents, retrieves relevant ones, and generates a grounded answer
- [ ] You have completed the architecture comparison document

### Expected Successful Output (aiq_mini_rag.py)

```
Step 1: Embedding documents with NIM embedding endpoint...
  Embedded 5 documents, dimension: 1024
Step 2: Building FAISS index (local, CPU)...
  FAISS index built with 5 vectors

Step 3: Processing query: 'How does RAG work and what are its benefits?'
  Retrieved top 3 documents:
    Rank 1 (score: 0.8923): Retrieval-Augmented Generation (RAG) combines a retriever that finds relevant...
    Rank 2 (score: 0.7451): NVIDIA Blueprints are reference architectures for building AI applications...
    Rank 3 (score: 0.6832): NVIDIA NIM is a set of optimized inference microservices for deploying AI...

Step 4: Generating response with meta/llama-3.3-70b-instruct...

Final Answer:
Based on the provided context, RAG (Retrieval-Augmented Generation) works by combining
two key components (Document 3): a retriever that finds relevant documents from a
knowledge base, and a generator (LLM) that produces answers based on the retrieved
context. The main benefits include reduced hallucination and responses that are
grounded in factual information from the retrieved documents...

Tokens used: 342
```

---

## Common Failure Cases

### 1. Assuming You Need Full Hardware to Learn from the Blueprint

**Symptom**: Learner skips the entire walkthrough because they do not have 2x H100.

**Fix**: The walkthrough is explicitly designed so that Steps 1-5, 7-9 work without GPUs. Step 6 (full deployment) is clearly marked as optional. The learning value is in architecture study and partial reproduction, not in running containers.

### 2. Confusing Blueprint Reference Architecture with a Tutorial

**Symptom**: Learner expects step-by-step hand-holding to build the entire system from scratch.

**Fix**: Blueprints are reference implementations, not tutorials. They show you what a production system looks like. You study them, understand the design decisions, and adapt what you need. This walkthrough provides the guided study that the Blueprint itself does not.

### 3. Not Reading the Docker Compose File

**Symptom**: Learner clones the repo but does not understand which services exist or how they connect.

**Fix**: The Docker Compose file is the single most informative file in any Blueprint. It maps the logical architecture to concrete services with ports, GPUs, and dependencies. Step 5 provides a template for systematically extracting this information.

### 4. Not Understanding Which Components Are NVIDIA-Specific vs. Generic

**Symptom**: Learner assumes everything in the Blueprint requires NVIDIA-specific technology.

**Fix**: Milvus is open source. FAISS is open source. The agent orchestration logic can run on any framework. What is NVIDIA-specific is the NIM inference layer (optimized model serving) and NeMo Retriever (document processing). The Step 7 table makes this distinction explicit.

### 5. Embedding Model Name or Endpoint Errors

**Symptom**: HTTP 404 or "model not found" when calling the embedding endpoint.

**Fix**:
- Verify the embedding model name at build.nvidia.com (search for "embed").
- The embedding endpoint uses the same base URL (`integrate.api.nvidia.com/v1`) but the endpoint is `/embeddings` not `/chat/completions`.
- The `openai` library handles this correctly when you call `client.embeddings.create()`.

### 6. FAISS Installation Issues

**Symptom**: `ModuleNotFoundError: No module named 'faiss'`.

**Fix**: Install the CPU version: `pip install faiss-cpu`. Do NOT install `faiss-gpu` unless you have a CUDA-capable GPU and the CUDA toolkit installed.

---

## Cleanup / Cost-Awareness Notes

- **API costs for partial reproduction**: The `aiq_mini_rag.py` script makes approximately:
  - 2 embedding API calls (documents + query) — very low cost
  - 1 LLM API call — a few hundred tokens
  - Total: under 2,000 tokens, negligible cost
- **Docker resources**: If you ran the full deployment (Step 6), shut down with `docker compose down`. This stops all containers and frees GPU memory.
- **Disk space**: The cloned repo may be large (especially if it includes model weights or large configs). Check with `du -sh aiq-research-assistant/`. The repo itself can be deleted after study if disk space is a concern.
- **No persistent cloud resources**: Unless you deployed to a cloud GPU instance, there are no ongoing costs. If you did use a cloud instance for Step 6, shut it down when done.
