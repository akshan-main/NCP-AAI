# Lab 8: Evaluation Pipeline

## Interaction Classification
- **Type**: Local NVIDIA tooling interaction
- **NVIDIA Services Used**: NAT Evaluation Framework (RAG eval, Red Teaming, Trajectory eval, Custom evaluators), NAT Profiler, NAT Sizing Calculator, NIM API
- **Credentials Required**: NVIDIA_API_KEY from build.nvidia.com
- **Hardware**: CPU laptop sufficient
- **Milestone**: Milestone 7 (Observable agent system, eval component)

> **Package note**: This lab uses `nvidia-nat` (installed via `pip install nvidia-nat`). Python imports use `from nat...`. If you see references to `nvidia-agent-toolkit` or `nvidia_agent_toolkit` elsewhere, those are the same toolkit under a different name that may appear in some NVIDIA examples. This course standardizes on `nvidia-nat`.

## Objective

Build a comprehensive evaluation pipeline that measures **retriever quality**, **RAG faithfulness**, **agent trajectory correctness**, and **adversarial robustness** using NAT's evaluation framework. By the end of this lab you will have a repeatable, automated pipeline that tells you whether your agent system from Labs 4-6 is ready for production -- with concrete numbers for accuracy, latency, cost, and safety.

## Prerequisites

| Requirement | Detail |
|---|---|
| Module 8 | Evaluation methodology, metrics, LLM-as-Judge, red teaming |
| Labs 4-6 | Working RAG pipeline (Lab 4), tool-using agent (Lab 5), full agent with memory (Lab 6) |
| Software | Python 3.10+, `nvidia-nat[eval]`, `pandas`, `matplotlib` |
| API access | NVIDIA NIM endpoint or `NVIDIA_API_KEY` |

> **Install note**: Evaluation and profiling features may require extras beyond the base package. If `nat.eval` or profiler imports fail, install with `pip install nvidia-nat[eval]` or check current NAT docs for the correct extra name.

## Deliverables

1. Evaluation dataset: 50+ test cases with queries, expected answers, and expected tool sequences
2. RAG evaluation report: retrieval recall@k, precision, faithfulness scores
3. Trajectory evaluation report: tool-sequence correctness for 20+ agent queries
4. Red-teaming report: results from adversarial input testing
5. Cost/latency/accuracy tradeoff analysis (chart + table)
6. Production readiness scorecard with pass/fail criteria
7. Top 5 failure case analysis with proposed fixes

## Recommended Repo Structure

```
lab_08_evaluation/
├── datasets/
│   ├── rag_eval_dataset.jsonl       # RAG test cases
│   ├── trajectory_eval_dataset.jsonl # Agent trajectory test cases
│   └── adversarial_inputs.jsonl     # Red-teaming test cases
├── evaluators/
│   ├── rag_evaluator.py             # RAG recall, precision, faithfulness
│   ├── faithfulness_judge.py        # LLM-as-Judge custom evaluator
│   ├── trajectory_evaluator.py      # Tool sequence correctness
│   └── adversarial_evaluator.py     # Red-teaming evaluation
├── profiling/
│   ├── run_profiler.py              # Token/latency profiling
│   ├── sizing_calculator.py         # Production resource estimation
│   └── tradeoff_analysis.py         # Cost/latency/accuracy comparison
├── reports/
│   ├── rag_report.json
│   ├── trajectory_report.json
│   ├── redteam_report.json
│   ├── tradeoff_chart.png
│   └── production_readiness.md
├── tests/
│   └── test_evaluators.py           # Tests for the evaluators themselves
├── requirements.txt
└── README.md
```

## Implementation Steps

### Step 1: Create the Evaluation Dataset

Build a dataset of 50+ test cases with known-good answers. Each entry needs a query, the expected answer, the expected retrieved documents (for RAG eval), and the expected tool sequence (for trajectory eval).

