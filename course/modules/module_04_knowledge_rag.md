# Module 4: Knowledge Integration and RAG Pipelines

**Primary Exam Domain:** Knowledge Integration and Data Handling (10%)
**NVIDIA Tools:** NeMo Retriever, NeMo Curator, NAT retrievers and embedders, AI-Q Blueprint RAG stack (Llama 3.2 NV EmbedQA, Llama 3.2 NV RerankQA, Milvus + cuVS, PaddleOCR), NIM embedding endpoints

---

## A. Module Title

**Knowledge Integration and RAG Pipelines**

---

## B. Why This Module Matters in Real Systems

Large language models are trained on static snapshots of data. The moment training ends, the model's knowledge begins to decay. In production systems — enterprise support bots, internal knowledge assistants, compliance research tools — stale or hallucinated answers are not inconveniences; they are operational failures. Retrieval-Augmented Generation (RAG) is the dominant architectural pattern for grounding LLM outputs in current, authoritative data without retraining. If you build agentic systems, you will build RAG pipelines. The question is whether you build good ones.

The quality gap between a naive RAG pipeline and a well-engineered one is enormous. A naive pipeline — fixed-size chunks, a mediocre embedding model, no reranking, a brute-force index — will retrieve irrelevant passages, confuse the LLM, and produce answers that look plausible but are wrong. A well-engineered pipeline uses document-structure-aware chunking, high-quality embeddings, hybrid retrieval, reranking, and GPU-accelerated vector search to deliver precise context to the LLM. The difference is measurable: 20-40% improvement in answer accuracy is typical when moving from naive to optimized RAG.

NVIDIA's ecosystem provides purpose-built tooling for every stage of this pipeline. NeMo Curator handles GPU-accelerated data preparation at scale. NIM endpoints serve embedding and reranking models with low latency. Milvus with cuVS provides GPU-accelerated vector similarity search. The AI-Q Blueprint assembles these into a complete reference architecture that runs on 2xH100 GPUs. Understanding these components — individually and as a system — is essential for the NCP-AAI exam and for building production-grade agentic AI systems.

---

## C. Learning Objectives

By the end of this module, the student will be able to:

1. Diagram the seven stages of a RAG pipeline (Ingest, Chunk, Embed, Store, Retrieve, Rerank, Generate) and identify the NVIDIA tool responsible for each stage.
2. Evaluate chunking strategies (fixed-size, recursive, semantic, document-structure-aware) and select the appropriate strategy for a given document corpus.
3. Explain how NeMo Curator accelerates data preparation on GPUs and describe at least three preprocessing operations it supports.
4. Configure NIM embedding endpoints and describe the architecture of Llama 3.2 NV EmbedQA (1B parameter embedding model).
5. Compare vector index types (flat, IVF, HNSW) and explain how cuVS accelerates similarity search in Milvus.
6. Implement hybrid retrieval combining dense (embedding) and sparse (BM25) retrieval, and explain when each dominates.
7. Articulate why reranking improves retrieval quality and describe the cross-encoder architecture of Llama 3.2 NV RerankQA.
8. Map the AI-Q Blueprint components to their roles in a production RAG deployment.

---

## D. Required Concepts

Before starting this module, students must understand:

- **Transformer architecture basics**: attention mechanism, encoder vs decoder models, and why some models produce embeddings while others generate text.
- **Vector similarity**: cosine similarity, dot product, L2 distance. What it means for two vectors to be "close."
- **Basic information retrieval**: the concept of a query, a document corpus, and relevance scoring.
- **Container fundamentals**: Docker images, containers, volumes, and Docker Compose multi-service orchestration.
- **GPU memory basics**: why model size and batch size constrain what fits on a given GPU (relevant for understanding H100 requirements).

---

## E. Core Lesson Content

### RAG Pipeline Anatomy [BEGINNER]

A RAG pipeline is a data processing pipeline with seven stages. Each stage transforms the data in a specific way, and each stage has failure modes that degrade the entire system.

