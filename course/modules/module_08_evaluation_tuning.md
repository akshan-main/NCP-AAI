# Module 8: Evaluation and Tuning

**Primary exam domains:** Evaluation and Tuning (13%)
**NVIDIA tools:** NAT Evaluation Framework, NAT Profiler, NAT Sizing Calculator, NAT Finetuning (DPO with NeMo Customizer, OpenPipe ART), NeMo Evaluator microservice, Data Flywheel Blueprint

---

## A. Module Title

**Evaluation and Tuning: Measuring, Profiling, and Optimizing Agentic Systems**

---

## B. Why This Module Matters in Real Systems

Evaluation is the hardest unsolved problem in agent engineering. Unlike traditional software where tests are deterministic, agents are non-deterministic, multi-step, and exhibit compound error rates. If each step in a 5-step agent workflow is 90% correct, the end-to-end accuracy is 0.9^5 = 59%. This makes rigorous evaluation not optional but existential — without it, you have no idea whether your system works.

The difficulty compounds because "works" is itself ambiguous for agents. A RAG system might retrieve the right documents but generate the wrong answer. An agent might use the right tools in the wrong order. It might produce a correct answer via a lucky path that will not generalize. Evaluation must cover components individually, the pipeline as a whole, the agent's decision trajectory, and system-level properties like latency and cost.

NVIDIA provides concrete tooling for this: the NAT Evaluation Framework for structured evaluation runs, the NAT Profiler for latency and token profiling, the NAT Sizing Calculator for resource planning, and finetuning paths (DPO via NeMo Customizer, OpenPipe ART) for when evaluation reveals that prompt engineering has hit its ceiling. This module teaches you to build an evaluation practice that drives principled improvement rather than guesswork.

---

## C. Learning Objectives

After completing this module, you will be able to:

1. Explain why agent evaluation requires a different approach than traditional software testing, citing compound error rates.
2. Distinguish four evaluation levels: component, pipeline, agent trajectory, and system.
3. Calculate and interpret retriever metrics (MRR, NDCG, Recall@k, Precision@k) for a RAG system.
4. Configure and run evaluations using the NAT Evaluation Framework, including dataset loading, evaluator selection, and custom evaluator development.
5. Use the NAT Profiler to identify token and latency bottlenecks at the workflow, agent, and tool level.
6. Configure NAT Red Teaming evaluators to find agent failure modes before production deployment.
7. Make principled decisions about when to finetune (DPO, OpenPipe ART) versus when to fix prompts, tools, or retrieval.
8. Describe the Data Flywheel Blueprint pattern and explain how it connects evaluation to continuous improvement.

---

## D. Required Concepts

- Completion of Module 5 (Agentic Reasoning) — you must understand agent workflows and tool use to evaluate them.
- Completion of Module 4 (RAG) — you must understand retrieval pipelines to evaluate retrieval quality.
- Basic statistics: precision, recall, ranking metrics.
- Familiarity with NAT configuration and deployment (Module 6).

---

## E. Core Lesson Content

### [BEGINNER] Why Agent Evaluation Is Hard

Three properties make agent evaluation fundamentally different from traditional software testing:

**Non-determinism.** The same input may produce different outputs across runs. Temperature, sampling, and model updates all introduce variance. You cannot write a unit test that checks for exact string equality.

**Multi-step compound error.** Each agent step (retrieval, reasoning, tool selection, tool execution, response generation) has its own error rate. These compound multiplicatively. A 5-step workflow with 95% per-step accuracy yields 77% end-to-end accuracy. A 10-step workflow drops to 60%.

**Ambiguous correctness.** For many agent tasks, there is no single correct answer. A customer service response can be helpful in multiple ways. A code generation task can have multiple valid solutions. Evaluation must capture quality on a spectrum, not binary pass/fail.

### [BEGINNER] Evaluation Taxonomy

```
EVALUATION LEVELS
=================

Level 1: COMPONENT         Level 2: PIPELINE
+----------+               +----------+    +----------+
| Retriever|               | Retriever|--->| Generator|
| alone    |               |          |    |          |
+----------+               +----------+    +----------+
  MRR, NDCG,                 Faithfulness, Answer
  Recall@k                   Relevance, Context metrics

Level 3: AGENT TRAJECTORY   Level 4: SYSTEM
+---+  +---+  +---+         +------------------------+
| T1|->| T2|->| T3|         | Latency | Cost | Uptime|
+---+  +---+  +---+         +------------------------+
  Tool sequence, error         End-to-end performance
  recovery, efficiency         under production load
```