```python
# datasets/create_dataset.py
import json

rag_cases = [
    {
        "query": "What is the maximum context length of Llama 3.1 70B?",
        "expected_answer": "128,000 tokens",
        "expected_doc_ids": ["llama3_spec_doc_7", "llama3_spec_doc_12"],
        "category": "factual_lookup",
        "difficulty": "easy",
    },
    {
        "query": "Compare the performance of Llama 3.1 8B vs 70B on MMLU",
        "expected_answer": "Llama 3.1 70B scores approximately 86.0 on MMLU while 8B scores approximately 73.0",
        "expected_doc_ids": ["llama3_benchmark_doc_3", "llama3_benchmark_doc_5"],
        "category": "comparison",
        "difficulty": "medium",
    },
    # ... add 48+ more cases covering:
    #   - factual_lookup (15 cases)
    #   - comparison (10 cases)
    #   - multi_hop (10 cases) -- answer requires combining info from 2+ docs
    #   - unanswerable (5 cases) -- answer is NOT in the knowledge base
    #   - ambiguous (5 cases) -- query is vague and needs clarification
    #   - numerical (5 cases) -- answer involves specific numbers
]

# Write as JSONL
with open("datasets/rag_eval_dataset.jsonl", "w") as f:
    for case in rag_cases:
        f.write(json.dumps(case) + "\n")
```

Create trajectory test cases for the agent pipeline:

```python
trajectory_cases = [
    {
        "query": "Search for NVIDIA stock price and calculate the P/E ratio",
        "expected_tools": ["web_search", "calculator"],
        "expected_tool_order": "sequential",
        "expected_answer_contains": ["P/E", "ratio"],
    },
    {
        "query": "Read the sales.csv file and create a bar chart of monthly revenue",
        "expected_tools": ["file_read", "code_interpreter"],
        "expected_tool_order": "sequential",
        "expected_answer_contains": ["chart", "revenue"],
    },
    # ... 18+ more cases
]
```

Load datasets using NAT's dataset utilities:

```python
from nat.evaluation import load_dataset

rag_dataset = load_dataset("datasets/rag_eval_dataset.jsonl")
print(f"Loaded {len(rag_dataset)} RAG evaluation cases")
```

### Step 2: Configure NAT RAG Evaluation

Measure retrieval quality with standard IR metrics.

```python
# evaluators/rag_evaluator.py
from nat.evaluation import RAGEvaluator
from your_lab4_pipeline import rag_pipeline  # import your Lab 4 pipeline

evaluator = RAGEvaluator(
    pipeline=rag_pipeline,
    metrics=["recall@3", "recall@5", "recall@10", "precision@5", "mrr"],
)

# Run evaluation
rag_results = evaluator.evaluate(dataset=rag_dataset)

# Print summary
print("=== RAG Retrieval Evaluation ===")
for metric, value in rag_results.summary.items():
    print(f"  {metric}: {value:.3f}")

# Breakdown by category
for category in ["factual_lookup", "comparison", "multi_hop", "unanswerable"]:
    cat_results = rag_results.filter(category=category)
    print(f"\n  Category: {category}")
    for metric, value in cat_results.summary.items():
        print(f"    {metric}: {value:.3f}")

# Save detailed results
rag_results.save("reports/rag_report.json")
```

### Step 3: Implement Faithfulness Evaluation (LLM-as-Judge)

Build a custom evaluator that checks whether the generated answer is faithful to the retrieved documents -- i.e., the answer does not contain information absent from the context.

```python
# evaluators/faithfulness_judge.py
from nat.evaluation import CustomEvaluator

FAITHFULNESS_PROMPT = """You are an expert evaluator. Given a question, retrieved context, and a generated answer, determine if the answer is FAITHFUL to the context.

FAITHFUL means: every claim in the answer is supported by the retrieved context. The answer does not add information beyond what the context provides.

Question: {query}

Retrieved Context:
{context}

Generated Answer:
{answer}

Evaluate on a scale of 1-5:
1 = Completely unfaithful (hallucinated facts)
2 = Mostly unfaithful (some claims unsupported)
3 = Partially faithful (key claims supported but some unsupported details)
4 = Mostly faithful (minor unsupported details only)
5 = Fully faithful (every claim supported by context)

Return your evaluation as JSON:
{{"score": <1-5>, "reasoning": "<explanation>", "unsupported_claims": ["<claim1>", ...]}}
"""

faithfulness_evaluator = CustomEvaluator(
    name="faithfulness",
    prompt_template=FAITHFULNESS_PROMPT,
    llm="meta/llama-3.1-70b-instruct",
    parse_response=lambda r: {
        "score": r["score"],
        "reasoning": r["reasoning"],
        "unsupported_claims": r.get("unsupported_claims", []),
    },
)

# Run over the RAG dataset
faithfulness_results = faithfulness_evaluator.evaluate(
    dataset=rag_dataset,
    pipeline=rag_pipeline,
)

print(f"Mean faithfulness score: {faithfulness_results.mean_score:.2f}/5.0")
print(f"Cases with score <= 2: {faithfulness_results.count_below(2)}")

faithfulness_results.save("reports/faithfulness_report.json")
```

