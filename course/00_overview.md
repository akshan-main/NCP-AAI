# NCP-AAI: Production Agentic AI on the NVIDIA Stack

## Course Title
**Architecting, Building, and Governing Production Agentic AI Systems — NVIDIA Certified Professional Preparation**

## Course Promise
This course teaches you to architect, implement, evaluate, deploy, and govern production-grade agentic AI systems using the NVIDIA ecosystem. By the end, you will be able to design agent architectures from first principles, build them using NeMo Agent Toolkit and associated NVIDIA tools, evaluate them rigorously, deploy them at scale on NIM microservices, and enforce safety through NeMo Guardrails — and you will be prepared to pass the NCP-AAI certification exam.

## Prerequisites
- **Python**: Comfortable with OOP, decorators, async/await, packaging. You will write real systems, not scripts.
- **ML/LLM Fundamentals**: Understand what a transformer is, what tokenization does, what inference means. You do not need to train models from scratch.
- **Basic API Usage**: Can call REST APIs, read JSON responses, use `requests` or `httpx`.
- **Command Line**: Comfortable with terminal, git, pip/conda, Docker basics.
- **Optional but helpful**: Familiarity with LangChain or any agent framework. Experience with vector databases. Kubernetes basics for deployment modules.

## What This Course Is NOT
- Not a motivational overview of AI trends
- Not a LangChain tutorial with NVIDIA branding
- Not a survey of 15 agent frameworks
- Not a course that teaches you to call APIs and call it "production"

## Learning Outcomes
Upon completion, you will be able to:

1. **Explain and compare** agent architecture patterns (ReAct, Plan-and-Execute, function-calling, router, hierarchical) with tradeoffs for each
2. **Design** reasoning, planning, and memory systems appropriate to task complexity
3. **Build** RAG pipelines using NeMo Retriever, embedding models, and vector stores with proper chunking, reranking, and evaluation
4. **Develop** agents using NeMo Agent Toolkit with YAML configuration, tool registries, MCP support, and framework integration
5. **Implement** multi-agent systems with coordination, communication, and shared state
6. **Evaluate** agent systems using NeMo Evaluator, LLM-as-Judge, RAG metrics (faithfulness, relevance, recall), and trajectory analysis
7. **Configure** NeMo Guardrails with Colang 2.0 for input/output/dialog/topical/execution rails
8. **Apply** the 4-stage safety lifecycle (Evaluate → Post-Train → Deploy → Runtime) as a conceptual model, implemented through current NVIDIA tooling: NeMo Guardrails, NeMo Auditor, Safety NIMs, and NAT defense/red-teaming middleware
9. **Deploy** agent systems on NIM microservices with Kubernetes/Helm, NVIDIA Dynamo optimization, and proper scaling
10. **Monitor** production agent systems with OpenTelemetry, NeMo Agent Toolkit telemetry, and structured observability
11. **Design** human-in-the-loop oversight, escalation paths, and feedback loops
12. **Pass** the NCP-AAI certification exam with confidence

## Estimated Duration
- **Total modules**: 13
- **Estimated hours**: 90-110 hours
  - Core lesson content: ~40 hours
  - Labs and exercises: ~30 hours
  - Capstone projects: ~15 hours
  - Review and certification prep: ~10 hours
- **Recommended pace**: 8-10 weeks at 10-12 hours/week
- **Intensive pace**: 5-6 weeks at 18-20 hours/week

## Module Dependency Graph

```
M1 (Foundations)
├── M2 (Architecture Patterns)
│   ├── M3 (Reasoning, Planning, Memory)
│   │   └── M7 (Multi-Agent Systems)
│   ├── M5 (Tools & Prompt Engineering)
│   │   └── M6 (NeMo Agent Toolkit & Orchestration)
│   │       └── M7 (Multi-Agent Systems)
│   └── M4 (Knowledge Integration & RAG)
│       └── M6 (NeMo Agent Toolkit & Orchestration)
├── M8 (Evaluation & Tuning) ← requires M4, M5, M6
│   └── M9 (Safety & Guardrails) ← requires M8
├── M10 (Deployment & Scaling) ← requires M6
│   └── M11 (Observability & Monitoring) ← requires M10
│       └── M12 (Human-AI Interaction) ← requires M9, M11
└── M13 (Certification Prep) ← requires all
```