**Level 1 — Component evaluation:** Test each component in isolation. Retriever accuracy, generator quality, individual tool reliability.

**Level 2 — Pipeline evaluation:** Test the RAG pipeline end-to-end. Does retrieving these specific chunks lead to a correct and grounded answer?

**Level 3 — Agent trajectory evaluation:** Evaluate the sequence of agent actions. Did it use the right tools in the right order? Did it recover from errors? Did it avoid unnecessary steps?

**Level 4 — System evaluation:** Latency, cost per query, reliability, throughput under load. These are operational metrics, not quality metrics.

### [INTERMEDIATE] Retriever Evaluation Metrics

*NOTE: These are standard information retrieval metrics applied to NVIDIA RAG pipelines. NVIDIA does not prescribe specific retriever metrics; these are industry standard.*

**MRR (Mean Reciprocal Rank):** For each query, find the rank of the first relevant result. MRR = average of 1/rank across all queries. MRR = 1.0 means the first result is always relevant. Use when you care most about the top result.

**NDCG (Normalized Discounted Cumulative Gain):** Measures ranking quality across all positions, with higher weight on top positions. Accounts for graded relevance (not just binary relevant/irrelevant). Use when ranking order matters across multiple results.

**Recall@k:** Of all relevant documents, what fraction appears in the top-k results? Use when you need to ensure important documents are not missed.

**Precision@k:** Of the top-k results, what fraction is relevant? Use when you want to minimize irrelevant noise in retrieved chunks.

**Practical guidance:** For RAG systems, Recall@k is often more important than Precision@k. Missing a relevant chunk means the generator cannot produce a correct answer. Including an irrelevant chunk wastes context window but may not degrade output quality if the generator is robust.

### [INTERMEDIATE] RAG Evaluation Metrics

*NOTE: These follow the RAGAS framework pattern. NAT confirms "RAG evaluation" support without specifying exact metric implementations.*

**Faithfulness:** Is the generated answer grounded in the retrieved context? A faithfulness failure means the LLM hallucinated information not present in the chunks. Measured by decomposing the answer into claims and checking each against the context.

**Answer Relevance:** Does the response actually answer the question asked? An answer can be faithful to the context but miss the point of the question.

**Context Precision:** Of the retrieved chunks, how many are actually relevant to answering the question? Low context precision means the retriever is returning noise.

**Context Recall:** Were all the chunks needed to fully answer the question actually retrieved? Low context recall means the retriever is missing critical information.

```
RAG Evaluation Matrix:
                    Answer Correct    Answer Wrong
Context Good     |  System works   |  Generator issue  |
Context Bad      |  Lucky/risky    |  Retriever issue  |
```

### [INTERMEDIATE] LLM-as-Judge

When human evaluation is too expensive or slow, use an LLM to judge another LLM's output.

**How it works:** A judge LLM receives the original query, the generated response, and optionally a reference answer. It scores the response on specified criteria (relevance, completeness, safety).

**Judge prompt design principles:**
- Specify evaluation criteria explicitly (do not ask "is this good?").
- Use rubrics with defined score levels (1-5 with descriptions for each level).
- Include examples of each score level when possible.
- Request structured output (JSON with score and reasoning).

**Calibration:** Run the judge on a sample with known human ratings. Check correlation. If the judge consistently disagrees with humans on a specific category, adjust the rubric.

**Bias awareness:** LLM judges tend to prefer longer responses, more formal language, and responses from the same model family. Mitigate by testing for these biases explicitly.

**NAT custom evaluator development** supports implementing LLM-as-Judge patterns within the evaluation framework, allowing you to define custom scoring logic and integrate it into evaluation pipelines.

### [INTERMEDIATE] Agent Trajectory Evaluation

Evaluating agents on final output alone misses critical information. An agent might reach the correct answer through an inefficient or unreliable path.

**What to evaluate in trajectories:**
- **Tool selection accuracy:** Did the agent choose the right tool for each step?
- **Tool call efficiency:** Did it minimize unnecessary tool calls?
- **Error recovery:** When a tool call failed, did the agent recover gracefully?
- **Ordering correctness:** Were dependent operations executed in the right sequence?
- **Termination:** Did the agent stop when the task was complete, or did it continue unnecessarily?

NAT confirms trajectory evaluation support. This allows you to define expected action sequences and compare them against actual agent behavior.

### [INTERMEDIATE] NAT Evaluation Framework