```
RAG Pipeline — Full Architecture with NVIDIA Components

 ┌─────────────────────────────────────────────────────────────────────┐
 │                        INGESTION LAYER                             │
 │  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────┐  │
 │  │  Raw Docs    │───>│  PaddleOCR   │───>│   NeMo Curator       │  │
 │  │  (PDF, HTML, │    │  (text       │    │   (GPU-accelerated   │  │
 │  │   images)    │    │   extraction)│    │    dedup, filter,    │  │
 │  └──────────────┘    └──────────────┘    │    language detect)   │  │
 │                                          └──────────┬───────────┘  │
 ├─────────────────────────────────────────────────────┼──────────────┤
 │                       CHUNKING LAYER                │              │
 │                                          ┌──────────▼───────────┐  │
 │                                          │  Chunking Engine     │  │
 │                                          │  (fixed / recursive  │  │
 │                                          │   / semantic /       │  │
 │                                          │   structure-aware)   │  │
 │                                          └──────────┬───────────┘  │
 ├─────────────────────────────────────────────────────┼──────────────┤
 │                      EMBEDDING LAYER                │              │
 │                                          ┌──────────▼───────────┐  │
 │                                          │  NIM Embedding       │  │
 │                                          │  Endpoint            │  │
 │                                          │  (Llama 3.2 NV      │  │
 │                                          │   EmbedQA 1B)       │  │
 │                                          └──────────┬───────────┘  │
 ├─────────────────────────────────────────────────────┼──────────────┤
 │                       STORAGE LAYER                 │              │
 │                                          ┌──────────▼───────────┐  │
 │                                          │  Milvus + cuVS      │  │
 │                                          │  (GPU-accelerated   │  │
 │                                          │   vector index)     │  │
 │                                          └──────────┬───────────┘  │
 ├═════════════════════════════════════════════════════╪══════════════┤
 │                     QUERY-TIME PATH                 │              │
 │  ┌──────────┐    ┌──────────┐           ┌──────────▼───────────┐  │
 │  │  User    │───>│  Embed   │──────────>│  Retrieve (dense +   │  │
 │  │  Query   │    │  Query   │           │  sparse / hybrid)    │  │
 │  └──────────┘    └──────────┘           └──────────┬───────────┘  │
 │                                          ┌──────────▼───────────┐  │
 │                                          │  Rerank              │  │
 │                                          │  (Llama 3.2 NV      │  │
 │                                          │   RerankQA 1B)      │  │
 │                                          └──────────┬───────────┘  │
 │                                          ┌──────────▼───────────┐  │
 │                                          │  Generate            │  │
 │                                          │  (Llama 3.3 Nemotron│  │
 │                                          │   Super 49B /       │  │
 │                                          │   Llama 3.3 70B)    │  │
 │                                          └──────────────────────┘  │
 └─────────────────────────────────────────────────────────────────────┘
```

The pipeline splits into two phases: **indexing time** (Ingest through Store — runs offline, potentially on a schedule) and **query time** (Retrieve through Generate — runs per-request, latency-sensitive). Optimizing these phases requires different strategies.

### Document Ingestion and Preprocessing [BEGINNER]

Raw documents arrive in heterogeneous formats: PDF, HTML, Word, scanned images, markdown, plain text. Before any NLP processing, you must extract clean text.

**PaddleOCR** (used in the AI-Q Blueprint) handles text extraction from document images and scanned PDFs. It provides detection, recognition, and layout analysis. For structured documents (tables, forms), layout-aware extraction preserves the spatial relationships that carry meaning.

**NeMo Curator** provides GPU-accelerated data preparation operations:
- **Deduplication**: exact and fuzzy dedup at scale (MinHash LSH). Critical for avoiding embedding the same content multiple times.
- **Language identification**: filter documents by language.
- **Quality filtering**: heuristic and classifier-based filtering to remove low-quality text.
- **PII redaction**: detect and remove personally identifiable information before indexing.

NeMo Curator processes data on GPUs using RAPIDS/cuDF, achieving 10-100x speedups over CPU-based pipelines for large corpora. This matters when your corpus is millions of documents, not hundreds.

### Chunking Strategies [INTERMEDIATE]

