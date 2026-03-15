# Module 11: Observability, Monitoring, and Maintenance

## A. Module Title

**Observability, Monitoring, and Maintenance of Agentic AI Systems**

Primary Exam Domain: Run, Monitor, and Maintain (5%)

---

## B. Why This Module Matters in Real Systems

A deployed agent that you cannot observe is a deployed liability. Traditional API monitoring — request count, error rate, latency — captures perhaps 20% of what matters for an agent system. The other 80% is invisible without purpose-built observability: which tool did the agent call and why? How many tokens did step 3 consume? Did the guardrail fire correctly or was it a false positive? Why did the agent loop 7 times before answering? Without answers to these questions, you are operating blind, and your production agent is one subtle prompt regression away from an outage nobody can diagnose.

Agent observability is harder than API observability for structural reasons. An API call is a single span — request in, response out. An agent execution is a tree of spans: the workflow span contains agent spans, which contain LLM call spans and tool call spans, which may themselves contain sub-agent spans. Each span has different metrics that matter (tokens for LLM calls, latency for tool calls, pass/fail for guardrails). The trace structure is not fixed — it varies per request based on the agent's decisions. This dynamic structure requires OpenTelemetry-based distributed tracing, not simple log aggregation.

NVIDIA provides specific observability tooling through NAT: integrations with Phoenix, Weave, and Langfuse for trace visualization; OpenTelemetry as the underlying protocol; redaction processors for sensitive data; and custom telemetry exporters for bespoke systems. NeMo Guardrails adds its own structured logging and trace data. This module teaches you to instrument, monitor, debug, and maintain production agent systems using these tools.

---

## C. Learning Objectives

1. Explain why agent observability requires distributed tracing rather than simple log aggregation
2. Configure NAT observability integrations (Phoenix, Weave, Langfuse) and select the right tool for a given requirement
3. Read and interpret a multi-level execution trace (workflow → agent → tool → LLM)
4. Implement token counting and cost tracking across agent executions
5. Configure NAT redaction processors to remove sensitive data from telemetry
6. Set up monitoring dashboards with appropriate metrics and alerting thresholds
7. Debug agent failures using traces: identify LLM timeouts, tool errors, infinite loops, context overflow, and guardrail false positives
8. Design a maintenance strategy for model updates, configuration changes, and RAG index freshness

---

## D. Required Concepts

- OpenTelemetry fundamentals (traces, spans, metrics, exporters)
- Distributed tracing concepts (trace context propagation, span hierarchies)
- NAT agent architecture (Modules 3-5)
- NeMo Guardrails (Module 8)
- Deployment architecture (Module 10)
- Basic understanding of Prometheus/Grafana or equivalent monitoring stacks

---

## E. Core Lesson Content

### [BEGINNER] Why Agent Observability Is Different

Consider a standard REST API: request comes in, handler executes, response goes out. One log line captures the essentials. Now consider an agent execution:

1. User sends a question
2. Agent plans 3 steps
3. Step 1: calls a search tool (200ms, returns 5 results)
4. Step 2: calls LLM to synthesize (1200ms, 3400 input tokens, 800 output tokens)
5. Step 3: calls a database tool (fails, retries, succeeds on retry) (1800ms total)
6. Agent re-plans because step 3 results changed the approach
7. Step 4: calls LLM again (900ms, 4200 input tokens, 400 output tokens)
8. Guardrail checks output (50ms, passes)
9. Final response returned

Total: 4150ms, 8800 tokens, 1 tool failure with retry, 1 re-plan. A single log line saying "200 OK, 4150ms" captures almost nothing useful. You need a trace with nested spans showing the full execution tree.

### [BEGINNER] NAT Observability Integrations

NAT provides built-in integrations with three observability platforms:

**Phoenix (Arize)** — Open-source, self-hosted. Best for: teams that need full control over trace data, cost-sensitive deployments, integration with Arize's evaluation tools. Provides trace visualization, LLM call inspection, and evaluation dataset management.

**Weave (Weights & Biases)** — Cloud-hosted, strong experiment tracking. Best for: teams already using W&B for ML experiments, those who want trace data alongside training metrics. Provides trace visualization integrated with W&B's experiment dashboard.

**Langfuse** — Open-source with managed cloud option. Best for: teams focused on LLM-specific observability, prompt management, cost tracking. Provides trace visualization, prompt versioning, and detailed cost analytics.