The NAT Evaluation Framework provides a structured approach to running evaluations:

**Workflow:**
1. **Load dataset:** Import evaluation datasets with queries, expected outputs, and optional metadata. NAT supports dataset loading and filtering to select subsets for targeted evaluation.
2. **Configure evaluators:** Select built-in evaluators (RAG eval, trajectory eval) or define custom evaluators.
3. **Run evaluation:** Execute the evaluation pipeline, which runs the agent on each dataset entry and scores the results.
4. **Interpret results:** Aggregate scores, identify failure patterns, and prioritize improvements.

**Custom evaluator development:** When built-in evaluators do not cover your use case, NAT allows you to write custom evaluators. This is the path for implementing domain-specific quality criteria, custom LLM-as-Judge configurations, or composite metrics.

**Dataset filtering:** Filter evaluation datasets by metadata (difficulty, category, source) to run targeted evaluations. This is critical for debugging — when overall scores drop, filter to identify which category is underperforming.

### [INTERMEDIATE] NAT Red Teaming Evaluators

Red teaming evaluators probe the agent for failure modes through adversarial inputs:

- **Prompt injection attempts:** Inputs designed to override system instructions.
- **Out-of-scope requests:** Queries the agent should refuse or redirect.
- **Edge cases:** Unusual inputs that test boundary behavior.
- **Multi-turn manipulation:** Sequences of inputs that gradually steer the agent into unsafe behavior.

Configure red teaming evaluators to run regularly — not just before launch. Agent behavior can drift as underlying models are updated or context changes.

### [ADVANCED] NAT Profiler Deep Dive

The NAT Profiler instruments entire workflows from the top level down to individual tool calls.

**What it measures:**
- Input/output token counts at each step (critical for cost estimation).
- Latency per step: LLM inference time, tool execution time, overhead.
- Breakdown by agent and by tool within each agent.

**Interpreting profiler output:**

```
Profiler Output (illustrative):
+---------------------+--------+--------+---------+
| Component           | Tokens | Latency| % Total |
+---------------------+--------+--------+---------+
| Router Agent        | 1,200  | 450ms  | 12%     |
|   -> LLM routing    | 800    | 380ms  | 10%     |
|   -> Overhead        | 400    | 70ms   | 2%      |
| Search Agent        | 4,500  | 1,800ms| 48%     |
|   -> Retrieval       | -      | 200ms  | 5%      |
|   -> LLM generation  | 3,200  | 1,500ms| 40%     |
|   -> Post-processing | 1,300  | 100ms  | 3%      |
| Response Agent      | 2,000  | 1,500ms| 40%     |
+---------------------+--------+--------+---------+
| TOTAL               | 7,700  | 3,750ms| 100%    |
+---------------------+--------+--------+---------+
```

From this, you can see that Search Agent's LLM generation is the dominant cost. Optimization options: use a smaller/faster model for search synthesis, reduce chunk count to lower token input, or cache frequent queries.

### [ADVANCED] NAT Sizing Calculator

The Sizing Calculator estimates resource requirements for a workflow based on expected traffic patterns, model sizes, and latency targets. Use it during capacity planning to determine GPU allocation, instance count, and scaling triggers.

### [ADVANCED] Finetuning: When and How

**When finetuning is the right response:**
- Evaluation shows consistent failures on a specific task type that prompt engineering cannot fix.
- The model lacks domain-specific knowledge that cannot be provided through RAG.
- You need to reduce output token count or latency by teaching the model a more concise response style.

**When finetuning is the wrong response:**
- The retriever is returning bad chunks (fix retrieval, not the model).
- The system prompt is poorly written (fix the prompt first — it is cheaper).
- Tool definitions are ambiguous (fix tool descriptions and schemas).
- You have fewer than 100 high-quality training examples.

**NAT Finetuning paths:**

**DPO (Direct Preference Optimization) with NeMo Customizer:** Align model behavior with human preferences by training on preference pairs (chosen vs. rejected responses). Effective for improving response style, safety compliance, and format adherence without requiring reward model training.

**OpenPipe ART:** Alternative finetuning path for rapid iteration. Use when you need faster turnaround on finetuning experiments.

**Decision framework:**
```
Evaluation failure detected
    |
    v
Is the root cause in retrieval? --> Fix retrieval pipeline
    |  No
    v
Is the root cause in the prompt? --> Rewrite system prompt
    |  No
    v
Is it a tool definition issue? --> Fix tool schemas
    |  No
    v
Do you have 100+ quality examples? --> No: Collect more data
    |  Yes
    v
Is it a style/format issue? --> DPO with NeMo Customizer
    |  No
    v
Is it a knowledge gap that RAG can't fill? --> Full finetuning
```

