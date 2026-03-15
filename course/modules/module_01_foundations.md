# Module 1: Foundations — What Makes a System Agentic

**Primary exam domains:** Architecture (15%), Cognition (10%)
**NVIDIA tools:** NeMo stack overview, NIM API orientation, `pip install nvidia-nat` setup

---

## A. Module Title

**Foundations — What Makes a System Agentic**

---

## B. Why This Module Matters in Real Systems

Most teams building with large language models end up with one of two outcomes: a fragile demo that breaks on real inputs, or an over-engineered pipeline that could have been a prompt template. The difference between those outcomes and a reliable production system starts with understanding what "agentic" actually means — not as a marketing term, but as a set of architectural properties that determine how your system perceives its environment, reasons about what to do, takes actions, and observes results. Without this foundation, every subsequent design decision is guesswork.

This module matters because the NVIDIA certification tests whether you can distinguish between systems that are genuinely agentic and systems that merely call tools in a fixed sequence. That distinction is not academic. An agentic system can recover from unexpected tool failures, adapt its plan when new information arrives, and operate with varying degrees of autonomy. A pipeline cannot. If you build a pipeline when you need an agent, you will spend months patching edge cases. If you build an agent when you need a pipeline, you will burn tokens and introduce unnecessary nondeterminism. Knowing which is which saves engineering time and infrastructure cost.

The NVIDIA ecosystem provides a full stack for building agentic systems — from inference (NIM) to orchestration (NAT) to safety (NeMo Guardrails) to evaluation (NeMo Evaluator). This module gives you a map of that stack so that when subsequent modules dive deep into each component, you already know where it fits. You will make your first NIM API call and install the NeMo Agent Toolkit. Everything that follows builds on this.

---

## C. Learning Objectives

By the end of this module, you will be able to:

1. Define "agentic" precisely and identify the four phases of the agent loop (Perceive, Reason, Act, Observe).
2. Classify a given AI system as a chatbot, pipeline, single-step tool caller, multi-step reasoner, planner, or multi-agent coordinator.
3. List the components of the NVIDIA agentic AI stack and describe the role of each (NIM, NAT, NeMo Guardrails, NeMo Evaluator, NeMo Retriever, NeMo Curator, NeMo Customizer).
4. Execute a successful API call to an NVIDIA NIM endpoint and interpret the response.
5. Install and verify the NeMo Agent Toolkit (`nvidia-nat`) in a local development environment.
6. Identify at least five differences between a toy agent demo and a production agent system.
7. Draw the NVIDIA agentic stack from memory and explain data flow between components.

---

## D. Required Concepts

Before starting this module, you should be comfortable with:

- **Python 3.10+**: Functions, classes, async/await, package installation with pip, virtual environments.
- **REST APIs**: HTTP methods, request/response structure, JSON payloads, authentication headers.
- **LLM basics**: What a large language model is, what tokens are, what a prompt is, what inference means.
- **Command-line tools**: Terminal navigation, environment variables, running scripts.
- **JSON and YAML**: Reading and writing both formats.

If any of these are unfamiliar, address them before proceeding. This course does not teach Python or REST fundamentals.

---

## E. Core Lesson Content

### [BEGINNER] What "Agentic" Actually Means

An AI agent is a system that can **perceive** its environment, **reason** about what to do, **take actions** that affect the environment, and **observe** the results of those actions to inform its next step. This is the agent loop:

```
┌──────────────────────────────────────────────┐
│                  AGENT LOOP                  │
│                                              │
│   ┌───────────┐      ┌───────────┐          │
│   │ PERCEIVE  │─────>│  REASON   │          │
│   │ (inputs,  │      │ (analyze, │          │
│   │  context) │      │  decide)  │          │
│   └───────────┘      └─────┬─────┘          │
│         ^                  │                 │
│         │                  v                 │
│   ┌───────────┐      ┌───────────┐          │
│   │  OBSERVE  │<─────│    ACT    │          │
│   │ (check    │      │ (call     │          │
│   │  result)  │      │  tools)   │          │
│   └───────────┘      └───────────┘          │
│                                              │
│   Loop continues until goal is met or        │
│   a termination condition is reached.        │
└──────────────────────────────────────────────┘
```

The critical property is the **loop**. A chatbot receives a prompt and returns a response — one pass, no loop. A pipeline executes a fixed sequence of steps — no decision-making between steps. An agent decides, after each action, whether to continue, change approach, or stop. That decision-making loop is what makes a system agentic.