Selection guidance:
| Criterion | Phoenix | Weave | Langfuse |
|-----------|---------|-------|----------|
| Self-hosted | Yes | No (cloud) | Yes + cloud |
| Cost tracking | Basic | Basic | Detailed |
| Prompt management | No | No | Yes |
| ML experiment integration | Arize | W&B | No |
| Open source | Yes | No | Yes |

### [INTERMEDIATE] OpenTelemetry: The Underlying Protocol

NAT uses OpenTelemetry (OTel) as its telemetry protocol. All traces, regardless of which visualization platform you use, flow through OTel.

```
Observability Pipeline
=========================================

  NAT Agent Execution
       |
       | (instrumented spans)
       v
  OpenTelemetry SDK
       |
       | (OTLP export)
       v
  OTel Collector (optional)
       |
       +-----+------+------+
       |      |      |      |
       v      v      v      v
   Phoenix  Weave  Langfuse  Custom
                              Exporter
       |      |      |      |
       v      v      v      v
   Dashboards / Alerts / Analysis
```

Key OTel concepts for agent tracing:

- **Trace**: The entire agent execution from request to response. One trace per user request.
- **Span**: A single operation within the trace. Examples: "LLM call", "tool: search_database", "guardrail: output_check".
- **Span attributes**: Key-value metadata on each span. For LLM spans: model name, token counts, temperature. For tool spans: tool name, input parameters, success/failure.
- **Span hierarchy**: Spans nest. A workflow span contains agent spans, which contain tool and LLM spans.

### [INTERMEDIATE] Execution Tracing: Reading a Trace

A NAT execution trace has a consistent hierarchy:

```
Trace: user_request_abc123
├── Span: workflow_execution (total: 4150ms)
│   ├── Span: agent_step_1 (tool_call: search_db, 200ms)
│   │   ├── Attribute: tool.name = "search_db"
│   │   ├── Attribute: tool.input = {"query": "..."}
│   │   └── Attribute: tool.output.count = 5
│   ├── Span: agent_step_2 (llm_call, 1200ms)
│   │   ├── Attribute: llm.model = "meta/llama-3.1-70b"
│   │   ├── Attribute: llm.tokens.input = 3400
│   │   ├── Attribute: llm.tokens.output = 800
│   │   └── Attribute: llm.latency_first_token = 180ms
│   ├── Span: agent_step_3 (tool_call: query_api, 1800ms)
│   │   ├── Attribute: tool.name = "query_api"
│   │   ├── Attribute: tool.retry_count = 1
│   │   └── Status: OK (after retry)
│   ├── Span: agent_replan (50ms)
│   ├── Span: agent_step_4 (llm_call, 900ms)
│   │   ├── Attribute: llm.tokens.input = 4200
│   │   └── Attribute: llm.tokens.output = 400
│   └── Span: guardrail_check (output_rail, 50ms)
│       ├── Attribute: guardrail.name = "output_safety"
│       └── Attribute: guardrail.result = "pass"
```

How to read this trace for debugging:
- **Latency bottleneck**: Step 3 took 1800ms (43% of total) due to a retry. Investigate the external API reliability.
- **Cost driver**: Step 2 used the most tokens (3400 input). Check if the context could be trimmed.
- **Re-plan event**: The agent changed its plan mid-execution. This is expected behavior but worth monitoring for frequency.

### [INTERMEDIATE] Token Counting and Cost Tracking

Every LLM call in an agent execution consumes tokens. For cost management, you need per-step token tracking:

- Input tokens: the prompt, including conversation history, RAG context, and tool results
- Output tokens: the LLM's response (more expensive per token on most providers)
- Total tokens per execution: sum across all LLM calls in the trace
- Cost per execution: tokens x price-per-token for the model used

NAT traces include token counts as span attributes. Aggregate these in your observability platform to track:
- Average cost per request
- Cost distribution (p50, p95, p99 — some requests may cost 10x the average)
- Cost trends over time (are prompts getting longer? Is the agent making more LLM calls?)

### [INTERMEDIATE] NAT Redaction Processors

Agent traces often contain sensitive data: user queries may include personal information, tool outputs may include database records, LLM responses may echo back sensitive inputs. NAT provides redaction processors that strip sensitive data from telemetry before export.