Chunking is the most under-appreciated stage of RAG. Poor chunking directly causes poor retrieval.

**Fixed-size chunking**: Split text into chunks of N tokens (e.g., 512 tokens) with an overlap of M tokens (e.g., 64 tokens). Simple to implement. Breaks semantic boundaries arbitrarily. A paragraph about quarterly revenue might be split across two chunks, and neither chunk contains the complete thought.

**Recursive chunking**: Split on semantic boundaries in priority order — first by section headers, then by paragraphs, then by sentences, then by tokens. Preserves document structure better than fixed-size. LangChain's `RecursiveCharacterTextSplitter` is a common implementation.

**Semantic chunking**: Use an embedding model to detect topic shifts. Adjacent sentences with high similarity stay in the same chunk; a drop in similarity triggers a split. More expensive (requires embedding at chunk time) but produces semantically coherent chunks.

**Document-structure-aware chunking**: Parse the document's native structure (HTML headers, PDF sections, markdown headings, table boundaries) and chunk accordingly. Preserves the author's intended organization. Requires format-specific parsers.

**Tradeoffs table:**

| Strategy | Coherence | Implementation Cost | Compute Cost | Best For |
|---|---|---|---|---|
| Fixed-size | Low | Trivial | Minimal | Prototyping, uniform text |
| Recursive | Medium | Low | Minimal | General-purpose |
| Semantic | High | Medium | Moderate (embedding) | Mixed-topic documents |
| Structure-aware | Highest | High (per format) | Minimal | Structured docs (manuals, reports) |

**Chunk size tradeoff**: Too large (>1024 tokens) and the chunk contains too much off-topic content, diluting the signal. Too small (<128 tokens) and the chunk lacks sufficient context for the LLM to use. The sweet spot depends on your domain; 256-512 tokens is a common starting range.

### Embedding [INTERMEDIATE]

Embedding models convert text into dense vector representations where semantic similarity maps to geometric proximity. The choice of embedding model determines the quality ceiling of your retrieval.

**Llama 3.2 NV EmbedQA** is a 1B parameter embedding model used in the AI-Q Blueprint. It is trained specifically for question-answering retrieval: given a question, it produces embeddings that are close to embeddings of passages containing the answer. This is distinct from general-purpose sentence similarity models.

**NIM embedding endpoints** serve embedding models as microservices with:
- Batched inference for throughput
- Dynamic batching to handle variable request rates
- GPU acceleration with TensorRT-LLM optimization
- OpenAI-compatible API format

**NAT embedder configuration** (inferred from NAT documentation patterns — not directly verified in NVIDIA docs):
NAT allows configuring which embedding model and endpoint to use for document indexing and query embedding. The configuration specifies the model name, endpoint URL, and dimensionality.

```python
# Example: calling a NIM embedding endpoint
import requests

response = requests.post(
    "http://localhost:8080/v1/embeddings",
    json={
        "model": "nvidia/nv-embedqa-e5-v5",
        "input": ["What is the capital of France?"],
        "encoding_format": "float"
    }
)
embedding = response.json()["data"][0]["embedding"]
# Returns a dense vector, e.g., 1024 dimensions
```

### Vector Stores and Indexing [INTERMEDIATE]

Once chunks are embedded, you need a data structure that supports fast approximate nearest neighbor (ANN) search over potentially millions of vectors.

**Milvus** is the vector database used in the AI-Q Blueprint. With **cuVS** (CUDA Vector Search), Milvus offloads similarity search to GPUs, achieving significant latency and throughput improvements over CPU-based search.

**Index types and tradeoffs:**

| Index | Search Quality | Speed | Memory | Build Time | Best For |
|---|---|---|---|---|---|
| FLAT | Exact (100%) | Slow | Low | None | <100K vectors, ground truth |
| IVF_FLAT | High (95-99%) | Fast | Medium | Medium | 100K-10M vectors |
| IVF_PQ | Moderate (90-95%) | Very fast | Low | Medium | >10M vectors, memory-constrained |
| HNSW | High (95-99%) | Very fast | High | Slow | Low-latency requirements |
| GPU_IVF_FLAT (cuVS) | High (95-99%) | Very fast | GPU mem | Fast (GPU) | GPU-available, high throughput |