**The autonomy spectrum:**

| Level | Description | Example |
|-------|-------------|---------|
| 0 | Fixed pipeline | ETL job with LLM summarization step |
| 1 | Single-step tool call | Chatbot that calls a weather API when asked |
| 2 | Multi-step tool use | System that searches, reads results, searches again |
| 3 | Planning + execution | System that decomposes a task, creates a plan, executes steps |
| 4 | Multi-agent coordination | Multiple specialized agents collaborating on a complex task |
| 5 | Fully autonomous | System that sets its own goals and operates indefinitely |

Most production systems operate at levels 1-3. Level 5 is research territory and generally not what the exam covers. The exam focuses on levels 1-4.

### [BEGINNER] Taxonomy of Agent Types

**Single-step tool callers** receive a user query, decide which tool to call, call it once, and return a result. This is the simplest agentic behavior. Example: a customer support bot that looks up an order status.

**Multi-step reasoners** can call multiple tools in sequence, using the output of one call to inform the next. They reason about what to do at each step. Example: a research assistant that searches a knowledge base, reads the top result, extracts a date, then queries a calendar API with that date.

**Planners** decompose a complex task into subtasks before executing any of them. They create an explicit plan, then execute it. Example: a coding agent that first outlines the files to change, then makes each change, then runs tests.

**Multi-agent coordinators** route tasks to specialized sub-agents and aggregate results. Example: a financial analysis system with separate agents for data retrieval, quantitative analysis, and report generation.

### [INTERMEDIATE] The NVIDIA Agentic AI Stack

NVIDIA provides a layered stack for building agentic systems. Here is how the components relate:

```
┌─────────────────────────────────────────────────────────┐
│                    APPLICATION LAYER                     │
│          Your agentic application / workflow             │
├─────────────────────────────────────────────────────────┤
│                  ORCHESTRATION LAYER                     │
│    NeMo Agent Toolkit (NAT)                             │
│    - Agent types (ReAct, Reasoning, ReWOO, Router...)   │
│    - Tool management, memory, context handling          │
├──────────┬──────────┬───────────┬───────────────────────┤
│ SAFETY   │ EVAL     │ RETRIEVAL │ DATA & CUSTOMIZATION  │
│ NeMo     │ NeMo     │ NeMo      │ NeMo Curator (data)   │
│ Guard-   │ Evalu-   │ Retriever │ NeMo Customizer       │
│ rails    │ ator     │ (RAG)     │ (fine-tuning)         │
├──────────┴──────────┴───────────┴───────────────────────┤
│                   INFERENCE LAYER                        │
│    NVIDIA NIM (model serving, API endpoints)            │
├─────────────────────────────────────────────────────────┤
│                   INFRASTRUCTURE                         │
│    NVIDIA GPUs, CUDA, TensorRT-LLM                      │
└─────────────────────────────────────────────────────────┘
```

**Component roles:**

- **NVIDIA NIM** (NVIDIA Inference Microservices): Serves LLMs as API endpoints. You send a prompt, you get a completion. NIM handles batching, quantization, and GPU optimization. This is the engine that powers every agent's "thinking."
- **NeMo Agent Toolkit (NAT)**: The orchestration framework. NAT provides pre-built agent patterns (ReAct, ReWOO, Router, etc.), tool integration, memory management, and configuration via YAML. This is where you define what your agent does.
- **NeMo Guardrails**: Safety layer. Defines rules for what the agent can and cannot do, input/output filtering, topic control. Sits between the user and the agent or between the agent and its tools.
- **NeMo Evaluator**: Benchmarking and evaluation. Measures agent performance across accuracy, latency, safety, and cost metrics.
- **NeMo Retriever**: RAG (Retrieval-Augmented Generation) infrastructure. Embedding models, vector databases, retrieval pipelines.
- **NeMo Curator**: Data processing pipeline for preparing training and fine-tuning data.
- **NeMo Customizer**: Fine-tuning service for adapting base models to specific domains.

### [INTERMEDIATE] First NIM API Call

Before building agents, you must be able to call the inference layer directly. Here is a minimal NIM API call:

```python
import requests

# NIM endpoint — replace with your deployed endpoint or NVIDIA API Catalog URL
NIM_URL = "https://integrate.api.nvidia.com/v1/chat/completions"
API_KEY = "nvapi-..."  # Your NVIDIA API key

response = requests.post(
    NIM_URL,
    headers={
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
    },
    json={
        "model": "meta/llama-3.1-70b-instruct",
        "messages": [
            {"role": "user", "content": "What is an AI agent?"}
        ],
        "max_tokens": 256,
        "temperature": 0.2,
    },
)

result = response.json()
print(result["choices"][0]["message"]["content"])
```

Key observations:
- NIM exposes an **OpenAI-compatible API**. If you have used the OpenAI Python client, you can point it at a NIM endpoint with minimal changes.
- The `model` field specifies which model NIM serves. In production, this is typically a model you deployed on your own infrastructure.
- `temperature` controls randomness. For agentic systems, lower temperature (0.0-0.3) is typical for tool-calling decisions; higher temperature may be used for creative generation steps.

### [INTERMEDIATE] Installing and Verifying NAT

```bash
# Create a virtual environment (recommended)
python -m venv ncp-aai-env
source ncp-aai-env/bin/activate

# Install NAT
pip install nvidia-nat

# Verify installation
python -c "import nat; print(nat.__version__)"
```

> **Install note**: The base `nvidia-nat` package covers core agent types, YAML configuration, functions, and middleware. If your lab also uses evaluation, profiling, LangChain integration, or other advanced features, install the required extras — see `platform_setup/feature_to_install_matrix.md` for the full mapping.

After installation, verify you can import core components:

```python
from nat.agents import ReActAgent
from nat.tools import Tool
from nat.memory import InMemoryObjectStore

print("NAT installation verified successfully.")
```

**Note:** NAT is under active development. The exact import paths and class names may change between releases. Always check the version-specific documentation. The examples in this course are based on the NAT version available at time of writing and may require minor adjustments.

### [ADVANCED] Toy Demos vs. Production Agent Systems

A demo agent that works in a notebook is separated from a production agent by at least these concerns:

| Concern | Demo | Production |
|---------|------|------------|
| **Latency** | Seconds acceptable | Sub-second for user-facing; SLA-bound |
| **Error handling** | Crashes on failure | Retries, fallbacks, graceful degradation |
| **Observability** | Print statements | Structured logging, tracing (OpenTelemetry), metrics |
| **Safety** | Trust all inputs | Input validation, guardrails, output filtering |
| **Cost** | Unlimited tokens | Token budgets, model routing, caching |
| **Concurrency** | Single user | Thousands of concurrent sessions |
| **Memory** | In-process Python dict | Redis, MySQL, or S3-backed persistent storage |
| **Testing** | Manual spot-checks | Automated regression suites, eval benchmarks |
| **Deployment** | Local notebook | Containerized, orchestrated, health-checked |

The exam tests awareness of these production concerns. You do not need to implement all of them in the labs, but you must know they exist and understand the tradeoffs.

### [ADVANCED] Architecture Decision: When to Make a System Agentic

Not every LLM application should be agentic. Making a system agentic introduces nondeterminism (the agent might take different paths on the same input), higher latency (multiple LLM calls per request), and higher cost (more tokens consumed). Use an agentic architecture when:

- The task requires **dynamic decision-making** that cannot be hardcoded.
- The number of possible tool sequences is too large to enumerate.
- The system must **adapt** to novel situations without code changes.
- **Multi-step reasoning** is required to produce a correct answer.

Do not use an agentic architecture when:
- A fixed pipeline produces correct results reliably.
- Latency requirements are extremely tight (single-digit milliseconds).
- The task is purely generative with no tool use.
- Deterministic output is required for regulatory or compliance reasons.

---

## F. Terminology Box

| Term | Definition |
|------|-----------|
| **Agent** | A system that perceives, reasons, acts, and observes in a loop to accomplish a goal. |
| **Agent loop** | The cyclic process of Perceive → Reason → Act → Observe that defines agentic behavior. |
| **NIM** | NVIDIA Inference Microservices — serves LLMs as optimized API endpoints. |
| **NAT** | NeMo Agent Toolkit — NVIDIA's framework for building agentic AI applications. |
| **NeMo Guardrails** | Safety framework for controlling agent behavior via programmable rules. |
| **NeMo Evaluator** | Benchmarking tool for measuring agent accuracy, latency, safety, and cost. |
| **NeMo Retriever** | RAG infrastructure: embeddings, vector stores, retrieval pipelines. |
| **NeMo Curator** | Data processing pipeline for preparing training data. |
| **NeMo Customizer** | Fine-tuning service for adapting models to specific domains. |
| **Autonomy spectrum** | The range from fixed pipelines (no autonomy) to fully autonomous agents. |
| **Tool calling** | The ability of an LLM to invoke external functions or APIs. |
| **Token budget** | The maximum number of tokens an agent is allowed to consume per request. |

