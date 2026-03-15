# Lab 4: RAG Pipeline with NVIDIA Retrieval Stack

## Interaction Classification
- **Type**: Hosted + Local NVIDIA interaction
- **NVIDIA Services Used**: NIM embedding endpoint, NIM reranking endpoint, NIM LLM endpoint, NAT retriever, Milvus (or FAISS local)
- **Credentials Required**: NVIDIA_API_KEY from build.nvidia.com
- **Hardware**: CPU laptop sufficient for FAISS; Docker for Milvus
- **Milestone**: Milestone 4 (RAG on NVIDIA stack)

## Objective

Build a complete **Retrieval-Augmented Generation (RAG)** pipeline using NVIDIA tools end-to-end: document ingestion, chunking, embedding via NIM, vector storage in Milvus, retrieval, reranking, and generation. You will experiment with multiple chunking strategies, evaluate retrieval quality quantitatively, and assess generation faithfulness. This lab produces a reusable RAG tool that Lab 5 will integrate into a tool-using agent.

## Prerequisites

| Requirement | Details |
|---|---|
| Modules | Module 4 — Retrieval-Augmented Generation |
| Prior labs | Lab 1 (NIM API access) |
| Software | Python 3.10+, `nvidia-nat`, Docker (for Milvus) |
| Accounts | NVIDIA NGC account with NIM API key (embedding + reranking + LLM endpoints) |
| Hardware | CPU-only for development with FAISS; Milvus requires Docker with 4GB+ RAM |
| Data | 50-100 documents (see Step 1 for sources) |

### NIM Endpoints Used

| Endpoint | Model | Purpose |
|---|---|---|
| Embedding | `nvidia/nv-embedqa-e5-v5` | Document and query embedding |
| Reranking | `nvidia/nv-rerankqa-mistral-4b-v3` | Reranking retrieved passages |
| Generation | `meta/llama-3.1-70b-instruct` | Answer generation |

All three are available at build.nvidia.com with your existing API key.

## Deliverables

1. `ingest.py` — document ingestion and chunking pipeline
2. `embed.py` — embedding generation using NIM endpoint
3. `store.py` — vector storage (FAISS for local dev, Milvus for production)
4. `retrieve.py` — retrieval + reranking pipeline
5. `generate.py` — RAG generation pipeline
6. `evaluate.py` — retrieval and generation quality evaluation
7. `rag_config.yaml` — NAT configuration for the RAG pipeline
8. `chunking_comparison.md` — analysis of 2+ chunking strategies with metrics
9. `data/` — document corpus
10. `eval_results/` — evaluation metrics and test query results

## Recommended Repo Structure

```
lab_04_rag_pipeline/
├── .env
├── rag_config.yaml
├── ingest.py
├── embed.py
├── store.py
├── retrieve.py
├── generate.py
├── evaluate.py
├── chunking_comparison.md
├── data/
│   ├── raw/                   # Original documents
│   └── chunks/                # Chunked documents (JSON)
├── vectorstore/               # FAISS index files (local dev)
├── eval_results/
│   ├── retrieval_metrics.json
│   └── generation_eval.json
└── requirements.txt
```

## Implementation Steps

### Step 1: Prepare the document corpus

You need 50-100 documents on a focused topic. Options:

**Option A (recommended): ArXiv papers on agentic AI**

```python
# download_papers.py
import httpx
import os
import json

os.makedirs("data/raw", exist_ok=True)

# Use ArXiv API to fetch recent papers on agentic AI
QUERY = "all:agentic+AI+OR+all:LLM+agents"
response = httpx.get(
    "http://export.arxiv.org/api/query",
    params={"search_query": QUERY, "start": 0, "max_results": 60},
    timeout=30.0,
)

# Parse the Atom XML response
import xml.etree.ElementTree as ET
root = ET.fromstring(response.text)
ns = {"atom": "http://www.w3.org/2005/Atom"}

papers = []
for entry in root.findall("atom:entry", ns):
    paper = {
        "title": entry.find("atom:title", ns).text.strip(),
        "abstract": entry.find("atom:summary", ns).text.strip(),
        "id": entry.find("atom:id", ns).text.strip(),
        "published": entry.find("atom:published", ns).text.strip(),
    }
    papers.append(paper)
    # Save each paper as a text file
    safe_title = paper["title"][:80].replace("/", "-").replace(" ", "_")
    with open(f"data/raw/{safe_title}.txt", "w") as f:
        f.write(f"Title: {paper['title']}\n\n")
        f.write(f"Published: {paper['published']}\n\n")
        f.write(f"Abstract:\n{paper['abstract']}\n")

print(f"Downloaded {len(papers)} papers")
```

