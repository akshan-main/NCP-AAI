# Learning Path and Study Guide

## How to Use This Course

### If You Have 10 Weeks
Follow the phases in order. Complete every lab. Do both capstones. Spend the final week on mock exam prep.

### If You Have 6 Weeks (Intensive)
- Week 1: Modules 1-3
- Week 2: Modules 4-5
- Week 3: Modules 6-7
- Week 4: Modules 8-9
- Week 5: Modules 10-12
- Week 6: Module 13 + Capstones + Mock Exam

### If You Already Know Agent Basics
Skip Module 1 lesson content (still do the review quiz to verify). Start at Module 2. You will save ~5 hours.

### If You Already Know RAG
Skim Module 4 lesson content. Focus on the NVIDIA-specific parts (NeMo Retriever, Curator). Do Lab 4 regardless — it uses NVIDIA tooling you likely haven't used.

## Weekly Checkpoints

Each week should end with:
1. **All labs for that week completed** — code pushed to your course repo
2. **Review quizzes passed** — aim for 80%+ before moving on
3. **One oral explanation exercise** — explain a key concept from that week out loud for 3 minutes without notes. Record yourself. If you stumble, you don't know it well enough.
4. **Architecture sketch** — for any module that introduces architecture patterns, draw the pattern from memory

## Study Principles

### The Explanation Test
For every concept, ask: "Could I explain this in a technical interview with a principal engineer?" If yes, move on. If no, go deeper.

### The Implementation Test
For every tool/framework, ask: "Could I build a working prototype using this in 2 hours without documentation open?" If yes, you've practiced enough. If no, do another lab iteration.

### The Debugging Test
For every system, ask: "If this broke in production at 2 AM, could I identify the failure layer?" If yes, you understand the architecture. If no, study failure modes.

## Lab Progression Overview

Labs build on each other. Each lab produces an artifact that later labs extend.

```
Lab 1: Basic Agent Loop (standalone)
  └── Lab 2: Architecture Pattern Implementation
      └── Lab 3: Planning + Memory Integration
          └── Lab 4: RAG Pipeline with NeMo Retriever
              └── Lab 5: Tool-Using Agent
                  └── Lab 6: NeMo Agent Toolkit Pipeline
                      └── Lab 7: Multi-Agent System
                          └── Lab 8: Evaluation Pipeline
                              └── Lab 9: Guardrails + Safety
                                  └── Lab 10: NIM Deployment
                                      └── Lab 11: Monitoring + Observability
                                          └── Lab 12: Human-in-the-Loop Integration
```

**Capstone 1** builds on Labs 4-6, 8-9 (RAG + agent + eval + safety)
**Capstone 2** builds on Labs 7, 10-12 (multi-agent + deployment + monitoring + oversight)

## Required Tools and Environment Setup

### Software
- Python 3.10+
- Docker and Docker Compose
- Git
- NVIDIA GPU (recommended: 1x A100 or H100 for deployment labs; cloud instances acceptable). Always verify hardware requirements on the specific Blueprint page or repository before attempting full deployment. Requirements vary significantly across Blueprints.
  - Labs 1-9 can run on CPU or with NVIDIA API access (build.nvidia.com)
  - Labs 10-12 and capstones benefit from local GPU access
- `pip install nvidia-nat` (NeMo Agent Toolkit)

> **Install note**: The base `nvidia-nat` package covers core agent types, YAML configuration, functions, and middleware. If your lab also uses evaluation, profiling, LangChain integration, or other advanced features, install the required extras — see `platform_setup/feature_to_install_matrix.md` for the full mapping.

- `pip install nemoguardrails langchain-nvidia-ai-endpoints` (NeMo Guardrails — library mode; microservice mode via NGC container is optional/advanced)
- `pip install langchain langchain-nvidia-ai-endpoints`
- Vector database: Milvus or FAISS
- Kubernetes (minikube acceptable for labs; production deployment needs a real cluster)

### NVIDIA API Access
- Create an account at build.nvidia.com
- Obtain API keys for NIM endpoints
- Free tier is sufficient for most labs
- Deployment labs may require NVIDIA AI Enterprise or cloud GPU instances

### Recommended Repo Structure
```
ncp-aai-course/
├── labs/
│   ├── lab_01_agent_loop/
│   ├── lab_02_architecture/
│   ├── lab_03_planning_memory/
│   ├── lab_04_rag_pipeline/
│   ├── lab_05_tool_agent/
│   ├── lab_06_nat_pipeline/
│   ├── lab_07_multi_agent/
│   ├── lab_08_evaluation/
│   ├── lab_09_guardrails/
│   ├── lab_10_deployment/
│   ├── lab_11_monitoring/
│   └── lab_12_human_loop/
├── capstones/
│   ├── capstone_01_rag_agent/
│   └── capstone_02_multi_agent/
├── notes/
├── configs/
└── README.md
```

## Recommended NVIDIA Training Courses (Official)

These are the 5 NVIDIA-recommended courses for NCP-AAI. Take them alongside or before this course.

| Course | Cost | Hours | When to Take | What It Adds |
|--------|------|-------|-------------|--------------|
| Building RAG Agents With LLMs | $90 | 8 | During Modules 4-5 | Hands-on RAG pipeline with LangChain, FAISS, Gradio, evaluation |
| Evaluating RAG and Semantic Search Systems | $30 | 3 | During Module 8 | RAG evaluation metrics, semantic search quality |
| Building Agentic AI Applications With LLMs | $90 | 8 | During Modules 6-7 | Agent orchestration, multi-step reasoning, tool use |
| Adding New Knowledge to LLMs | $500 | 8 | During Module 4 | Fine-tuning, knowledge injection, NeMo Customizer |
| Deploying RAG Pipelines for Production at Scale | $90 | 8 | During Module 10 | Production RAG deployment, scaling, NIM |