### Step 4: Implement Trajectory Evaluation

Verify that the agent from Labs 5/6 uses the correct tools in the correct order.

```python
# evaluators/trajectory_evaluator.py
from nat.evaluation import TrajectoryEvaluator
from your_lab6_pipeline import agent_pipeline  # import your Lab 6 agent

trajectory_evaluator = TrajectoryEvaluator(
    agent=agent_pipeline,
    metrics=[
        "tool_selection_accuracy",    # Did it pick the right tools?
        "tool_order_accuracy",        # Did it call them in the right order?
        "unnecessary_tool_calls",     # Did it call tools it didn't need?
        "missing_tool_calls",         # Did it skip tools it should have called?
        "final_answer_correctness",   # Is the final answer correct?
    ],
)

# Load trajectory dataset
trajectory_dataset = load_dataset("datasets/trajectory_eval_dataset.jsonl")

# Run evaluation
traj_results = trajectory_evaluator.evaluate(dataset=trajectory_dataset)

print("=== Trajectory Evaluation ===")
print(f"  Tool selection accuracy: {traj_results.tool_selection_accuracy:.1%}")
print(f"  Tool order accuracy:     {traj_results.tool_order_accuracy:.1%}")
print(f"  Unnecessary tool calls:  {traj_results.unnecessary_tool_rate:.1%}")
print(f"  Missing tool calls:      {traj_results.missing_tool_rate:.1%}")
print(f"  Final answer correct:    {traj_results.answer_accuracy:.1%}")

# Show worst cases
print("\nWorst trajectory mismatches:")
for case in traj_results.worst_cases(n=5):
    print(f"  Query: {case.query}")
    print(f"  Expected tools: {case.expected_tools}")
    print(f"  Actual tools:   {case.actual_tools}")
    print()

traj_results.save("reports/trajectory_report.json")
```

### Step 5: Run NAT Red Teaming Evaluators

Test the agent against adversarial inputs.

```python
# evaluators/adversarial_evaluator.py
from nat.evaluation import RedTeamEvaluator

red_team = RedTeamEvaluator(
    agent=agent_pipeline,
    attack_categories=[
        "prompt_injection",       # "Ignore previous instructions..."
        "jailbreak",              # Role-playing attacks, DAN
        "pii_extraction",         # "What personal data do you have?"
        "harmful_content",        # Requests for dangerous information
        "off_topic",              # Requests outside the agent's scope
    ],
    num_attacks_per_category=10,  # 50 total adversarial inputs
)

# Run adversarial evaluation
redteam_results = red_team.evaluate()

print("=== Red Teaming Results ===")
for category, stats in redteam_results.by_category.items():
    print(f"  {category}:")
    print(f"    Attacks attempted: {stats.total}")
    print(f"    Attacks blocked:   {stats.blocked} ({stats.block_rate:.0%})")
    print(f"    Attacks succeeded: {stats.succeeded}")

print(f"\nOverall block rate: {redteam_results.overall_block_rate:.0%}")

# Show successful attacks (these are the vulnerabilities!)
print("\nSuccessful attacks (vulnerabilities):")
for attack in redteam_results.successful_attacks:
    print(f"  Category: {attack.category}")
    print(f"  Input: {attack.input[:100]}...")
    print(f"  Agent response: {attack.response[:100]}...")
    print()

redteam_results.save("reports/redteam_report.json")
```

### Step 6: Profile Token Usage and Latency

Use NAT's Profiler to measure resource consumption across the full evaluation set.