**Option B: NVIDIA documentation pages**

Download 50+ pages from NVIDIA's technical documentation (NIM, NAT, NeMo) as text files.

**Option C: Custom corpus**

Any collection of 50-100 text documents on a technical topic you are familiar with (so you can judge answer quality).

### Step 2: Implement document chunking with two strategies

Create `ingest.py`:

```python
import os
import json
import re

def chunk_fixed_size(text: str, chunk_size: int = 512, overlap: int = 64) -> list[dict]:
    """Strategy 1: Fixed-size character chunks with overlap."""
    chunks = []
    start = 0
    while start < len(text):
        end = start + chunk_size
        chunk_text = text[start:end]
        chunks.append({
            "text": chunk_text,
            "strategy": "fixed_size",
            "start_char": start,
            "end_char": min(end, len(text)),
        })
        start += chunk_size - overlap
    return chunks

def chunk_semantic(text: str, max_chunk_size: int = 1024) -> list[dict]:
    """
    Strategy 2: Semantic chunking — split on paragraph boundaries,
    then merge small paragraphs up to max_chunk_size.
    """
    paragraphs = re.split(r"\n\s*\n", text)
    chunks = []
    current_chunk = ""

    for para in paragraphs:
        para = para.strip()
        if not para:
            continue
        if len(current_chunk) + len(para) + 2 > max_chunk_size and current_chunk:
            chunks.append({
                "text": current_chunk.strip(),
                "strategy": "semantic",
            })
            current_chunk = para
        else:
            current_chunk += "\n\n" + para if current_chunk else para

    if current_chunk.strip():
        chunks.append({
            "text": current_chunk.strip(),
            "strategy": "semantic",
        })

    return chunks

def ingest_documents(raw_dir: str, output_dir: str, strategy: str = "fixed_size"):
    """Read all documents, chunk them, and save as JSON."""
    os.makedirs(output_dir, exist_ok=True)
    all_chunks = []

    for filename in sorted(os.listdir(raw_dir)):
        filepath = os.path.join(raw_dir, filename)
        if not os.path.isfile(filepath):
            continue
        with open(filepath, "r", encoding="utf-8", errors="ignore") as f:
            text = f.read()

        if strategy == "fixed_size":
            chunks = chunk_fixed_size(text)
        elif strategy == "semantic":
            chunks = chunk_semantic(text)
        else:
            raise ValueError(f"Unknown strategy: {strategy}")

        for i, chunk in enumerate(chunks):
            chunk["source_file"] = filename
            chunk["chunk_id"] = f"{filename}::chunk_{i}"
            all_chunks.append(chunk)

    output_path = os.path.join(output_dir, f"chunks_{strategy}.json")
    with open(output_path, "w") as f:
        json.dump(all_chunks, f, indent=2)

    print(f"Strategy '{strategy}': {len(all_chunks)} chunks from {len(os.listdir(raw_dir))} files")
    print(f"  Avg chunk length: {sum(len(c['text']) for c in all_chunks) / len(all_chunks):.0f} chars")
    print(f"  Saved to: {output_path}")
    return all_chunks

if __name__ == "__main__":
    for strategy in ["fixed_size", "semantic"]:
        ingest_documents("data/raw", "data/chunks", strategy=strategy)
```

Run it:

```bash
python ingest.py
```

### Step 3: Generate embeddings using NIM

Create `embed.py`:

```python
import httpx
import json
import os
import time
from dotenv import load_dotenv

load_dotenv()

API_KEY = os.environ["NVIDIA_API_KEY"]
EMBED_URL = "https://integrate.api.nvidia.com/v1/embeddings"
EMBED_MODEL = "nvidia/nv-embedqa-e5-v5"
BATCH_SIZE = 50  # NIM embedding endpoint limit per request

def embed_texts(texts: list[str]) -> list[list[float]]:
    """Embed a list of texts using NIM embedding endpoint."""
    all_embeddings = []

    for i in range(0, len(texts), BATCH_SIZE):
        batch = texts[i:i + BATCH_SIZE]
        response = httpx.post(
            EMBED_URL,
            headers={
                "Authorization": f"Bearer {API_KEY}",
                "Content-Type": "application/json",
            },
            json={
                "model": EMBED_MODEL,
                "input": batch,
                "input_type": "passage",  # Use "query" for query embedding
                "encoding_format": "float",
            },
            timeout=60.0,
        )
        response.raise_for_status()
        data = response.json()
        batch_embeddings = [item["embedding"] for item in data["data"]]
        all_embeddings.extend(batch_embeddings)

        print(f"  Embedded batch {i//BATCH_SIZE + 1} ({len(batch)} texts)")
        time.sleep(0.5)  # Rate limit courtesy

    return all_embeddings

def embed_query(query: str) -> list[float]:
    """Embed a single query — uses input_type='query' for asymmetric retrieval."""
    response = httpx.post(
        EMBED_URL,
        headers={
            "Authorization": f"Bearer {API_KEY}",
            "Content-Type": "application/json",
        },
        json={
            "model": EMBED_MODEL,
            "input": [query],
            "input_type": "query",
            "encoding_format": "float",
        },
        timeout=30.0,
    )
    response.raise_for_status()
    return response.json()["data"][0]["embedding"]

def embed_chunks(chunks_path: str, output_path: str):
    """Load chunks from JSON, embed all texts, save with embeddings."""
    with open(chunks_path) as f:
        chunks = json.load(f)

    texts = [c["text"] for c in chunks]
    print(f"Embedding {len(texts)} chunks...")
    embeddings = embed_texts(texts)

    for chunk, emb in zip(chunks, embeddings):
        chunk["embedding"] = emb

    with open(output_path, "w") as f:
        json.dump(chunks, f)

    print(f"Saved embeddings to {output_path}")
    print(f"Embedding dimension: {len(embeddings[0])}")

if __name__ == "__main__":
    os.makedirs("data/chunks", exist_ok=True)
    for strategy in ["fixed_size", "semantic"]:
        input_path = f"data/chunks/chunks_{strategy}.json"
        output_path = f"data/chunks/chunks_{strategy}_embedded.json"
        if os.path.exists(input_path):
            embed_chunks(input_path, output_path)
```

Run it:

```bash
python embed.py
```

This will take a few minutes depending on corpus size and rate limits.

### Step 4: Store embeddings in a vector database

Create `store.py`:

```python
import json
import numpy as np
import os

# ---- FAISS backend (local dev, no Docker needed) ----
def create_faiss_index(embedded_chunks_path: str, index_dir: str = "vectorstore"):
    """Build a FAISS index from embedded chunks."""
    try:
        import faiss
    except ImportError:
        print("Install faiss-cpu: pip install faiss-cpu")
        return

    os.makedirs(index_dir, exist_ok=True)

    with open(embedded_chunks_path) as f:
        chunks = json.load(f)

    embeddings = np.array([c["embedding"] for c in chunks], dtype=np.float32)
    dimension = embeddings.shape[1]

    # Normalize for cosine similarity
    faiss.normalize_L2(embeddings)
    index = faiss.IndexFlatIP(dimension)  # Inner product after normalization = cosine
    index.add(embeddings)

    # Save index and metadata separately
    strategy = "fixed_size" if "fixed_size" in embedded_chunks_path else "semantic"
    faiss.write_index(index, os.path.join(index_dir, f"index_{strategy}.faiss"))

    # Save metadata (text, source, chunk_id) without embeddings
    metadata = [{k: v for k, v in c.items() if k != "embedding"} for c in chunks]
    with open(os.path.join(index_dir, f"metadata_{strategy}.json"), "w") as f:
        json.dump(metadata, f)

    print(f"FAISS index created: {embeddings.shape[0]} vectors, dimension {dimension}")

# ---- Milvus backend (production, requires Docker) ----
def create_milvus_collection(embedded_chunks_path: str, collection_name: str = "rag_docs"):
    """Create a Milvus collection and insert embedded chunks."""
    try:
        from pymilvus import connections, Collection, FieldSchema, CollectionSchema, DataType, utility
    except ImportError:
        print("Install pymilvus: pip install pymilvus")
        return

    connections.connect("default", host="localhost", port="19530")

    with open(embedded_chunks_path) as f:
        chunks = json.load(f)

    dimension = len(chunks[0]["embedding"])

    # Drop existing collection if present
    if utility.has_collection(collection_name):
        utility.drop_collection(collection_name)

    fields = [
        FieldSchema(name="id", dtype=DataType.INT64, is_primary=True, auto_id=True),
        FieldSchema(name="chunk_id", dtype=DataType.VARCHAR, max_length=256),
        FieldSchema(name="text", dtype=DataType.VARCHAR, max_length=4096),
        FieldSchema(name="source_file", dtype=DataType.VARCHAR, max_length=256),
        FieldSchema(name="embedding", dtype=DataType.FLOAT_VECTOR, dim=dimension),
    ]
    schema = CollectionSchema(fields, description="RAG document chunks")
    collection = Collection(collection_name, schema)

    # Insert in batches
    batch_size = 100
    for i in range(0, len(chunks), batch_size):
        batch = chunks[i:i + batch_size]
        collection.insert([
            [c["chunk_id"] for c in batch],
            [c["text"][:4096] for c in batch],  # Truncate to fit VARCHAR limit
            [c.get("source_file", "unknown") for c in batch],
            [c["embedding"] for c in batch],
        ])

    # Create index for fast search
    collection.create_index(
        field_name="embedding",
        index_params={
            "metric_type": "COSINE",
            "index_type": "IVF_FLAT",
            "params": {"nlist": 128},
        },
    )
    collection.load()

    print(f"Milvus collection '{collection_name}': {collection.num_entities} entities, dimension {dimension}")

if __name__ == "__main__":
    for strategy in ["fixed_size", "semantic"]:
        path = f"data/chunks/chunks_{strategy}_embedded.json"
        if os.path.exists(path):
            print(f"\n--- Creating FAISS index for '{strategy}' strategy ---")
            create_faiss_index(path)
            # Uncomment the next line if you have Milvus running:
            # create_milvus_collection(path, collection_name=f"rag_{strategy}")
```

```bash
pip install faiss-cpu
python store.py
```

For Milvus (optional but recommended):

```bash
# Start Milvus with Docker Compose
# Download from: https://milvus.io/docs/install_standalone-docker.md
docker compose up -d
pip install pymilvus
# Then uncomment the Milvus line in store.py and re-run
```

### Step 5: Implement retrieval with reranking

Create `retrieve.py`:

```python
import json
import numpy as np
import httpx
import os
from dotenv import load_dotenv
from embed import embed_query

load_dotenv()

API_KEY = os.environ["NVIDIA_API_KEY"]
RERANK_URL = "https://integrate.api.nvidia.com/v1/ranking"
RERANK_MODEL = "nvidia/nv-rerankqa-mistral-4b-v3"

def retrieve_faiss(query: str, strategy: str = "fixed_size", top_k: int = 20) -> list[dict]:
    """Retrieve top-k chunks from FAISS index."""
    import faiss

    query_embedding = np.array([embed_query(query)], dtype=np.float32)
    faiss.normalize_L2(query_embedding)

    index = faiss.read_index(f"vectorstore/index_{strategy}.faiss")
    with open(f"vectorstore/metadata_{strategy}.json") as f:
        metadata = json.load(f)

    scores, indices = index.search(query_embedding, top_k)

    results = []
    for score, idx in zip(scores[0], indices[0]):
        if idx == -1:
            continue
        result = metadata[idx].copy()
        result["retrieval_score"] = float(score)
        results.append(result)

    return results

def rerank(query: str, passages: list[dict], top_n: int = 5) -> list[dict]:
    """Rerank passages using NIM reranking endpoint."""
    response = httpx.post(
        RERANK_URL,
        headers={
            "Authorization": f"Bearer {API_KEY}",
            "Content-Type": "application/json",
        },
        json={
            "model": RERANK_MODEL,
            "query": {"text": query},
            "passages": [{"text": p["text"]} for p in passages],
        },
        timeout=30.0,
    )
    response.raise_for_status()
    rankings = response.json()["rankings"]

    # Sort by reranking score and take top_n
    ranked = sorted(rankings, key=lambda x: x["logit"], reverse=True)[:top_n]

    reranked_results = []
    for r in ranked:
        idx = r["index"]
        result = passages[idx].copy()
        result["rerank_score"] = r["logit"]
        reranked_results.append(result)

    return reranked_results

def retrieve_and_rerank(query: str, strategy: str = "fixed_size", initial_k: int = 20, final_k: int = 5) -> list[dict]:
    """Full retrieval pipeline: embed query -> FAISS search -> rerank -> return top results."""
    print(f"  Retrieving top-{initial_k} from FAISS ({strategy})...")
    candidates = retrieve_faiss(query, strategy=strategy, top_k=initial_k)

    print(f"  Reranking to top-{final_k}...")
    reranked = rerank(query, candidates, top_n=final_k)

    for i, r in enumerate(reranked):
        print(f"    [{i+1}] score={r['rerank_score']:.3f} | {r['source_file']} | {r['text'][:80]}...")

    return reranked

if __name__ == "__main__":
    test_queries = [
        "How do LLM agents use tools?",
        "What are the challenges of planning in agentic AI?",
        "Explain retrieval augmented generation.",
    ]
    for q in test_queries:
        print(f"\nQuery: {q}")
        results = retrieve_and_rerank(q, strategy="semantic")
        print()
```