Configuration:
- Define regex patterns for PII (email, SSN, phone number, credit card)
- Define field-level redaction (always redact `tool.input.password`, etc.)
- Apply redaction at the OTel exporter level so all downstream systems receive clean data
- Redaction is irreversible — design patterns carefully to avoid losing debugging-critical data

### [INTERMEDIATE] NeMo Guardrails Observability

NeMo Guardrails generates its own traces that show:
- Input rail execution (which rails fired, pass/fail, latency)
- Dialog rail execution (conversation flow decisions)
- Output rail execution (content filtering results)

These traces use OpenTelemetry and can be correlated with NAT workflow traces using trace context propagation. When a guardrail blocks a response, the NAT trace shows the guardrail span with a "blocked" status, and the guardrails trace shows which specific rail triggered and why.

Structured logging in NeMo Guardrails captures:
- Rail evaluation decisions at configurable log levels
- Input/output pairs for each rail (subject to redaction)
- Performance timing for each rail in the chain

### [ADVANCED] Monitoring Dashboards

**Note**: Dashboard design is operational best practice, not NVIDIA-prescribed. The following recommendations are inferred from production agent monitoring experience.

Key metrics for an agent monitoring dashboard:

**Reliability metrics**:
- Success rate: % of requests that complete without error
- Error rate by type: LLM timeout, tool failure, guardrail block, context overflow
- Retry rate: how often tools or LLM calls are retried

**Latency metrics**:
- End-to-end latency: p50, p95, p99
- Time-to-first-token: how quickly the user sees a streaming response begin
- Per-step latency breakdown: LLM time vs. tool time vs. overhead

**Cost metrics**:
- Token cost per request (average and p99)
- Daily/weekly token spend by model
- Cost per user or per use case

**Quality metrics**:
- Guardrail trigger rate (if rising, the model may be degrading)
- Re-plan rate (if rising, the agent is struggling with tasks)
- Average steps per execution (if rising, efficiency may be declining)

Alerting recommendations:
- Error rate > 5%: page on-call
- p99 latency > 30s: warn
- Guardrail block rate > 10%: investigate (either bad inputs or false positives)
- Token spend exceeds daily budget: alert finance + throttle traffic

### [ADVANCED] Debugging Agent Failures

Common failure patterns and how traces reveal them:

1. **LLM timeout**: Trace shows an LLM span with duration > timeout threshold and an error status. Cause: model overloaded, input too long, or network issue. Fix: increase timeout, reduce context, or add NIM replicas.

2. **Tool failure**: Trace shows a tool span with error status. Check `tool.error` attribute for details. Common: API rate limits, authentication expiry, invalid input format.

3. **Infinite loop**: Trace shows repeated identical tool calls or LLM calls. The agent is stuck in a plan-execute loop. Fix: implement maximum step limits in NAT configuration, improve agent instructions.

4. **Context overflow**: Trace shows an LLM span failure with a token limit error. The accumulated context (history + RAG + tool results) exceeds the model's context window. Fix: implement context summarization or sliding window.

5. **Guardrail false positive**: Trace shows a guardrail span blocking a legitimate response. Check the guardrail trace for which specific rail triggered. Fix: tune the rail configuration, add exceptions, or adjust thresholds.

NAT-specific debugging tools:
- `nat_test_llm`: Replace the real LLM with a deterministic mock for replay testing. Reproduce exact failure conditions without LLM cost or non-determinism.
- NAT profiler: Identify performance bottlenecks at the framework level (serialization overhead, memory allocation, event loop contention).

### [ADVANCED] Continuous Improvement and Maintenance

**Data Flywheel**: Collect production traces → identify failure patterns → create evaluation datasets from real failures → evaluate prompt/config changes → deploy improvements → repeat. NAT's evaluation framework (Module 9) connects directly to this loop — use production data as eval inputs.

**Model updates without downtime**: Use blue-green deployment (Module 10). Deploy new NIM container with updated model alongside old one. Route traffic gradually. Monitor quality metrics during transition. Roll back if quality degrades.

**Configuration hot-reloading**: NeMo Guardrails configs can be updated without restarting the agent. Use ConfigMap updates in Kubernetes with volume mounts that detect file changes.