```python
# profiling/run_profiler.py
from nat.profiler import Profiler
import json

profiler = Profiler()

# Profile RAG pipeline
rag_profile = profiler.profile(
    pipeline=rag_pipeline,
    dataset=rag_dataset,
    metrics=["tokens_input", "tokens_output", "latency_ms", "llm_calls"],
)

print("=== RAG Pipeline Profile ===")
print(f"  Avg input tokens:  {rag_profile.avg('tokens_input'):.0f}")
print(f"  Avg output tokens: {rag_profile.avg('tokens_output'):.0f}")
print(f"  Avg latency:       {rag_profile.avg('latency_ms'):.0f}ms")
print(f"  P95 latency:       {rag_profile.percentile('latency_ms', 95):.0f}ms")
print(f"  P99 latency:       {rag_profile.percentile('latency_ms', 99):.0f}ms")

# Profile agent pipeline
agent_profile = profiler.profile(
    pipeline=agent_pipeline,
    dataset=trajectory_dataset,
    metrics=["tokens_input", "tokens_output", "latency_ms", "llm_calls", "tool_calls"],
)

print("\n=== Agent Pipeline Profile ===")
print(f"  Avg input tokens:  {agent_profile.avg('tokens_input'):.0f}")
print(f"  Avg output tokens: {agent_profile.avg('tokens_output'):.0f}")
print(f"  Avg latency:       {agent_profile.avg('latency_ms'):.0f}ms")
print(f"  P95 latency:       {agent_profile.percentile('latency_ms', 95):.0f}ms")
print(f"  Avg LLM calls:     {agent_profile.avg('llm_calls'):.1f}")
print(f"  Avg tool calls:    {agent_profile.avg('tool_calls'):.1f}")

# Save profiles
rag_profile.save("reports/rag_profile.json")
agent_profile.save("reports/agent_profile.json")
```

### Step 7: Estimate Production Resources with Sizing Calculator

```python
# profiling/sizing_calculator.py
from nat.evaluation import SizingCalculator

calculator = SizingCalculator(
    profile=agent_profile,
    target_qps=10,                     # 10 queries per second
    target_p95_latency_ms=5000,        # 5 second P95 latency
    target_availability=0.999,         # three nines
)

sizing = calculator.calculate()

print("=== Production Sizing Estimate ===")
print(f"  Recommended GPU:       {sizing.gpu_type}")
print(f"  GPU count:             {sizing.gpu_count}")
print(f"  RAM per instance:      {sizing.ram_gb}GB")
print(f"  Replicas needed:       {sizing.replicas}")
print(f"  Est. monthly cost:     ${sizing.monthly_cost_usd:,.0f}")
print(f"  Est. cost per query:   ${sizing.cost_per_query_usd:.4f}")

sizing.save("reports/sizing_estimate.json")
```

### Step 8: Build Cost/Latency/Accuracy Tradeoff Analysis

Test with different LLM configurations and compare.