**cuVS** accelerates both index building and search. For large-scale deployments (millions of vectors, high QPS), GPU-accelerated search can reduce p99 latency from milliseconds to sub-milliseconds.

**FAISS** is a commonly used alternative for local development and smaller deployments. It supports the same index types but runs on CPU (with optional GPU support). Useful for prototyping before deploying to Milvus.

### Retrieval Strategies [INTERMEDIATE]

**Dense retrieval**: Embed the query, search the vector index for nearest neighbors. Strengths: captures semantic meaning ("automobile" matches "car"). Weaknesses: can miss exact keyword matches that matter in technical domains.

**Sparse retrieval (BM25)**: Traditional term-frequency-based retrieval. Strengths: exact keyword matching, no embedding model needed, fast. Weaknesses: misses synonyms and paraphrases entirely.

**Hybrid retrieval**: Combine dense and sparse scores. Typically done with Reciprocal Rank Fusion (RRF) or weighted linear combination. This is the recommended approach for production systems because it captures both semantic similarity and keyword relevance.

```
Hybrid Retrieval Flow:

  Query ──┬──> Dense Retrieval (embedding + ANN search) ──> Top-K dense
           │
           └──> Sparse Retrieval (BM25 over inverted index) ──> Top-K sparse
                                                                     │
                         RRF / Weighted Fusion <─────────────────────┘
                                   │
                              Merged Top-K ──> Reranker
```

NAT retriever configuration allows specifying retrieval mode (dense, sparse, hybrid), the number of candidates to retrieve (top-K), and the fusion method.

### Reranking [ADVANCED]

Retrieval casts a wide net. Reranking sharpens it. A retriever might return 20-50 candidate passages; a reranker scores each passage against the query with a more expensive but more accurate model, then selects the top 3-5.

**Why reranking matters**: Bi-encoder retrieval (the embedding approach) encodes query and passage independently. This is fast but loses cross-attention between query and passage tokens. A cross-encoder reranker processes query and passage together, allowing token-level interaction. This consistently improves precision by 10-25%.

**Llama 3.2 NV RerankQA** is a 1B parameter cross-encoder reranker in the AI-Q Blueprint. It takes (query, passage) pairs and outputs a relevance score. At 1B parameters, it balances accuracy with latency — small enough to run inference quickly, large enough to capture nuanced relevance.

**Cross-encoder vs bi-encoder:**
- **Bi-encoder** (used for retrieval): encodes query and document separately. O(1) per query at search time (pre-computed document embeddings). Lower accuracy.
- **Cross-encoder** (used for reranking): encodes query-document pair jointly. O(N) per query where N is the number of candidates. Higher accuracy. Too expensive for full-corpus search, perfect for reranking 20-50 candidates.

### NeMo Retriever Microservice [ADVANCED]

NeMo Retriever packages the embedding, retrieval, and reranking stages into a managed microservice. It provides:
- Automatic embedding model serving with optimized inference
- Vector store management (index creation, updates, deletion)
- Retrieval API with configurable strategies
- Reranking integration

**When to use NeMo Retriever vs building your own**: Use NeMo Retriever when you want a turnkey solution and your requirements fit its supported configurations. Build your own when you need custom chunking, non-standard vector stores, domain-specific preprocessing, or fine-grained control over each pipeline stage. (Note: the exact feature boundaries of NeMo Retriever are evolving; check current NVIDIA documentation for the latest capabilities.)

### AI-Q Blueprint Reference Architecture [ADVANCED]

The AI-Q Blueprint is NVIDIA's complete reference implementation for an agentic RAG system. It demonstrates how the individual components fit together in a production configuration.

**Components:**
- **LLMs**: Llama 3.3 Nemotron Super 49B (reasoning), Llama 3.3 70B Instruct (general)
- **Embedding**: Llama 3.2 NV EmbedQA 1B
- **Reranking**: Llama 3.2 NV RerankQA 1B
- **Vector store**: Milvus with cuVS
- **OCR**: PaddleOCR
- **Orchestration**: NeMo Agent Toolkit
- **Web search**: Tavily integration
- **Infrastructure**: Minimum 2xH100 80GB GPUs for Docker Compose deployment