**RAG index freshness**: Stale indexes cause agents to return outdated information. Implement scheduled re-indexing. Monitor index staleness as a metric. Alert when the newest document in the index is older than a threshold.

**When to tune prompts vs. retrain vs. change architecture**:
- Prompt tuning: first response to quality issues. Fast, cheap, reversible.
- Configuration changes (guardrails, tool parameters): when the issue is behavioral boundaries, not LLM quality.
- Model swap: when the current model fundamentally cannot handle the task (e.g., reasoning capability gap).
- Architecture change: last resort. When the agent type, memory strategy, or tool design is wrong.

---

## Hands-on NVIDIA Platform Interaction

**Title: Instrument a NAT Workflow with Tracing**

**Requirements:**
- `NVIDIA_API_KEY`
- `nvidia-nat` installed locally
- Phoenix (`pip install arize-phoenix`) or Langfuse account
- Execution: local NAT + local Phoenix + hosted NIM endpoint

**Exercise:**

1. Install Phoenix: `pip install arize-phoenix` and start the Phoenix server: `python -m phoenix.server.main serve`
2. Configure your NAT workflow (from Module 6 / Lab 6) to export telemetry to Phoenix.
3. Run 5 queries through the workflow.
4. Open Phoenix UI (localhost:6006). Find the traces for your 5 queries.
5. For one trace: identify every span (workflow → agent → LLM call → tool call). Note the token count and latency at each level.
6. Configure NAT redaction processor to mask any PII in telemetry. Run another query with PII in the input. Verify redaction in Phoenix.
7. Simulate a failure: modify a tool to raise an exception. Run a query that triggers it. Find the error in Phoenix traces.

**Deliverable:**
- (a) Screenshot of Phoenix showing a full trace with spans.
- (b) Annotated trace showing token/latency breakdown.
- (c) Screenshot showing PII redaction in action.
- (d) Screenshot showing error trace from simulated failure.

**Verification:** Traces appear in Phoenix with correct span hierarchy. PII is redacted. Error traces show the failure point.

**Classification:** Local NVIDIA tooling interaction (NAT observability, Phoenix) + hosted NVIDIA API interaction (NIM endpoint)

---

## F. Terminology Box

| Term | Definition |
|------|-----------|
| **OpenTelemetry (OTel)** | Open standard for distributed tracing, metrics, and logs. The protocol NAT uses for telemetry export. |
| **Trace** | A complete record of a single agent execution, from request to response, composed of nested spans. |
| **Span** | A single operation within a trace (e.g., one LLM call, one tool invocation, one guardrail check). |
| **Phoenix** | Open-source observability platform (by Arize) integrated with NAT for trace visualization. |
| **Weave** | Weights & Biases observability product integrated with NAT for trace visualization. |
| **Langfuse** | Open-source LLM observability platform integrated with NAT for tracing and cost analytics. |
| **Redaction Processor** | NAT component that strips sensitive data from telemetry before export. |
| **nat_test_llm** | NAT testing utility that replaces the real LLM with a deterministic mock for replay debugging. |
| **Time-to-First-Token (TTFT)** | Latency from request submission to when the first token of the response is generated. |
| **Data Flywheel** | Cycle of collecting production data, evaluating quality, making improvements, and redeploying. |
| **Context Overflow** | Failure when accumulated context exceeds the LLM's maximum token window. |
| **Hot-Reload** | Updating configuration without restarting the service. |

---

## G. Common Misconceptions

1. **"Logging is sufficient for agent observability."** Logs capture events but not relationships. A log line saying "tool call failed" doesn't tell you which agent step triggered it, what the agent did next, or how it affected the final output. You need distributed tracing with span hierarchies.

2. **"All agent requests cost about the same in tokens."** Token cost per request varies by 10-100x depending on the task complexity, number of tool calls, context size, and re-planning events. You must track cost distributions, not just averages.

3. **"Guardrail blocks are always correct."** High guardrail block rates often indicate false positives rather than genuine safety catches. Always investigate guardrail trigger patterns and review blocked outputs to tune rail configurations.

4. **"You only need to monitor the agent, not the models."** The LLM inference layer (NIM) has its own failure modes: GPU OOM, model loading failures, inference queue saturation. Monitor both the orchestration layer (NAT) and the inference layer (NIM) independently.