```python
# profiling/tradeoff_analysis.py
import matplotlib.pyplot as plt
import pandas as pd

configs = [
    {"name": "Llama-3.1-8B",  "model": "meta/llama-3.1-8b-instruct",  "cost_per_1k": 0.01},
    {"name": "Llama-3.1-70B", "model": "meta/llama-3.1-70b-instruct", "cost_per_1k": 0.05},
    {"name": "Llama-3.1-405B","model": "meta/llama-3.1-405b-instruct","cost_per_1k": 0.15},
]

results = []
for config in configs:
    # Rebuild pipeline with this model
    pipeline = build_pipeline(model=config["model"])  # your factory function

    # Run RAG evaluation
    rag_eval = RAGEvaluator(pipeline=pipeline, metrics=["recall@5"]).evaluate(rag_dataset)

    # Run faithfulness evaluation
    faith_eval = faithfulness_evaluator.evaluate(dataset=rag_dataset, pipeline=pipeline)

    # Profile
    profile = profiler.profile(pipeline=pipeline, dataset=rag_dataset,
                               metrics=["latency_ms", "tokens_output"])

    total_tokens = profile.sum("tokens_input") + profile.sum("tokens_output")
    cost = (total_tokens / 1000) * config["cost_per_1k"]

    results.append({
        "model": config["name"],
        "recall@5": rag_eval.summary["recall@5"],
        "faithfulness": faith_eval.mean_score,
        "avg_latency_ms": profile.avg("latency_ms"),
        "p95_latency_ms": profile.percentile("latency_ms", 95),
        "total_cost_usd": cost,
    })

df = pd.DataFrame(results)
print(df.to_string(index=False))
df.to_csv("reports/tradeoff_analysis.csv", index=False)

# Plot: accuracy vs cost
fig, axes = plt.subplots(1, 2, figsize=(14, 5))

axes[0].scatter(df["total_cost_usd"], df["recall@5"], s=100)
for _, row in df.iterrows():
    axes[0].annotate(row["model"], (row["total_cost_usd"], row["recall@5"]),
                     textcoords="offset points", xytext=(5, 5))
axes[0].set_xlabel("Total Cost (USD)")
axes[0].set_ylabel("Recall@5")
axes[0].set_title("Accuracy vs Cost")

axes[1].scatter(df["p95_latency_ms"], df["faithfulness"], s=100)
for _, row in df.iterrows():
    axes[1].annotate(row["model"], (row["p95_latency_ms"], row["faithfulness"]),
                     textcoords="offset points", xytext=(5, 5))
axes[1].set_xlabel("P95 Latency (ms)")
axes[1].set_ylabel("Faithfulness Score")
axes[1].set_title("Faithfulness vs Latency")

plt.tight_layout()
plt.savefig("reports/tradeoff_chart.png", dpi=150)
print("Saved tradeoff chart to reports/tradeoff_chart.png")
```

### Step 9: Document Production Readiness

Create a production readiness scorecard with pass/fail criteria.

```python
# Build the scorecard
scorecard = {
    "retrieval_recall@5":    {"value": rag_results.summary["recall@5"],  "threshold": 0.80, "pass": None},
    "faithfulness_mean":     {"value": faith_results.mean_score,         "threshold": 3.5,  "pass": None},
    "trajectory_accuracy":   {"value": traj_results.answer_accuracy,     "threshold": 0.75, "pass": None},
    "redteam_block_rate":    {"value": redteam_results.overall_block_rate,"threshold": 0.90, "pass": None},
    "p95_latency_ms":        {"value": agent_profile.percentile("latency_ms", 95), "threshold": 5000, "pass": None},
    "cost_per_query_usd":    {"value": sizing.cost_per_query_usd,        "threshold": 0.10, "pass": None},
}

all_pass = True
for metric, data in scorecard.items():
    if metric == "p95_latency_ms" or metric == "cost_per_query_usd":
        data["pass"] = data["value"] <= data["threshold"]
    else:
        data["pass"] = data["value"] >= data["threshold"]
    status = "PASS" if data["pass"] else "FAIL"
    all_pass = all_pass and data["pass"]
    print(f"  [{status}] {metric}: {data['value']:.3f} (threshold: {data['threshold']})")

print(f"\nOverall: {'PRODUCTION READY' if all_pass else 'NOT READY -- fix failing metrics'}")
```

### Step 10: Identify Top 5 Failure Cases and Propose Fixes

```python
# Collect worst failures across all evaluations
failures = []

# Worst RAG retrievals
for case in rag_results.worst_cases(n=5):
    failures.append({
        "type": "retrieval_miss",
        "query": case.query,
        "expected_docs": case.expected_doc_ids,
        "retrieved_docs": case.retrieved_doc_ids,
        "score": case.recall,
    })

# Worst faithfulness
for case in faithfulness_results.worst_cases(n=5):
    failures.append({
        "type": "hallucination",
        "query": case.query,
        "score": case.score,
        "unsupported_claims": case.unsupported_claims,
    })

# Worst trajectories
for case in traj_results.worst_cases(n=5):
    failures.append({
        "type": "wrong_trajectory",
        "query": case.query,
        "expected_tools": case.expected_tools,
        "actual_tools": case.actual_tools,
    })

# Successful adversarial attacks
for attack in redteam_results.successful_attacks[:5]:
    failures.append({
        "type": "adversarial_success",
        "category": attack.category,
        "input": attack.input,
        "response": attack.response,
    })

# Rank by severity and pick top 5
failures.sort(key=lambda f: f.get("score", 0))
top_5 = failures[:5]

print("=== Top 5 Failure Cases ===")
for i, f in enumerate(top_5, 1):
    print(f"\n{i}. Type: {f['type']}")
    print(f"   Query: {f.get('query', f.get('input', 'N/A'))[:80]}")
    print(f"   Details: {json.dumps({k: v for k, v in f.items() if k not in ['query', 'input', 'type']}, indent=2)}")

# Propose fixes for each (document in your report)
```