### Step 6: Build the generation pipeline

Create `generate.py`:

```python
import httpx
import os
from dotenv import load_dotenv
from retrieve import retrieve_and_rerank

load_dotenv()

API_KEY = os.environ["NVIDIA_API_KEY"]
BASE_URL = "https://integrate.api.nvidia.com/v1"
MODEL = "meta/llama-3.1-70b-instruct"

def generate_answer(query: str, context_chunks: list[dict], max_tokens: int = 1024) -> dict:
    """Generate an answer using retrieved context."""
    # Format context for the prompt
    context_text = ""
    for i, chunk in enumerate(context_chunks):
        source = chunk.get("source_file", "unknown")
        context_text += f"\n[Source {i+1}: {source}]\n{chunk['text']}\n"

    system_prompt = """You are a knowledgeable research assistant. Answer the user's question
using ONLY the provided context. If the context does not contain enough information to answer,
say so explicitly. Always cite which source(s) you used by referencing [Source N]."""

    user_prompt = f"""Context:
{context_text}

Question: {query}

Answer based on the context above. Cite your sources."""

    response = httpx.post(
        f"{BASE_URL}/chat/completions",
        headers={
            "Authorization": f"Bearer {API_KEY}",
            "Content-Type": "application/json",
        },
        json={
            "model": MODEL,
            "messages": [
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": user_prompt},
            ],
            "max_tokens": max_tokens,
            "temperature": 0,
        },
        timeout=60.0,
    )
    response.raise_for_status()
    result = response.json()

    return {
        "answer": result["choices"][0]["message"]["content"],
        "usage": result.get("usage", {}),
        "sources": [c.get("source_file") for c in context_chunks],
    }

def rag_pipeline(query: str, strategy: str = "semantic") -> dict:
    """Complete RAG pipeline: retrieve -> rerank -> generate."""
    print(f"RAG Pipeline for: {query}")
    chunks = retrieve_and_rerank(query, strategy=strategy)
    result = generate_answer(query, chunks)
    print(f"\nAnswer:\n{result['answer']}\n")
    print(f"Sources: {result['sources']}")
    print(f"Tokens: {result['usage']}")
    return result

if __name__ == "__main__":
    queries = [
        "How do LLM agents use tools?",
        "What are the main challenges in building agentic AI systems?",
        "Explain the ReAct pattern for LLM agents.",
    ]
    for q in queries:
        print(f"\n{'='*60}")
        rag_pipeline(q, strategy="semantic")
```

### Step 7: Configure RAG as a NAT retriever (optional)

Create `rag_config.yaml` to configure the pipeline via NAT:

```yaml
agent:
  type: tool_calling
  llm:
    type: nim
    model: meta/llama-3.1-70b-instruct
    api_key: ${NVIDIA_API_KEY}
    base_url: https://integrate.api.nvidia.com/v1
    max_tokens: 1024
  system_prompt: >
    You are a research assistant with access to a document search tool.
    Use it to find information before answering questions.
    Always cite your sources.
  retriever:
    type: nim
    embedder:
      model: nvidia/nv-embedqa-e5-v5
      api_key: ${NVIDIA_API_KEY}
      base_url: https://integrate.api.nvidia.com/v1
    reranker:
      model: nvidia/nv-rerankqa-mistral-4b-v3
      api_key: ${NVIDIA_API_KEY}
      base_url: https://integrate.api.nvidia.com/v1
    vector_store:
      type: faiss
      index_path: vectorstore/index_semantic.faiss
      metadata_path: vectorstore/metadata_semantic.json
    top_k: 20
    rerank_top_n: 5
  functions:
    - name: document_search
      description: "Search the document corpus for relevant information."
      parameters:
        type: object
        properties:
          query:
            type: string
            description: "The search query"
        required: [query]
```

### Step 8: Evaluate retrieval quality

Create `evaluate.py`:

```python
import json
import os
from retrieve import retrieve_faiss, retrieve_and_rerank

# Define test queries with known relevant documents
# You need to manually identify which documents are relevant to each query
EVAL_SET = [
    {
        "query": "How do LLM agents use tools?",
        "relevant_sources": [
            # Fill in filenames of documents that actually answer this question
            # e.g., "Toolformer_A_Language_Model_That_Can_Use_Tools.txt"
        ],
    },
    {
        "query": "What is retrieval augmented generation?",
        "relevant_sources": [],
    },
    {
        "query": "Explain planning in agentic AI systems",
        "relevant_sources": [],
    },
    {
        "query": "What are the safety risks of AI agents?",
        "relevant_sources": [],
    },
    {
        "query": "How do multi-agent systems coordinate?",
        "relevant_sources": [],
    },
]

def recall_at_k(retrieved: list[dict], relevant_sources: list[str], k: int) -> float:
    """What fraction of relevant documents appear in the top-k results?"""
    if not relevant_sources:
        return -1.0  # Cannot evaluate without ground truth
    retrieved_sources = {r["source_file"] for r in retrieved[:k]}
    hits = sum(1 for s in relevant_sources if s in retrieved_sources)
    return hits / len(relevant_sources)

def evaluate_retrieval(strategy: str = "semantic"):
    """Evaluate retrieval quality across the eval set."""
    results = []
    for item in EVAL_SET:
        query = item["query"]
        relevant = item["relevant_sources"]

        # Retrieval only (no reranking)
        retrieved = retrieve_faiss(query, strategy=strategy, top_k=20)
        recall_5_no_rerank = recall_at_k(retrieved, relevant, k=5)
        recall_10_no_rerank = recall_at_k(retrieved, relevant, k=10)

        # With reranking
        reranked = retrieve_and_rerank(query, strategy=strategy, initial_k=20, final_k=10)
        recall_5_reranked = recall_at_k(reranked, relevant, k=5)

        results.append({
            "query": query,
            "recall@5_no_rerank": recall_5_no_rerank,
            "recall@10_no_rerank": recall_10_no_rerank,
            "recall@5_reranked": recall_5_reranked,
            "num_relevant": len(relevant),
        })

    os.makedirs("eval_results", exist_ok=True)
    with open(f"eval_results/retrieval_metrics_{strategy}.json", "w") as f:
        json.dump(results, f, indent=2)

    print(f"\n{'Query':<50} {'R@5':>6} {'R@10':>6} {'R@5+RR':>8}")
    print("-" * 75)
    for r in results:
        q = r["query"][:48]
        r5 = f"{r['recall@5_no_rerank']:.2f}" if r["recall@5_no_rerank"] >= 0 else "N/A"
        r10 = f"{r['recall@10_no_rerank']:.2f}" if r["recall@10_no_rerank"] >= 0 else "N/A"
        r5rr = f"{r['recall@5_reranked']:.2f}" if r["recall@5_reranked"] >= 0 else "N/A"
        print(f"{q:<50} {r5:>6} {r10:>6} {r5rr:>8}")

if __name__ == "__main__":
    print("IMPORTANT: Fill in 'relevant_sources' in EVAL_SET before running.\n")
    for strategy in ["fixed_size", "semantic"]:
        print(f"\n{'='*60}")
        print(f"Evaluating strategy: {strategy}")
        print(f"{'='*60}")
        evaluate_retrieval(strategy)
```