5. **"Once deployed, agent prompts don't drift."** Even with fixed prompts, agent behavior drifts because: (a) the data environment changes (RAG indexes get stale), (b) external tools change their API responses, (c) user behavior evolves. Continuous monitoring is not optional.

6. **"Redaction removes the need for access controls on trace data."** Redaction is a defense-in-depth measure, not a substitute for access controls. Redaction patterns can miss novel PII formats. Always combine redaction with role-based access to observability platforms.

---

## H. Failure Modes / Anti-Patterns

1. **Observability as afterthought**: Adding tracing after production issues arise means you cannot debug the first incident. Instrument from day one. NAT provides built-in instrumentation — enable it during development, not after launch.

2. **Trace data without retention policy**: Agent traces are large (kilobytes per trace, millions of traces per month). Without retention policies, storage costs grow unbounded. Define retention: full traces for 7 days, sampled traces for 90 days, aggregated metrics indefinitely.

3. **Alerting on averages instead of percentiles**: Average latency of 2 seconds masks a p99 of 45 seconds. Always alert on percentiles. The worst 1% of user experiences define your reputation.

4. **No correlation between NAT and guardrails traces**: If NAT traces and NeMo Guardrails traces use different trace contexts, you cannot tell which guardrail check belongs to which agent execution. Ensure trace context propagation is configured end-to-end.

5. **Redacting too aggressively**: Redacting all user input from traces makes debugging impossible. Find the balance: redact PII patterns (emails, SSNs) but preserve the semantic content needed for debugging (query topics, intent categories).

6. **Ignoring the data flywheel**: Running production agents without feeding failure data back into evaluation and improvement means you learn nothing from every failure. Implement automated collection of failed traces into eval datasets.

---

## I. Hands-On Lab

**Lab 11: Instrument and Monitor a NAT Agent**

Configure NAT observability with Phoenix (self-hosted). Deploy a multi-tool agent that performs search and summarization. Generate 50 requests with varying complexity. Read traces in Phoenix to identify: (a) the most expensive request by token count, (b) a request with a tool failure, (c) the average number of LLM calls per request. Configure a redaction processor to remove email addresses from traces. Verify redaction works by inspecting traces.

---

## J. Stretch Lab

**Stretch Lab 11: End-to-End Monitoring Pipeline with Alerting**

Extend Lab 11 with Prometheus metrics export and Grafana dashboards. Build dashboards for: success rate, p50/p95/p99 latency, token cost per request, guardrail trigger rate. Configure alerting rules: page on error rate > 5%, warn on p99 > 20s. Simulate a failure (kill NIM container) and verify alerts fire within 60 seconds. Implement the data flywheel: automatically export failed traces to an evaluation dataset file.

---

## K. Review Quiz

**Q1.** NAT uses which protocol for telemetry export?
- A) StatsD
- B) Prometheus native
- C) OpenTelemetry
- D) Custom binary protocol

**Answer: C** — NAT uses OpenTelemetry (OTel) as its telemetry protocol.

**Q2.** Which NAT observability integration provides detailed cost analytics and prompt versioning?
- A) Phoenix
- B) Weave
- C) Langfuse
- D) Grafana

**Answer: C** — Langfuse focuses on LLM-specific observability with detailed cost tracking and prompt management.

**Q3.** An agent trace shows 12 consecutive identical tool calls. What failure pattern is this?
- A) Context overflow
- B) LLM timeout
- C) Infinite loop
- D) Guardrail false positive

**Answer: C** — Repeated identical operations indicate the agent is stuck in a loop. Implement step limits.

**Q4.** What does `nat_test_llm` provide?
- A) A faster LLM for testing
- B) A deterministic mock LLM for replay debugging
- C) A benchmark suite for LLM performance
- D) A tool for testing LLM fine-tuning

**Answer: B** — `nat_test_llm` replaces the real LLM with a deterministic mock, enabling exact reproduction of failure conditions.

**Q5.** Why is average latency a poor alerting metric for agent systems?
- A) It's too difficult to compute
- B) It masks extreme outliers; p99 latency may be 20x the average
- C) Agent latency can't be measured
- D) Average latency is always too high

**Answer: B** — Variable-length agent executions create heavy-tailed latency distributions where the average hides the worst cases.

**Q6.** What is the correct order of the data flywheel?
- A) Deploy → Collect → Evaluate → Retrain
- B) Collect → Evaluate → Improve → Deploy → Collect
- C) Retrain → Deploy → Collect → Evaluate
- D) Evaluate → Retrain → Deploy → Collect