The 2xH100 requirement reflects the combined memory needs: the 49B reasoning model alone requires approximately 100GB in FP16, which spans both GPUs. The embedding and reranking models fit in the remaining GPU memory or run on separate inference endpoints. Always verify hardware requirements on the specific Blueprint page or repository before attempting full deployment. Requirements vary significantly across Blueprints.

---

## F. Terminology Box

| Term | Definition |
|---|---|
| **RAG** | Retrieval-Augmented Generation. An architecture that retrieves relevant documents and provides them as context to an LLM before generation. |
| **Chunk** | A segment of a document, sized for embedding and retrieval. Typically 128-1024 tokens. |
| **Embedding** | A dense vector representation of text in a continuous vector space where semantic similarity corresponds to geometric proximity. |
| **ANN** | Approximate Nearest Neighbor. Search algorithms that trade exact accuracy for speed when finding similar vectors. |
| **Bi-encoder** | An architecture that encodes query and document independently into vectors, then compares them by similarity. Fast but less accurate. |
| **Cross-encoder** | An architecture that encodes query and document together, allowing full cross-attention. More accurate but slower. |
| **Reranking** | A second-stage scoring step that re-orders retrieved candidates using a more accurate model. |
| **cuVS** | CUDA Vector Search. NVIDIA's GPU-accelerated library for vector similarity search, integrated into Milvus. |
| **BM25** | Best Matching 25. A probabilistic sparse retrieval algorithm based on term frequency and inverse document frequency. |
| **RRF** | Reciprocal Rank Fusion. A method for combining ranked lists from multiple retrieval methods. |
| **IVF** | Inverted File Index. A vector index that partitions vectors into clusters and searches only relevant clusters. |
| **HNSW** | Hierarchical Navigable Small World. A graph-based vector index offering high recall and low latency. |

---

## G. Common Misconceptions

1. **"RAG eliminates hallucinations."** RAG reduces hallucinations grounded in factual recall, but it does not eliminate them. The LLM can still misinterpret retrieved passages, synthesize incorrect conclusions from correct facts, or hallucinate when the retrieved context is insufficient. RAG shifts the problem from "the model doesn't know" to "the model might misuse what it was given."

2. **"Bigger chunks are always better because they provide more context."** Larger chunks contain more information but also more noise. When a 2000-token chunk is retrieved because 50 tokens in the middle match the query, the surrounding 1950 tokens dilute the signal and can mislead the LLM. Chunk size must be tuned to the retrieval granularity your use case requires.

3. **"You can skip reranking to reduce latency."** Skipping reranking saves 50-200ms per query. However, without reranking, you must either retrieve fewer candidates (risking missed relevant documents) or pass more candidates to the LLM (increasing cost and latency elsewhere, and potentially confusing the model with irrelevant context). Reranking almost always improves end-to-end quality.

4. **"Any embedding model works for RAG."** Embedding models trained for semantic textual similarity (STS) are not optimized for retrieval. A model trained for retrieval (like NV EmbedQA) maps questions to answer-containing passages, which is a different task than mapping similar sentences to nearby points. Using the wrong model type can reduce retrieval recall by 20-30%.

5. **"GPU-accelerated vector search is only for massive datasets."** While the cost/benefit ratio improves with scale, cuVS provides latency benefits even at moderate scale (1M+ vectors) under high query throughput. The decision depends on QPS requirements, not just dataset size.

6. **"Once you build the index, you're done."** Indexes become stale as source documents change. Production RAG systems require incremental index updates, document version tracking, and periodic full re-indexing. A stale index is a silent accuracy degradation.

---

## H. Failure Modes / Anti-Patterns

1. **Chunk size mismatch with query type**: Using 1024-token chunks when users ask precise factual questions (which are answered in a single sentence). The retriever matches on one sentence but delivers 1023 tokens of noise. Fix: match chunk size to expected query granularity.

