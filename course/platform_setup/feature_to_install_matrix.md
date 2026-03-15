# Feature-to-Install Matrix

> **Important:** Package names, extras, and dependency structures may change between NAT releases. This matrix is based on NAT v1.5 documentation. Always verify against the current NAT installation docs at [docs.nvidia.com/nemo/agent-toolkit](https://docs.nvidia.com/nemo/agent-toolkit) before installing.

This file tells you exactly what to install for each feature used in this course. The base `nvidia-nat` package is **not** sufficient for all labs. Evaluation, profiling, LangChain integration, and other features may require extras or separate packages.

---

## Full Matrix

| Feature | Package(s) | Install Command | Credentials | Hardware | Used In |
|---------|-----------|-----------------|-------------|----------|---------|
| NAT core (agent types, YAML config, functions, middleware) | `nvidia-nat` | `pip install nvidia-nat` | NVIDIA_API_KEY (for NVIDIA-hosted LLMs) | CPU laptop | M1, M2, M3, M5, M6, M7, Labs 1-7 |
| NAT + LangChain integration | `nvidia-nat` with langchain extra or langchain plugin | `pip install nvidia-nat[langchain]` (verify against current docs -- extras may be named differently) | NVIDIA_API_KEY | CPU laptop | M6, Lab 6 |
| NAT + LlamaIndex integration | `nvidia-nat` with llamaindex extra | `pip install nvidia-nat[llamaindex]` (verify against current docs) | NVIDIA_API_KEY | CPU laptop | M6 (reference only) |
| NAT + CrewAI integration | `nvidia-nat` with crewai extra | `pip install nvidia-nat[crewai]` (verify against current docs) | NVIDIA_API_KEY | CPU laptop | M6 (reference only) |
| NAT evaluation framework (RAG eval, red teaming, trajectory eval, custom evaluators, dataset loading) | `nvidia-nat` with eval extra OR `nvidia-nat-eval` (check current docs -- evaluation may be a separate package or extra in v1.5+) | `pip install nvidia-nat[eval]` OR `pip install nvidia-nat nvidia-nat-eval` (verify against current docs) | NVIDIA_API_KEY | CPU laptop | M8, Labs 8-9 |
| NAT profiler and sizing calculator | `nvidia-nat` with profiling extra OR included in eval package (check current docs) | `pip install nvidia-nat[profiling]` OR may be included in eval extra (verify) | NVIDIA_API_KEY | CPU laptop | M8, Labs 2, 6, 8 |
| NAT observability (Phoenix, Weave, Langfuse exporters) | `nvidia-nat` + observability backend | `pip install nvidia-nat arize-phoenix` (for Phoenix) or `pip install nvidia-nat langfuse` (for Langfuse) | NVIDIA_API_KEY + Langfuse API key (if using Langfuse hosted) | CPU laptop | M11, Labs 6, 11 |
| NAT finetuning (DPO with NeMo Customizer) | `nvidia-nat` with finetuning extra (check current docs) | `pip install nvidia-nat[finetuning]` (verify) | NVIDIA_API_KEY + NGC API key (for NeMo Customizer) | GPU required (multi-GPU for large models) | M8 (advanced/optional) |
| NAT deployment (FastAPI server, MCP server, A2A server) | `nvidia-nat` (server features may be in base package or require extras) | `pip install nvidia-nat` (verify server dependencies in current docs) | NVIDIA_API_KEY | CPU laptop for dev; Docker recommended for production-like deployment | M10, Labs 7, 10 |
| NeMo Guardrails — library mode, chat CLI (with NVIDIA-hosted models) | `nemoguardrails` + `langchain-nvidia-ai-endpoints` | `pip install nemoguardrails langchain-nvidia-ai-endpoints` (`langchain-nvidia-ai-endpoints` is required for `engine: nim` or `engine: nvidia_ai_endpoints` to reach NVIDIA-hosted models; without it, config loads but model calls fail) | NVIDIA_API_KEY | CPU laptop | M9, Labs 9, 12, platform_setup/first_guardrails_run.md |
| NeMo Guardrails — server mode (`nemoguardrails server`) | `nemoguardrails` with server extra | `pip install nemoguardrails[server]` (server has its own dependencies — FastAPI, uvicorn, etc. — that are not in the base package; verify extra name against current Guardrails release notes) | NVIDIA_API_KEY | CPU laptop | M9 (Step 10 of first_guardrails_run), Lab 9, M10 |
| NeMo Guardrails (microservice mode -- heavy mode) | NeMo Guardrails container from NGC | Docker pull from nvcr.io/nvidia (requires NGC API key) | NGC API key + NVIDIA_API_KEY | Docker required; GPU recommended for NIM-backed safety models; Kubernetes for production | M10 (advanced), M9 (reference only) |
| LangChain NVIDIA AI Endpoints | `langchain-nvidia-ai-endpoints` | `pip install langchain-nvidia-ai-endpoints` | NVIDIA_API_KEY | CPU laptop | M4, Lab 4, platform_setup/first_nim_api_call.md |
| Vector store -- FAISS (local, no GPU) | `faiss-cpu` | `pip install faiss-cpu` | None | CPU laptop | M4, Lab 4 |
| Vector store -- Milvus (GPU-accelerated with cuVS) | `pymilvus` + Milvus server (Docker) | `pip install pymilvus` + `docker compose` for Milvus server | None (local Milvus) | Docker required; GPU recommended for cuVS acceleration | M4 (advanced), Lab 4 (optional), Capstone 1 |
| Redis (for NAT memory persistence) | `redis` (Python client) | `pip install redis` + `docker run -p 6379:6379 redis` for Redis server | None (local Redis) | Docker required | M3, Lab 3, Lab 10 |
| Phoenix (observability UI) | `arize-phoenix` | `pip install arize-phoenix` then `python -m phoenix.server.main serve` | None (runs locally) | CPU laptop | M11, Lab 11, platform_setup/blueprint_walkthrough.md |
| OpenAI Python client (for NIM OpenAI-compatible endpoint) | `openai` | `pip install openai` | NVIDIA_API_KEY (used as OpenAI API key with NVIDIA base URL) | CPU laptop | platform_setup/first_nim_api_call.md |

---

## Quick Install for Each Phase

**Phase 1 -- Foundations (M1-M3):**

```bash
pip install nvidia-nat openai
```

**Phase 2 -- Development (M4-M7):**

```bash
pip install nvidia-nat[langchain] langchain-nvidia-ai-endpoints faiss-cpu redis
```

**Phase 3 -- Quality (M8-M9):**

```bash
pip install nvidia-nat[eval] nemoguardrails langchain-nvidia-ai-endpoints arize-phoenix
```

**Phase 4 -- Production (M10-M12):**

Docker required for deployment labs. See the individual lab instructions for specific container setup.

> **Note:** These phase groupings are approximate. Verify each extra name against the current NAT documentation before installing. Extras and package names may differ from what is listed here depending on your NAT version.