**Answer: B** — The flywheel is a continuous loop: collect production data → evaluate quality → improve (prompt/config/model) → deploy → repeat.

**Q7.** NAT redaction processors operate at which level?
- A) At the LLM prompt level before inference
- B) At the OTel exporter level before telemetry leaves the agent
- C) At the dashboard level when traces are displayed
- D) At the Kubernetes network level

**Answer: B** — Redaction happens at the exporter level, ensuring all downstream systems receive clean data.

**Q8.** A guardrail block rate rises from 2% to 15% after a model update. What is the most likely cause?
- A) Users are sending more harmful content
- B) The new model generates outputs that trigger false positives in existing guardrails
- C) The guardrails configuration was accidentally deleted
- D) Network latency is causing guardrail timeouts

**Answer: B** — A model change alters output patterns, which may trigger guardrails calibrated for the old model. Re-calibrate rails for the new model.

**Q9.** How should NeMo Guardrails traces be correlated with NAT workflow traces?
- A) By timestamp matching
- B) By trace context propagation (shared trace ID)
- C) By user ID lookup
- D) They cannot be correlated

**Answer: B** — OpenTelemetry trace context propagation ensures guardrails spans share the same trace ID as the parent NAT workflow.

**Q10.** When should you change agent architecture rather than tune prompts?
- A) When latency is too high
- B) When the agent type, memory strategy, or tool design is fundamentally wrong for the task
- C) When token costs exceed budget
- D) When users report formatting issues

**Answer: B** — Architecture changes are reserved for fundamental design mismatches. Prompt tuning handles most quality issues; architecture changes are expensive and risky.

---

## L. Mini Project

**Build an observability runbook for a production agent.**

Deliverables:
1. Dashboard specification: list every metric, its data source (NAT span attribute), and its visualization type
2. Alert rules document: 8 alerting rules with thresholds, severity levels, and response procedures
3. Debugging playbook: for each of the 5 common failure patterns (LLM timeout, tool failure, infinite loop, context overflow, guardrail false positive), write a step-by-step trace investigation procedure
4. Redaction configuration: define regex patterns and field rules for a healthcare agent that processes patient queries
5. Data flywheel plan: how production failures get fed back into the evaluation → improvement loop

---

## M. How This May Appear on the Exam

1. **Integration selection**: "Which NAT observability integration supports self-hosting and is open-source?" Expect questions distinguishing Phoenix, Weave, and Langfuse capabilities.

2. **Trace interpretation**: "An agent trace shows [description]. What is the most likely failure mode?" Expect scenario-based debugging questions.

3. **Redaction configuration**: "Which NAT component ensures PII is removed from telemetry?" Test knowledge of redaction processors and their position in the pipeline.

4. **Metrics selection**: "Which metric best indicates that an agent is stuck in a loop?" Test ability to select the right monitoring metric for a given concern.

5. **Maintenance strategy**: "After updating the LLM model behind an agent, guardrail block rates increase. What is the recommended response?" Test understanding of the relationship between model changes and guardrail calibration.

---

## N. Checklist for Mastery

- [ ] I can explain why distributed tracing is necessary for agent systems (vs. simple logging)
- [ ] I can configure NAT to export traces to Phoenix, Weave, or Langfuse
- [ ] I can read a multi-level execution trace and identify latency bottlenecks, cost drivers, and failures
- [ ] I can describe the OpenTelemetry span hierarchy for a typical NAT agent execution
- [ ] I can configure NAT redaction processors with regex patterns for common PII types
- [ ] I can list the key metrics for an agent monitoring dashboard (success rate, latency percentiles, token cost, guardrail rate)
- [ ] I can debug each of the 5 common agent failure patterns using traces
- [ ] I can use `nat_test_llm` for deterministic replay of agent failures
- [ ] I can correlate NeMo Guardrails traces with NAT workflow traces via trace context propagation
- [ ] I can design alerting rules with appropriate thresholds and severity levels
- [ ] I can describe the data flywheel and how NAT evaluation framework supports it
- [ ] I can plan model updates using blue-green deployment with quality monitoring
- [ ] I can explain when to tune prompts vs. change configuration vs. swap models vs. change architecture