2. **No metadata filtering**: Retrieving across the entire corpus when the query implies a specific source, time range, or document type. Example: "What did Q3 2025 earnings say about margins?" retrieves passages from Q1 2024. Fix: extract metadata at ingestion time and apply filters before or during retrieval.

3. **Wrong embedding model for the domain**: Using a general-purpose embedding model for a highly technical domain (legal, medical, code) where domain-specific terminology dominates. Fix: evaluate retrieval recall on a domain-specific test set before committing to a model.

4. **Flat index in production at scale**: Using FLAT (brute-force) index with millions of vectors, resulting in linear scan times. Works fine in development with 10K vectors, collapses at scale. Fix: use IVF, HNSW, or GPU-accelerated indexes for any dataset above 100K vectors.

5. **Ignoring ingestion pipeline failures**: OCR errors, encoding issues, and truncated documents silently inject garbage into the index. The system retrieves garbage, and the LLM generates confident-sounding garbage. Fix: validate extraction quality with sampling, log ingestion errors, monitor retrieval quality metrics.

6. **Embedding query and documents with different models**: The query embedding and document embedding must come from the same model (or compatible models trained together). Mixing models produces vectors in incompatible spaces. Fix: use the same model and version for both indexing and querying.

7. **Over-retrieving without reranking**: Retrieving top-50 passages and passing all of them to the LLM without reranking. The LLM's context window fills with marginally relevant content, and answer quality degrades. Fix: retrieve broadly, rerank, and pass only top-3-5 to the LLM.

---

## I. Hands-On Lab

**Lab: Build a RAG Pipeline with NVIDIA Components**

Ingest a set of technical documents (PDF format). Use PaddleOCR for text extraction, implement recursive chunking with configurable chunk sizes, generate embeddings using a NIM embedding endpoint (or a local model as fallback), store embeddings in Milvus, implement hybrid retrieval (dense + BM25), add reranking, and generate answers using an LLM. Evaluate retrieval quality by comparing top-K recall with and without reranking on a prepared set of 20 question-answer pairs.

Full lab specification is in a separate file.

---

## J. Stretch Lab

**Stretch: Adaptive Chunking with Quality Feedback**

Implement a chunking evaluation loop: for a given set of queries with known relevant passages, automatically test multiple chunking strategies (fixed-256, fixed-512, recursive, semantic) and measure retrieval recall for each. Build a report showing which chunking strategy produces the best retrieval quality for your specific corpus. Bonus: implement incremental index updates that re-chunk and re-embed only changed documents.

---

## K. Review Quiz

**1.** In a RAG pipeline, which stage is responsible for converting text into dense vector representations?
**Answer:** The Embedding stage. It uses an embedding model (such as Llama 3.2 NV EmbedQA) to convert text chunks into fixed-dimensional dense vectors.

**2.** What is the primary advantage of hybrid retrieval over pure dense retrieval?
**Answer:** Hybrid retrieval combines the semantic matching capability of dense retrieval with the exact keyword matching of sparse retrieval (BM25). This captures both paraphrases/synonyms and exact technical terms, improving recall across a wider range of query types.

**3.** Why is a cross-encoder reranker more accurate than a bi-encoder retriever, and why can't it replace the retriever?
**Answer:** A cross-encoder processes query and passage jointly, allowing full cross-attention between all tokens. This is more accurate but has O(N) cost per query (must process every candidate pair). It cannot replace the retriever because scoring the entire corpus with a cross-encoder is computationally prohibitive. The retriever efficiently narrows candidates; the reranker refines them.

**4.** What GPU-accelerated library does Milvus integrate for vector similarity search in the AI-Q Blueprint?
**Answer:** cuVS (CUDA Vector Search).

**5.** Name two preprocessing operations that NeMo Curator performs on GPUs.
**Answer:** (Any two of) Deduplication (exact/fuzzy), language identification, quality filtering, PII redaction.