---

## G. Common Misconceptions

1. **"Any system that uses an LLM is an agent."** Incorrect. A system is agentic only if it has a decision-making loop. An LLM that summarizes text in a single pass is not an agent — it is an inference call.

2. **"Agents are always better than pipelines."** Incorrect. Agents introduce nondeterminism, latency, and cost. If a fixed pipeline solves the problem reliably, use the pipeline. Agents are for tasks where the path to the answer cannot be predetermined.

3. **"NIM is an agent framework."** Incorrect. NIM is an inference service. It serves model completions. NAT is the agent framework that orchestrates those completions into agentic behavior.

4. **"Agentic means autonomous."** Partially incorrect. Autonomy is a spectrum. A system that calls one tool based on LLM reasoning is minimally agentic. Full autonomy (self-directed goal-setting) is the far end of the spectrum and is rarely deployed in production.

5. **"Adding tool calling to a chatbot makes it an agent."** Only if the chatbot decides dynamically which tool to call and can loop on the result. If the tool-calling logic is hardcoded (e.g., always call the weather API when the user mentions weather), that is a pipeline with an LLM step, not an agent.

6. **"The NVIDIA agentic stack requires NVIDIA GPUs."** For NIM inference endpoints, yes — NIM is optimized for NVIDIA GPUs. However, NAT itself (the orchestration layer) runs on any Python environment. You can also use NVIDIA's cloud-hosted NIM endpoints without owning GPUs.

---

## H. Failure Modes / Anti-Patterns

1. **Agent-washing a pipeline.** Calling a fixed sequence of LLM calls an "agent" when there is no decision-making loop. This leads to confusion when the system cannot handle unexpected inputs — because it was never designed to.

2. **Unbounded agent loops.** Allowing an agent to loop indefinitely without a maximum iteration count or token budget. In production, this burns money and can hang user requests. Always set termination conditions.

3. **Skipping observability from the start.** Building an agent without structured logging or tracing, then trying to debug failures in production by reading print statements. Add tracing from day one.

4. **Ignoring latency compounding.** Each step in an agent loop requires an LLM call (100-2000ms each). A 5-step agent loop can take 5-10 seconds. Teams that do not account for this in their SLA commitments end up rearchitecting.

5. **Using the most powerful model for every step.** Not every agent step requires a 70B parameter model. Routing steps, classification steps, and simple extraction steps can often use smaller, faster models. Failing to use model routing wastes cost and latency.

6. **No fallback behavior.** When a tool call fails (API timeout, malformed response), the agent must have a recovery strategy. Without one, a single tool failure crashes the entire workflow.

---

## I. Hands-On Lab

**Lab 1: First Contact with the NVIDIA Agentic Stack**

- Install NAT in a virtual environment.
- Obtain an NVIDIA API key from the NVIDIA API Catalog.
- Make a direct NIM API call using `requests` and parse the response.
- Make the same call using the OpenAI Python client pointed at NIM.
- Instantiate a basic NAT agent (ReAct) with a single tool (a calculator function).
- Run the agent on three test queries and observe the agent loop in the logs.

Full lab specifications are in a separate file.

---

## J. Stretch Lab

**Stretch: Agent vs. Pipeline Comparison**

Build two systems that answer the same question ("What is the current weather in the capital of France?"):
1. A fixed pipeline: hardcode the steps (look up capital of France → call weather API).
2. An agent: give the agent a country-to-capital tool and a weather tool, let it figure out the steps.

Compare: latency, token usage, correctness on 10 varied inputs (e.g., "What's the weather where the Eiffel Tower is?"). Document when the agent outperforms the pipeline and vice versa.

---

## K. Review Quiz

**1.** Which phase of the agent loop involves calling an external tool or API?
a) Perceive b) Reason c) Act d) Observe
**Answer:** c) Act. The Act phase is where the agent executes an action, such as calling a tool.

**2.** What distinguishes an agent from a pipeline?
a) Agents use LLMs; pipelines do not. b) Agents have a decision-making loop; pipelines have a fixed sequence. c) Agents are faster. d) Pipelines cannot use APIs.
**Answer:** b) The decision-making loop is the defining characteristic.