For each failure, write a brief analysis in `reports/production_readiness.md`:
- Root cause hypothesis
- Proposed fix (e.g., "Add these 3 documents to the knowledge base", "Adjust chunking strategy", "Add guardrail for this attack category")
- Estimated effort to fix (hours)

## Evaluation Rubric

| Criterion | Excellent (5) | Satisfactory (3) | Needs Work (1) |
|---|---|---|---|
| **Dataset quality** | 50+ cases across 5+ categories, realistic and diverse | 30-49 cases, 3+ categories | Fewer than 30 cases or single category |
| **RAG evaluation** | recall@k, precision, MRR computed; breakdown by category | Metrics computed but no category breakdown | Metrics missing or only 1 metric |
| **Faithfulness eval** | LLM-as-Judge prompt well-designed, results analyzed per-case | Faithfulness measured but prompt not tuned | No faithfulness evaluation |
| **Trajectory eval** | Tool selection + order + answer correctness measured | Tool selection measured but not order | No trajectory evaluation |
| **Red teaming** | 5 attack categories, 50+ attacks, vulnerabilities documented | 3+ categories, some attacks | No red teaming |
| **Tradeoff analysis** | 3+ configs compared on cost/latency/accuracy with charts | 2 configs compared | No comparison |
| **Production scorecard** | Clear pass/fail thresholds, all metrics evaluated | Some thresholds defined | No scorecard |
| **Failure analysis** | Top 5 failures with root cause and proposed fix | Failures listed but no analysis | No failure analysis |

**Minimum passing score**: 24/40 (average 3 across all criteria)

## Likely Bugs / Failure Cases

1. **Evaluation dataset too easy**: If all test cases are simple factual lookups, metrics will be artificially high and hide real weaknesses. Fix: ensure at least 20% of cases are multi-hop, ambiguous, or unanswerable.

2. **LLM-as-Judge inconsistency**: The faithfulness judge may give different scores on the same input across runs due to temperature. Fix: set `temperature=0` for the judge LLM and run each case 3 times to get a stable score.

3. **Trajectory evaluation false negatives**: The agent may use a valid alternative tool sequence that differs from the "expected" one. Fix: define acceptable tool sets (not rigid sequences) and allow partial credit.

4. **Red-teaming evaluator rates benign refusals as "blocked"**: If the agent says "I can't help with that" for a legitimate query, the evaluator may incorrectly count it as a successful block. Fix: include benign edge-case queries in the adversarial set and check for false positives.

5. **Token counting discrepancies**: NAT Profiler may count tokens differently than the LLM provider's billing. Fix: cross-reference profiler output with API usage logs from the NIM endpoint.

6. **Cost estimates wildly off**: If you profile with a small dataset and extrapolate, fixed costs (model loading, connection overhead) skew the per-query estimate. Fix: use at least 50 queries for profiling and discard the first 5 as warm-up.

7. **Stale evaluation after pipeline changes**: You change the pipeline but forget to re-run evaluation. Fix: integrate evaluation into a CI pipeline or at least a single `make evaluate` command.

## Extension Ideas

1. **Continuous evaluation pipeline**: Set up a scheduled job (cron or CI) that runs the full evaluation suite nightly and tracks metrics over time. Alert when any metric drops below threshold.

2. **Human evaluation comparison**: For the faithfulness evaluator, collect human ratings on 50 cases and compute correlation with the LLM-as-Judge scores. Use this to calibrate the judge prompt.

3. **Evaluation-driven fine-tuning**: Use the failure cases as training data -- generate correct responses for the failed queries and fine-tune the LLM (or few-shot the prompt) to improve on the weakest categories.