**6.** A team deploys a RAG system and finds that retrieval quality is good for general questions but poor for questions containing specific product codes. What is the most likely cause?
**Answer:** The dense embedding model does not represent rare/specific tokens (like product codes) well. Sparse retrieval (BM25) would match product codes exactly. The fix is to enable hybrid retrieval so that exact keyword matches are captured.

**7.** What is the minimum GPU requirement for deploying the AI-Q Blueprint via Docker Compose?
**Answer:** 2x NVIDIA H100 80GB GPUs. Always verify hardware requirements on the specific Blueprint page or repository before attempting full deployment. Requirements vary significantly across Blueprints.

**8.** Explain the tradeoff between IVF_FLAT and HNSW index types.
**Answer:** IVF_FLAT has lower memory usage and faster build times but requires tuning the number of clusters (nlist) and probes (nprobe). HNSW has higher memory usage (stores a graph structure) but provides consistently high recall with low latency and fewer tuning parameters. HNSW is preferred for low-latency requirements; IVF_FLAT is preferred when memory is constrained.

**9.** What problem does document-structure-aware chunking solve that fixed-size chunking does not?
**Answer:** Fixed-size chunking breaks text at arbitrary token boundaries, splitting semantic units (paragraphs, sections, tables) across chunks. Document-structure-aware chunking respects the document's native organization, keeping semantically coherent units intact, which improves retrieval precision.

**10.** A RAG system was deployed 6 months ago. Users report that answers about recent policy changes are wrong. The LLM is current. What is the most likely issue?
**Answer:** The vector index is stale — it was built from documents available at deployment time and has not been updated with recent policy documents. The fix is to implement incremental index updates or scheduled re-indexing.

---

## L. Mini Project

**Project: Domain-Specific RAG Quality Benchmark**

Build a RAG evaluation harness for a specific domain (e.g., NVIDIA documentation). Collect 30 question-answer pairs where the answer is present in the corpus. Implement three configurations: (1) dense retrieval only, no reranking; (2) hybrid retrieval, no reranking; (3) hybrid retrieval with reranking. Measure and report: top-5 recall, top-10 recall, mean reciprocal rank (MRR), and end-to-end answer accuracy (LLM-as-judge or exact match). Produce a comparison table and a one-page analysis explaining which configuration is best for your domain and why.

---

## M. How This May Appear on the Exam

1. **Component identification**: Given a diagram or description of a RAG pipeline, identify which NVIDIA tool or model serves each stage. Expect questions about NeMo Retriever, NIM embedding endpoints, Milvus/cuVS, and the reranking model.

2. **Tradeoff selection**: "A customer has 50 million documents and requires sub-10ms retrieval latency. Which index type and deployment strategy should they use?" Expect questions requiring you to choose between index types and justify GPU-accelerated search.

3. **Failure diagnosis**: "A RAG system retrieves relevant passages but the LLM generates incorrect answers. Which pipeline stage is most likely at fault?" Expect questions that test understanding of how each stage contributes to end-to-end quality.

4. **AI-Q Blueprint architecture**: Questions about the specific models, infrastructure requirements, and component roles in the AI-Q Blueprint. Know the model names, sizes, and GPU requirements.

5. **Chunking strategy selection**: Given a description of a document corpus and query patterns, select the appropriate chunking strategy and justify the choice.

---

## N. Checklist for Mastery

The student should be able to:

- [ ] Draw the seven-stage RAG pipeline from memory and label each NVIDIA component
- [ ] Explain why reranking improves retrieval quality with a concrete example
- [ ] Describe the difference between bi-encoder and cross-encoder architectures
- [ ] List four chunking strategies and state when each is preferred
- [ ] Configure a NIM embedding endpoint and make an embedding API call
- [ ] Explain what cuVS does and when GPU-accelerated vector search is justified
- [ ] Compare at least three vector index types by recall, speed, and memory
- [ ] Implement hybrid retrieval using both dense and sparse signals
- [ ] Diagnose a stale index problem from symptoms described by users
- [ ] Describe the AI-Q Blueprint components and their infrastructure requirements
- [ ] Explain why using mismatched embedding models for queries and documents fails
- [ ] Articulate the role of NeMo Curator in a production data preparation pipeline