**3.** In the NVIDIA stack, which component serves LLMs as optimized API endpoints?
a) NAT b) NIM c) NeMo Guardrails d) NeMo Evaluator
**Answer:** b) NIM (NVIDIA Inference Microservices).

**4.** What is the role of NeMo Guardrails in the agentic stack?
a) Model fine-tuning b) Data curation c) Safety and behavior control d) Inference optimization
**Answer:** c) Safety and behavior control.

**5.** A system that always calls Tool A, then Tool B, then Tool C in that order is best classified as:
a) A multi-step reasoner b) A planner c) A pipeline d) A multi-agent coordinator
**Answer:** c) A pipeline — there is no dynamic decision-making.

**6.** Why is low temperature (0.0-0.3) typically used for tool-calling decisions in agents?
a) It is faster. b) It reduces nondeterminism in critical decision steps. c) NIM requires it. d) It uses fewer tokens.
**Answer:** b) Tool-calling decisions should be deterministic; low temperature reduces randomness.

**7.** Which of the following is NOT a production concern for agent systems?
a) Token budgets b) Structured logging c) Using the largest available model for all steps d) Graceful degradation on tool failure
**Answer:** c) Using the largest model for all steps is an anti-pattern, not a production best practice.

**8.** NAT runs on:
a) Only NVIDIA GPUs b) Any Python environment c) Only Linux d) Only NVIDIA DGX systems
**Answer:** b) NAT is a Python orchestration framework that runs on any Python environment.

**9.** At what autonomy level does a system that decomposes a task into subtasks and executes them operate?
a) Level 1 b) Level 2 c) Level 3 d) Level 5
**Answer:** c) Level 3 — Planning + execution.

**10.** Which NVIDIA component would you use to measure an agent's accuracy and latency?
a) NeMo Curator b) NeMo Customizer c) NeMo Evaluator d) NeMo Retriever
**Answer:** c) NeMo Evaluator.

---

## L. Mini Project

**Project: Agent Type Classifier**

Build a decision-tree tool (can be a simple Python script or flowchart) that takes as input a description of an AI system and outputs:
1. Whether it is agentic or not (with justification).
2. Its position on the autonomy spectrum (Level 0-5).
3. Which agent type it most closely resembles (single-step tool caller, multi-step reasoner, planner, multi-agent coordinator).
4. Which NVIDIA stack components would be involved in building it.

Test your classifier on at least 5 different system descriptions. Document where the classification is ambiguous and why.

---

## M. How This May Appear on the Exam

1. **Scenario-based classification:** You are given a description of a system (e.g., "A customer service bot that checks inventory, processes returns, and escalates to human agents based on sentiment analysis") and asked to identify its agent type and position on the autonomy spectrum.

2. **Stack component matching:** A question presents a requirement (e.g., "You need to ensure the agent never discusses competitor products") and asks which NVIDIA component addresses it (NeMo Guardrails).

3. **Architecture diagram interpretation:** You may be shown a diagram of an agentic system and asked to identify which component is missing or misconfigured.

4. **Production readiness:** A question describes a demo agent and asks what must change for production deployment. Expect answers involving observability, error handling, token budgets, and safety.

5. **Agent loop identification:** Given a sequence of system actions, identify which phase of the agent loop each action belongs to.

---

## N. Checklist for Mastery

Before moving to Module 2, confirm you can:

- [ ] Define "agentic" without using the word "agent" in the definition.
- [ ] Draw the Perceive → Reason → Act → Observe loop from memory.
- [ ] Classify any given AI system on the autonomy spectrum (Levels 0-5).
- [ ] Name all seven components of the NVIDIA agentic stack and state each one's purpose in one sentence.
- [ ] Draw the NVIDIA agentic stack architecture diagram from memory.
- [ ] Execute a NIM API call and parse the response without consulting documentation.
- [ ] Install NAT and verify the installation by importing core classes.
- [ ] List at least five differences between a demo agent and a production agent.
- [ ] Explain when NOT to use an agentic architecture (and what to use instead).
- [ ] Identify the agent type (tool caller, reasoner, planner, coordinator) given a system description.
- [ ] Explain why unbounded agent loops are dangerous in production.
- [ ] Describe the relationship between NIM (inference) and NAT (orchestration) without conflating them.
