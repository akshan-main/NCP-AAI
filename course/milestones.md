# Mandatory NVIDIA Platform Interaction Milestones

Every learner must complete all 8 milestones before attempting the NCP-AAI exam. Each milestone proves real NVIDIA platform interaction, not just theoretical knowledge.

## Interaction Level Classifications

| Level | Description | Example |
|-------|-------------|---------|
| Level 1 | NVIDIA-hosted API interaction | build.nvidia.com NIM endpoints |
| Level 2 | NVIDIA tooling interaction | NeMo Agent Toolkit (NAT) local |
| Level 3 | NVIDIA safety stack interaction | NeMo Guardrails with Colang 2.0 |
| Level 4 | NVIDIA reference architecture interaction | NVIDIA Blueprints |
| Level 5 | Optional advanced infrastructure | Self-hosted NIM, Kubernetes deployment |

---

## Milestone 1: First Successful NIM Call

**When:** After platform setup

**Classification:** Level 1

**Deliverables:**
- API key configured and stored securely (NVIDIA_API_KEY environment variable set)
- First hosted model call completed via curl
- First hosted model call completed via Python
- Prompt and response captured and saved to a file
- Latency noted for the call (time from request to first token, total time)

---

## Milestone 2: First NAT Workflow

**When:** After Module 1 / Lab 1

**Classification:** Level 2

**Deliverables:**
- NeMo Agent Toolkit installed locally and verified
- YAML workflow configuration created from scratch (not just copied)
- Successful tool-using agent execution demonstrated
- Execution trace saved and reviewed

---

## Milestone 3: Agent Pattern Comparison

**When:** After Module 2 / Lab 2

**Classification:** Level 2

**Deliverables:**
- 3 or more NAT agent types run on the same task
- Comparison table produced with the following columns:
  - Agent type
  - Token usage
  - Latency
  - Accuracy / correctness
  - Trace complexity (number of steps, tool calls)
- Written decision rationale explaining which agent type is best suited for the task and why

---

## Milestone 4: RAG on NVIDIA Stack

**When:** After Module 4 / Lab 4

**Classification:** Level 1 + Level 2

**Deliverables:**
- Document corpus selected and ingested
- NIM embedding endpoint used to generate embeddings
- Vector store populated with embedded documents
- Full RAG pipeline working: retrieval + reranking + generation
- Retrieval quality measured using recall@k metric

---

## Milestone 5: Guardrails in Action

**When:** After Module 9 / Lab 9

**Classification:** Level 3

**Deliverables:**
- NeMo Guardrails installed and configured
- At least one rail written in Colang 2.0 that blocks unsafe input
- Before/after comparison showing guarded vs unguarded responses
- Adversarial test results documented (at least 3 adversarial prompts tested)

---

## Milestone 6: Blueprint Reproduction

**When:** After platform setup Blueprint walkthrough

**Classification:** Level 4

**Deliverables:**
- AI-Q Blueprint studied in detail (architecture, components, data flow)
- Architecture comparison document: full Blueprint vs course reproduction
- Reproduced subset implemented:
  - NIM embedding endpoint integration
  - Local vector store setup
  - NIM generation endpoint integration
- Deviations from the full Blueprint documented with justifications

---

## Milestone 7: Observable Agent System

**When:** After Module 11 / Lab 11

**Classification:** Level 2

**Deliverables:**
- NAT observability configured with Phoenix or Langfuse
- Traces captured for 5 or more queries
- Token usage and latency breakdown documented per query
- At least one failure identified and debugged using trace data

---

## Milestone 8: Integrated Capstone

**When:** After Capstone 1 or 2

**Classification:** Level 1 + Level 2 + Level 3 + Level 4

**Deliverables:**
- End-to-end NVIDIA-aligned agent system running
- Evaluation results produced and analyzed
- Safety controls verified and documented
- Deployment configuration written (containerization, environment, scaling plan)
- Human oversight design documented (escalation paths, approval gates, monitoring)

---

## Milestone Tracking

| Milestone | Name | Completed | Date | Notes |
|-----------|------|-----------|------|-------|
| 1 | First Successful NIM Call | [ ] | | |
| 2 | First NAT Workflow | [ ] | | |
| 3 | Agent Pattern Comparison | [ ] | | |
| 4 | RAG on NVIDIA Stack | [ ] | | |
| 5 | Guardrails in Action | [ ] | | |
| 6 | Blueprint Reproduction | [ ] | | |
| 7 | Observable Agent System | [ ] | | |
| 8 | Integrated Capstone | [ ] | | |