## Recommended Study Order

### Phase 1: Foundations (Weeks 1-2)
- Module 1: Foundations of Agentic AI
- Module 2: Agent Architecture Patterns
- Module 3: Reasoning, Planning, and Memory

### Phase 2: Core Development (Weeks 3-5)
- Module 4: Knowledge Integration and RAG Pipelines
- Module 5: Agent Development — Tools and Prompt Engineering
- Module 6: Agent Development — NeMo Agent Toolkit and Orchestration
- Module 7: Multi-Agent Systems

### Phase 3: Quality and Safety (Weeks 6-7)
- Module 8: Evaluation and Tuning
- Module 9: Safety, Guardrails, and Governance

### Phase 4: Production (Weeks 8-9)
- Module 10: Deployment and Scaling
- Module 11: Observability, Monitoring, and Maintenance
- Module 12: Human-AI Interaction and Oversight

### Phase 5: Integration and Prep (Week 10)
- Module 13: Certification Prep and Integration
- Capstone finalization
- Mock exam practice

## Module Summary Table

| # | Module | Hours | Exam Domains Covered | NVIDIA Tools |
|---|--------|-------|---------------------|--------------|
| 1 | Foundations: What Makes a System Agentic | 5 | Arch (15%), Cognition (10%) | NeMo stack overview, NIM API |
| 2 | Agent Architecture Patterns | 7 | Arch (15%) | Blueprint reference architectures |
| 3 | Reasoning, Planning, and Memory | 7 | Cognition (10%) | NeMo Agent Toolkit, NIM models |
| 4 | Knowledge Integration and RAG | 8 | Knowledge (10%) | NeMo Retriever, Curator, embedding NIMs |
| 5 | Tools and Prompt Engineering | 7 | Development (15%) | Hyperparameter Optimizer, prompt opt |
| 6 | NeMo Agent Toolkit and Orchestration | 8 | Development (15%), Platform (7%) | NAT YAML, tool registry, MCP |
| 7 | Multi-Agent Systems | 7 | Arch (15%), Development (15%) | Warehouse Blueprint |
| 8 | Evaluation and Tuning | 8 | Evaluation (13%) | NeMo Evaluator, Data Flywheel |
| 9 | Safety, Guardrails, Governance | 8 | Safety (5%) | Guardrails, Colang, Safety NIMs, NeMo Auditor, NAT defense middleware |
| 10 | Deployment and Scaling | 8 | Deployment (13%) | NIM, Dynamo, NIM Operator, K8s |
| 11 | Observability and Monitoring | 6 | Monitor (5%) | OpenTelemetry, NAT telemetry |
| 12 | Human-AI Interaction and Oversight | 5 | Human-AI (5%) | Guardrails, escalation patterns |
| 13 | Certification Prep | 6 | All (100%) | Full stack |

## NVIDIA Platform Interaction Requirement

This course requires real interaction with NVIDIA-hosted services and NVIDIA tooling. Every module contains a "Hands-on NVIDIA Platform Interaction" section with concrete exercises, deliverables, and verification steps.

**Interaction levels** (cumulative):
- **Level 1**: NVIDIA-hosted API (build.nvidia.com, NIM endpoints) — required for all learners
- **Level 2**: NVIDIA tooling (NAT local installation + workflows) — required for all learners
- **Level 3**: NVIDIA safety stack (NeMo Guardrails + Colang 2.0) — required for all learners
- **Level 4**: NVIDIA reference architectures (Blueprint study + partial reproduction) — required for all learners
- **Level 5**: Self-hosted NIM, Kubernetes, NIM Operator — optional, clearly marked, requires GPU hardware

**Start here**: `platform_setup/build_nvidia_account_and_api_key.md` before any module.

**Accelerated paths**: See `minimum_nvidia_path.md` for 3-day, 7-day, and 14-day paths to hands-on NVIDIA exposure.

**Quality gate**: See `quality_gate.md` for the 29-item completion checklist. If any item is "no", the course is incomplete.

**Milestones**: See `milestones.md` for the 8 mandatory checkpoints that prove platform interaction through artifacts.