### [ADVANCED] Data Flywheel Blueprint

The Data Flywheel implements a continuous improvement loop:

```
DATA FLYWHEEL
=============
  +------------+
  | Production |
  | Traffic    |
  +-----+------+
        |
        v
  +-----+------+
  | Collect &  |
  | Filter Data|
  +-----+------+
        |
        v
  +-----+------+      +------------+
  | Evaluate   |----->| If good    |---> Continue
  | Quality    |      | enough     |
  +-----+------+      +------------+
        |
        v (if not)
  +-----+------+
  | Finetune / |
  | Retrain    |
  +-----+------+
        |
        v
  +-----+------+
  | Validate & |
  | Redeploy   |
  +-----+------+
        |
        +-------> Back to Production
```

**Key principle:** Production data is your best evaluation and training resource. The flywheel collects real user interactions, evaluates system performance on them, identifies failure patterns, and uses that data to improve through finetuning or pipeline changes.

### [ADVANCED] NeMo Evaluator Microservice

The NeMo Evaluator microservice provides evaluation capabilities as a scalable service for benchmarking and monitoring at scale. It is designed for integration into CI/CD pipelines and production monitoring systems.

*NOTE: Detailed API documentation for NeMo Evaluator may be thinner than NAT's built-in evaluation framework. For most users, the NAT Evaluation Framework is the primary evaluation tool; NeMo Evaluator is relevant for large-scale or microservice-native deployments.*

### [ADVANCED] Cost/Latency/Accuracy Tradeoff Framework

*NOTE: This is inferred best practice, not an NVIDIA-documented framework.*

Every optimization decision involves tradeoffs across three dimensions:

| Action | Cost | Latency | Accuracy |
|--------|------|---------|----------|
| Larger model | Higher | Higher | Usually higher |
| More retrieval chunks | Higher | Higher | Diminishing returns |
| Caching frequent queries | Lower (amortized) | Lower | Same or slightly stale |
| Model distillation | Lower | Lower | Lower but possibly sufficient |
| Finetuning smaller model | Upfront cost | Lower | Can match larger model on narrow tasks |

**Decision principle:** Define minimum acceptable accuracy first. Then optimize for cost and latency within that constraint. Never sacrifice accuracy below the threshold to save money — the cost of bad answers (user trust, rework) exceeds compute savings.

---

## Hands-on NVIDIA Platform Interaction

**Title: Run Your First NAT Evaluation**

**Requirements:**
- `NVIDIA_API_KEY`
- `nvidia-nat` installed locally
- The RAG pipeline from Module 4 / Lab 4 (or a minimal version)
- Execution: local NAT + hosted NIM endpoint

**Exercise:**

1. Create 10 test cases for your RAG pipeline: query + expected answer + relevant source document.
2. Configure NAT's evaluation framework to run RAG evaluation on these 10 test cases. Use NAT's dataset loading to load the test cases.
3. Run the evaluation. Inspect results: which queries passed, which failed, what retrieval quality looks like.
4. Run NAT Profiler on 5 of the test cases. Capture: tokens per step, latency per step, total cost estimate.
5. Write a 1-paragraph assessment: is your RAG pipeline production-ready based on these results?

**Deliverable:**
- (a) Evaluation results table (10 test cases, pass/fail, retrieval metrics).
- (b) Profiler output for 5 cases.
- (c) Production readiness assessment.

**Verification:** Evaluation framework runs without errors. You can identify which test cases fail and hypothesize why.

**Classification:** Local NVIDIA tooling interaction (NAT eval framework, NAT Profiler) + hosted NVIDIA API interaction (NIM endpoint for generation during evaluation)

---

## F. Terminology Box

| Term | Definition |
|------|-----------|
| **MRR** | Mean Reciprocal Rank — average of 1/(rank of first relevant result) across queries |
| **NDCG** | Normalized Discounted Cumulative Gain — ranking quality metric weighted toward top positions |
| **Recall@k** | Fraction of all relevant documents that appear in the top-k results |
| **Precision@k** | Fraction of top-k results that are relevant |
| **Faithfulness** | Whether a generated answer is grounded in the retrieved context (no hallucination) |
| **Answer Relevance** | Whether the response addresses the question that was actually asked |
| **LLM-as-Judge** | Using an LLM to evaluate another LLM's output quality |
| **Trajectory Evaluation** | Evaluating the sequence of agent actions, not just the final output |
| **DPO** | Direct Preference Optimization — finetuning method using human preference pairs |
| **Data Flywheel** | Continuous loop: collect production data, evaluate, retrain, redeploy |
| **Red Teaming** | Adversarial testing to find failure modes before production deployment |
| **NAT Profiler** | Tool for profiling workflows down to tool-level token counts and latency |