**Before running `evaluate.py`**, you must manually review your corpus and fill in the `relevant_sources` lists. This is intentionally manual — you need to understand what "ground truth" means for retrieval evaluation.

### Step 9: Evaluate generation quality

Add to `evaluate.py` or create a separate evaluation:

For each test query:
1. Run the full RAG pipeline
2. Manually assess the answer on three dimensions:
   - **Faithfulness**: Does the answer only use information from the retrieved context? (no hallucination)
   - **Relevance**: Does the answer address the question?
   - **Completeness**: Does the answer cover all relevant aspects available in the context?

Score each dimension 1-5 and record in `eval_results/generation_eval.json`.

### Step 10: Document chunking strategy comparison

In `chunking_comparison.md`, compare your two chunking strategies:

- Number of chunks produced
- Average chunk size
- Retrieval recall@5 and recall@10 for each strategy
- Generation quality scores for each strategy
- Qualitative observations: which strategy produces more coherent chunks? Which leads to better answers?

## Evaluation Rubric

| Criterion | Excellent (5) | Adequate (3) | Incomplete (1) |
|---|---|---|---|
| **Pipeline completeness** | All stages work end-to-end: ingest -> chunk -> embed -> store -> retrieve -> rerank -> generate | Pipeline works but skips reranking or uses only one backend | Missing stages or pipeline does not run |
| **Chunking comparison** | Two strategies implemented with quantitative metrics and qualitative analysis | Two strategies implemented but no comparison | Only one strategy |
| **Retrieval evaluation** | Recall@k computed with manually verified ground truth for 5+ queries | Metrics computed but ground truth is questionable | No retrieval evaluation |
| **Generation quality** | Answers assessed for faithfulness, relevance, completeness with documented scores | Answers spot-checked | No generation evaluation |
| **NIM integration** | Uses NIM embedding, reranking, and generation endpoints correctly | Uses NIM for some but not all stages | Does not use NIM endpoints |

## Likely Bugs and Failure Cases

1. **Embedding dimension mismatch**: If you switch embedding models between indexing and querying, the dimensions will not match and FAISS will throw a runtime error. Always use the same model for both. The `nv-embedqa-e5-v5` model produces 1024-dimensional embeddings.

2. **NIM rate limiting on embedding**: Embedding 500+ chunks with the free tier will hit rate limits. The `time.sleep(0.5)` in `embed.py` helps, but you may need to increase it to 1-2 seconds. If you get 429 errors, reduce `BATCH_SIZE` and increase sleep.

3. **VARCHAR overflow in Milvus**: Milvus has a max VARCHAR length. If your chunks exceed 4096 characters, the insert will fail. The `store.py` code truncates with `[:4096]`, but this may cut off important content. Consider increasing the field limit or splitting large chunks further.

4. **`input_type` mismatch in embedding**: The NIM embedding endpoint distinguishes between `"passage"` (for documents) and `"query"` (for search queries). Using `"passage"` for queries will degrade retrieval quality significantly. Make sure `embed.py` uses `"passage"` for indexing and `embed_query()` uses `"query"` for search.

5. **FAISS index not normalized**: If you skip `faiss.normalize_L2()`, the inner product scores will not be cosine similarity and results will be wrong. Always normalize before adding to an `IndexFlatIP`.

6. **Reranking endpoint returns different JSON structure**: The NIM reranking response format may differ between model versions. Check the actual response structure with a print statement before parsing. The `rankings` key and `logit` field names may vary.

7. **Empty or near-empty chunks**: Fixed-size chunking at boundaries can produce chunks with only whitespace or a few characters. Filter out chunks shorter than 50 characters before embedding to avoid wasting vector space.

## Extension Ideas

1. **Hybrid retrieval**: Combine dense (embedding) and sparse (BM25/keyword) retrieval. Use a library like `rank-bm25` for the sparse component and merge the two result sets before reranking. Compare hybrid vs. dense-only recall.

2. **Chunk metadata enrichment**: Before embedding, prepend each chunk with its source document title and section heading. This gives the embedding model more context and often improves retrieval quality. Measure the difference.

3. **Streaming generation**: Modify `generate.py` to use NIM's streaming endpoint (`stream: true`). Display the answer token-by-token in the terminal. This is important for user experience in production RAG systems.