---

## G. Common Misconceptions

1. **"If the final answer is correct, the agent is working well."** A correct final answer reached through an unreliable path (wrong tools used, errors that happened to cancel out) is a ticking time bomb. Trajectory evaluation catches this.

2. **"High retriever recall means the RAG system is good."** Recall measures whether relevant documents are retrieved, not whether the generator uses them correctly. You can have perfect recall and terrible faithfulness.

3. **"LLM-as-Judge is as good as human evaluation."** LLM judges have systematic biases (preferring length, formality, same-family models). They are useful for scaling evaluation but must be calibrated against human judgments regularly.

4. **"Finetuning is the first thing to try when evaluation scores are low."** Finetuning should be the last resort. Fix retrieval, prompts, and tool definitions first — these are cheaper, faster, and more reversible.

5. **"You only need to evaluate before launch."** Agent behavior drifts over time as underlying models update, data distributions shift, and user behavior changes. Continuous evaluation is a production requirement.

6. **"More evaluation metrics means better evaluation."** Tracking 50 metrics creates noise. Identify the 3-5 metrics that most directly correlate with user satisfaction and system reliability for your use case. Track those rigorously.

---

## H. Failure Modes / Anti-Patterns

1. **Evaluating only happy paths.** Running evaluation exclusively on clean, well-formed queries misses the adversarial, ambiguous, and edge-case inputs that cause production failures. Always include adversarial and edge-case samples.

2. **Overfitting to evaluation sets.** When the same evaluation dataset is used repeatedly and the system is tuned against it, scores improve without generalization. Maintain held-out evaluation sets that are never used for tuning decisions.

3. **Ignoring latency in evaluation.** A system that produces perfect answers in 30 seconds is unusable for real-time applications. Always include latency as an evaluation dimension.

4. **Finetuning on bad data.** Using production logs without filtering for quality leads to training on the system's own mistakes. Apply quality filters before using production data for finetuning.

5. **Missing baseline measurements.** Starting optimization without measuring current performance means you cannot quantify improvement. Always establish baselines before making changes.

6. **Evaluating components in isolation only.** Component metrics can all look good while end-to-end performance is poor due to integration issues. Run both component and pipeline evaluations.

---

## I. Hands-On Lab

**Lab 8: End-to-End Evaluation Pipeline**

Using the NAT Evaluation Framework, build a complete evaluation pipeline for a RAG agent. Load an evaluation dataset, configure RAG evaluators (faithfulness, answer relevance, context precision), run the evaluation, and interpret the results. Identify the weakest component (retriever vs. generator) and propose a fix. Then use the NAT Profiler to profile the same workflow and identify the latency bottleneck.

---

## J. Stretch Lab

**Stretch Lab 8: Custom Evaluator and Red Teaming**

Write a custom NAT evaluator that scores agent responses on domain-specific criteria of your choice (e.g., regulatory compliance, technical accuracy in a specific field). Then configure NAT Red Teaming evaluators to run adversarial probes against the agent. Analyze red teaming results and implement guardrail changes based on findings. Measure evaluation score changes before and after the fixes.

---

## K. Review Quiz

**Q1:** An agent workflow has 8 steps, each with 92% accuracy. What is the approximate end-to-end accuracy?
**Answer:** 0.92^8 = approximately 51.3%. This illustrates why compound error rates make multi-step agent evaluation critical.

**Q2:** A RAG system retrieves all relevant documents but generates an answer containing information not in any retrieved chunk. Which metric captures this failure?
**(a)** Context Recall **(b)** Faithfulness **(c)** Precision@k **(d)** Answer Relevance
**Answer:** (b) Faithfulness. The answer is not grounded in the retrieved context.

**Q3:** When should you use MRR vs. NDCG for retriever evaluation?
**Answer:** Use MRR when you only care about the rank of the first relevant result (e.g., single-answer retrieval). Use NDCG when ranking quality across multiple positions matters (e.g., returning several relevant chunks for RAG).

**Q4:** What is the primary risk of using LLM-as-Judge without calibration?
**Answer:** Systematic biases (preference for longer responses, formal language, same-model-family outputs) lead to scores that do not correlate with actual quality.

**Q5:** The NAT Profiler shows that 70% of workflow latency comes from a single LLM generation step. Name two optimization approaches.
**Answer:** (1) Use a smaller/faster model for that step. (2) Reduce input token count by retrieving fewer chunks or compressing context.

**Q6:** What is the Data Flywheel pattern?
**Answer:** A continuous loop: collect production data, evaluate system performance on it, finetune or adjust based on findings, validate, and redeploy. Production data drives ongoing improvement.

**Q7:** True or False: DPO requires training a separate reward model.
**Answer:** False. DPO (Direct Preference Optimization) directly optimizes using preference pairs without a separate reward model, which is one of its advantages over RLHF.

**Q8:** Your evaluation shows low context recall but high faithfulness. What does this tell you?
**Answer:** The retriever is missing relevant documents, but when it does retrieve relevant chunks, the generator produces grounded answers. The fix should target retrieval (better embeddings, chunking, or query expansion), not generation.

**Q9:** What is trajectory evaluation, and why does it matter beyond final-answer evaluation?
**Answer:** Trajectory evaluation assesses the sequence of agent actions (tool selections, ordering, error recovery). It matters because an agent can reach a correct answer through an unreliable path that will fail on different inputs.

**Q10:** You have 30 examples of a specific agent failure. Is finetuning the right response?
**Answer:** Likely not. 30 examples is generally too few for effective finetuning. First, try fixing the system prompt, tool definitions, or retrieval. If finetuning is still needed, collect at least 100+ high-quality examples.

---

## L. Mini Project

**Project: Evaluation Dashboard**

Build a comprehensive evaluation pipeline for an agent of your choice. Implement: (1) Component-level retriever metrics (Recall@k, MRR) on a test dataset of at least 50 queries. (2) Pipeline-level RAG metrics (faithfulness, answer relevance). (3) An LLM-as-Judge custom evaluator for one domain-specific quality criterion. (4) NAT Profiler analysis with latency breakdown. Produce a summary report identifying the top 3 improvement priorities, ranked by expected impact, and a recommended action for each (prompt fix, retrieval fix, or finetuning).

---

## M. How This May Appear on the Exam

1. **Metric selection:** Given a specific RAG failure scenario, identify which evaluation metric would detect the issue (e.g., "the answer contains information not in the context" = faithfulness).

2. **Evaluation level selection:** Given a system symptom (e.g., "correct answers but slow and expensive"), identify which evaluation level (component, pipeline, agent, system) to investigate.

3. **Finetuning decision:** Given evaluation results and available data, determine whether finetuning is appropriate or whether prompt/retrieval fixes should come first.

4. **NAT tooling:** Questions about the NAT Profiler (what it measures), the NAT Evaluation Framework (how to configure it), and NAT Red Teaming (what it tests).

5. **Data Flywheel:** Describe the continuous improvement loop and identify which NVIDIA tools support each stage.

---

## N. Checklist for Mastery

- [ ] I can explain why compound error rates make agent evaluation harder than traditional software testing.
- [ ] I can distinguish the four evaluation levels (component, pipeline, agent trajectory, system) and give examples of metrics for each.
- [ ] I can calculate MRR, NDCG, Recall@k, and Precision@k given a set of retrieval results and relevance labels.
- [ ] I can evaluate a RAG pipeline using faithfulness, answer relevance, context precision, and context recall.
- [ ] I can design an LLM-as-Judge prompt with rubric, calibrate it against human judgments, and identify systematic biases.
- [ ] I can configure the NAT Evaluation Framework: load datasets, select evaluators, run evaluations, and interpret results.
- [ ] I can write a custom NAT evaluator for a domain-specific quality criterion.
- [ ] I can use the NAT Profiler to identify token and latency bottlenecks and propose specific optimizations.
- [ ] I can configure NAT Red Teaming evaluators and interpret results to improve agent safety.
- [ ] I can make a principled decision about when to finetune vs. fix prompts/tools/retrieval, using the decision framework.
- [ ] I can describe the DPO finetuning process and explain when it is preferable to full supervised finetuning.
- [ ] I can explain the Data Flywheel pattern and identify where each NVIDIA tool fits in the loop.
- [ ] I can use the NAT Sizing Calculator for capacity planning.
