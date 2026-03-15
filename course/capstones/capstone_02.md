# Capstone 2: Multi-Agent System with Deployment and Governance

**Builds on:** Lab 7 (multi-agent), Labs 10-12 (deployment + monitoring + human oversight)
**Estimated time:** 8-10 hours
**Exam domains exercised:** Architecture (15%), Agent Development (15%), Deployment (13%), Monitor (5%), Human-AI (5%), Safety (5%)

---

## 1. Problem Statement

Build a multi-agent system for automated code review and documentation generation. The system has four agents:

1. **Router Agent**: Receives code submissions, classifies the task type, and dispatches to the appropriate specialist agent(s).
2. **Code Analysis Agent**: Analyzes submitted code for bugs, performance issues, security vulnerabilities, and style violations. Produces a structured analysis report.
3. **Documentation Agent**: Generates API documentation, inline comments, and README content from code and analysis output.
4. **Review Agent**: Reviews outputs from the other agents for quality, completeness, and accuracy. Flags items that need human approval.

The system must be:
- **Deployable**: Each agent runs as a separate service, communicable via A2A (Agent-to-Agent) protocol.
- **Observable**: Distributed tracing spans the full request lifecycle across all agents.
- **Safe**: No secrets or credentials appear in generated documentation. Agents stay in their assigned domain.
- **Human-governed**: Documentation publication requires human approval. Uncertain code analysis results are escalated.

This is not four prompts in a loop. This is a distributed system where agents coordinate, hand off work, and respect governance boundaries.

---

## 2. System Architecture

```
┌───────────────────────────────────────────────────────────────────────┐
│                    MULTI-AGENT CODE REVIEW SYSTEM                     │
│                                                                       │
│                        ┌──────────────────┐                          │
│                        │   USER / CI/CD    │                          │
│                        │   (Code Submit)   │                          │
│                        └────────┬─────────┘                          │
│                                 │                                     │
│                                 ▼                                     │
│                   ┌─────────────────────────┐                        │
│                   │     ROUTER AGENT        │                        │
│                   │   (NAT Router Agent)    │                        │
│                   │                         │                        │
│                   │  - Classify request     │                        │
│                   │  - Dispatch to agents   │                        │
│                   │  - Aggregate results    │                        │
│                   └───┬────────┬────────┬───┘                        │
│                       │        │        │                             │
│            ┌──────────┘        │        └──────────┐                 │
│            ▼                   ▼                    ▼                 │
│  ┌─────────────────┐ ┌────────────────┐ ┌──────────────────┐        │
│  │  CODE ANALYSIS  │ │ DOCUMENTATION  │ │  REVIEW AGENT    │        │
│  │  AGENT          │ │ AGENT          │ │                  │        │
│  │                 │ │                │ │  - Quality check │        │
│  │  - Bug detect   │ │ - API docs    │ │  - Completeness  │        │
│  │  - Security     │ │ - README gen  │ │  - Accuracy      │        │
│  │  - Performance  │ │ - Comments    │ │  - Escalation    │        │
│  │  - Style        │ │               │ │    decisions     │        │
│  │                 │ │               │ │                  │        │
│  │  NAT Tool       │ │ NAT Tool     │ │ NAT Tool         │        │
│  │  Calling Agent  │ │ Calling Agent │ │ Calling Agent    │        │
│  └────────┬────────┘ └───────┬───────┘ └────────┬─────────┘        │
│           │                  │                   │                   │
│           │    A2A Protocol  │    A2A Protocol   │                   │
│           │    (NAT A2A     │    (NAT A2A       │                   │
│           │     Server)      │     Server)        │                  │
│           │                  │                   │                   │
│  ┌────────▼──────────────────▼───────────────────▼─────────┐        │
│  │                  SHARED INFRASTRUCTURE                    │        │
│  │                                                          │        │
│  │  ┌──────────┐  ┌──────────┐  ┌────────────────────┐    │        │
│  │  │  Redis   │  │  NIM LLM │  │  NeMo Guardrails   │    │        │
│  │  │ (State)  │  │ Endpoint │  │  (Code Safety)     │    │        │
│  │  └──────────┘  └──────────┘  └────────────────────┘    │        │
│  │                                                          │        │
│  │  ┌──────────────────────────────────────────────┐       │        │
│  │  │  NAT Observability (OpenTelemetry → Phoenix) │       │        │
│  │  └──────────────────────────────────────────────┘       │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                       │
│  ┌───────────────────────────────────────────────────────────┐       │
│  │                 HUMAN OVERSIGHT LAYER                      │       │
│  │  Approval Queue │ Escalation Dashboard │ Feedback Loop     │       │
│  └───────────────────────────────────────────────────────────┘       │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 3. Data Flow

### Primary Flow: Code Review + Documentation

1. **Code Submission**: Developer submits code via API (or CI/CD webhook triggers submission). Payload includes code files, commit message, and request type (review, document, or both).

2. **Router Classification**: Router Agent receives the request and classifies it:
   - "review-only" → dispatch to Code Analysis Agent only
   - "document-only" → dispatch to Documentation Agent only
   - "full" → dispatch to Code Analysis Agent, then Documentation Agent (sequential dependency: docs benefit from analysis output)

3. **Code Analysis** (via A2A):
   - Router sends code to Code Analysis Agent via NAT A2A Server.
   - Agent uses NIM LLM endpoint with specialized tools (static analysis integration, security pattern matching, style guide lookup).
   - Produces a structured JSON report: `{bugs: [...], security: [...], performance: [...], style: [...], confidence: float}`.
   - If confidence < 0.7 on any finding, flags for escalation.

4. **Documentation Generation** (via A2A):
   - Router sends code + analysis report to Documentation Agent via NAT A2A Server.
   - Agent generates: API documentation (docstrings, type signatures), README section, inline comment suggestions.
   - NeMo Guardrails execution rail scans generated documentation for secrets/credentials patterns.
   - Produces a structured output with documentation artifacts.

5. **Quality Review** (via A2A):
   - Router sends all outputs to Review Agent via NAT A2A Server.
   - Review Agent evaluates:
     - Code analysis: Are findings actionable? Are they accurate based on the code?
     - Documentation: Is it complete? Does it match the code? Is it clear?
   - Produces a quality report with pass/fail for each artifact and specific feedback for failures.

6. **Human Approval Gate**:
   - If all artifacts pass review AND confidence is high → auto-approve minor items (style suggestions, comment additions).
   - If documentation is being published externally → always require human approval.
   - If code analysis confidence is low on any security finding → escalate to human reviewer.
   - Approval request placed in queue with full context (code, analysis, docs, review feedback).

7. **Output Delivery**:
   - Approved artifacts are returned to the user/CI system.
   - Rejected artifacts are returned with Review Agent feedback for the user to act on.
   - All decisions logged for audit trail.

---

## 4. Components

| Component | NVIDIA Tool | Role |
|-----------|-------------|------|
| Router Agent | NAT Router Agent | Classifies requests and dispatches to specialist agents based on request type |
| Code Analysis Agent | NAT Tool Calling Agent | Analyzes code using LLM + static analysis tools. Runs as standalone A2A service. |
| Documentation Agent | NAT Tool Calling Agent | Generates documentation artifacts from code and analysis. Runs as standalone A2A service. |
| Review Agent | NAT Tool Calling Agent | Reviews outputs from other agents for quality. Runs as standalone A2A service. |
| Inter-agent communication | NAT A2A Server | Provides A2A protocol endpoints for agent-to-agent communication |
| LLM inference | NIM LLM Endpoint | Shared LLM inference for all agents. Each agent uses different system prompts and tool sets. |
| Code safety guardrails | NeMo Guardrails (library mode for development; microservice mode for production) | Execution rails: scan generated docs for secrets/credentials. Content rail: agents stay in domain. |
| Shared state | Redis | Store intermediate results, approval queue, session state across agents |
| Observability | NAT Observability (OpenTelemetry) | Distributed tracing across all agents. Trace correlation via shared request ID. |
| Authentication | NAT Authentication | API key / OAuth2 for external access; inter-service auth for A2A communication |
| API gateway | NAT FastAPI Server (on Router) | External-facing API for code submission and result retrieval |

---

## 5. Evaluation Plan

### 5.1 Code Analysis Accuracy

| Metric | Target | How Measured |
|--------|--------|-------------|
| Bug detection precision | >= 0.80 | Compare agent findings against human expert reviews on 50 code samples. Precision = true bugs / all reported bugs. |
| Bug detection recall | >= 0.70 | Of known bugs in test samples, what fraction does the agent find? |
| Security finding accuracy | >= 0.85 | Security findings validated against known vulnerabilities in test corpus. Higher bar because false negatives are dangerous. |
| False positive rate | < 15% | Fraction of reported issues that are not actual problems. Measured by human expert validation. |

### 5.2 Documentation Quality

| Metric | Target | How Measured |
|--------|--------|-------------|
| Completeness | >= 0.85 | LLM-as-Judge via NAT custom evaluator: does the documentation cover all public functions, parameters, return types? |
| Accuracy | >= 0.90 | LLM-as-Judge: does the documentation accurately describe what the code does? Validated against manual documentation for test set. |
| Clarity | >= 0.80 | LLM-as-Judge: is the documentation clear and readable for the target audience (developers)? |
| Secret leakage | 0 incidents | Automated scan of all generated documentation for API keys, passwords, tokens, connection strings. |

### 5.3 System Performance

| Metric | Target | How Measured |
|--------|--------|-------------|
| End-to-end latency (review only) | p95 < 15 seconds | NAT Profiler: from submission to analysis report delivery. |
| End-to-end latency (full pipeline) | p95 < 30 seconds | NAT Profiler: from submission through analysis, documentation, and review. |
| Router overhead | < 2 seconds | NAT Profiler: time spent in routing classification and dispatch. |
| A2A communication overhead | < 500ms per hop | NAT Profiler: network + serialization time for each inter-agent call. |
| Coordination overhead ratio | < 20% of total | Profiler analysis: total time in routing + A2A + aggregation vs. total time in agent work. |

### 5.4 Agent Coordination Evaluation

| Metric | Target | How Measured |
|--------|--------|-------------|
| Routing accuracy | >= 0.95 | Of 100 test requests with known correct routing, how many are dispatched correctly? |
| Information passing completeness | >= 0.90 | Does the Documentation Agent receive all relevant analysis context? Verified by checking agent inputs. |
| Review consistency | >= 0.85 | When Review Agent rates the same output twice, do ratings agree? (Inter-rater reliability) |

### 5.5 Trajectory Evaluation

Use NAT trajectory evaluation to assess the full multi-agent workflow:
- Did the router make the correct dispatch decision?
- Did agents use tools in the expected order?
- Were unnecessary tool calls avoided?
- Did the review agent catch planted quality issues?

Build a trajectory test set of 30 scenarios with expected agent sequences and evaluate against actual trajectories.

---

## 6. Safety and Governance

### 6.1 Execution Rails

| Rail | Implementation | Protects Against |
|------|---------------|-----------------|
| Human approval for publication | NeMo Guardrails execution rail + approval queue | Auto-publishing incorrect or incomplete documentation |
| Secret detection | NeMo Guardrails execution rail with regex + pattern matching | API keys, passwords, tokens, connection strings in generated docs |
| Tool call validation | NeMo Guardrails execution rail per agent | Agents calling tools outside their authorized set |

**Human Approval Configuration**:
```colang
define flow documentation publication
  # Documentation agent produces output
  $doc = execute generate_documentation(...)

  # Always require human approval for publication
  $approved = execute request_human_approval(
    artifact=$doc,
    reason="Documentation ready for publication",
    timeout=3600
  )

  if $approved
    execute publish_documentation($doc)
  else
    bot say "Documentation was not approved. Review feedback has been recorded."
```

### 6.2 Content Safety

| Rail | Implementation | Protects Against |
|------|---------------|-----------------|
| Secret scanning | Custom output rail with regex patterns for common secret formats | Credentials, API keys, tokens appearing in any agent output |
| Code injection prevention | Input rail on Documentation Agent | Malicious code in documentation that could execute if rendered |
| Scope enforcement | Topical rail per agent | Code Analysis Agent attempting to generate documentation; Documentation Agent attempting security analysis |

### 6.3 Domain Isolation

Each agent has a Colang configuration that restricts its behavior:

```colang
# Code Analysis Agent - domain restriction
define user ask for documentation
  "Generate documentation for this code"
  "Write a README"
  "Add docstrings"

define flow code analysis domain
  user ask for documentation
  bot say "I only perform code analysis. Documentation requests should go through the Router."
```

---

## 7. Deployment Plan

### Docker Compose Architecture

```yaml
services:
  # Router Agent (external-facing)
  router:
    image: multi-agent-router:latest
    build:
      context: .
      dockerfile: Dockerfile.router
    ports:
      - "8000:8000"
    environment:
      - CODE_AGENT_URL=http://code-agent:8001
      - DOC_AGENT_URL=http://doc-agent:8002
      - REVIEW_AGENT_URL=http://review-agent:8003
      - NIM_ENDPOINT=http://nim-llm:8000
      - REDIS_HOST=redis
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://phoenix:4317
    depends_on:
      - code-agent
      - doc-agent
      - review-agent
      - redis

  # Code Analysis Agent (A2A service)
  code-agent:
    image: multi-agent-code:latest
    build:
      context: .
      dockerfile: Dockerfile.code-agent
    ports:
      - "8001:8001"
    environment:
      - NIM_ENDPOINT=http://nim-llm:8000
      - REDIS_HOST=redis
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://phoenix:4317

  # Documentation Agent (A2A service)
  doc-agent:
    image: multi-agent-doc:latest
    build:
      context: .
      dockerfile: Dockerfile.doc-agent
    ports:
      - "8002:8002"
    environment:
      - NIM_ENDPOINT=http://nim-llm:8000
      - REDIS_HOST=redis
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://phoenix:4317

  # Review Agent (A2A service)
  review-agent:
    image: multi-agent-review:latest
    build:
      context: .
      dockerfile: Dockerfile.review-agent
    ports:
      - "8003:8003"
    environment:
      - NIM_ENDPOINT=http://nim-llm:8000
      - REDIS_HOST=redis
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://phoenix:4317

  # Shared NIM LLM Endpoint
  nim-llm:
    image: nvcr.io/nim/meta/llama-3.1-70b-instruct:latest
    ports:
      - "8080:8000"
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 2
              capabilities: [gpu]

  # Shared State
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data

  # Observability
  phoenix:
    image: arizephoenix/phoenix:latest
    ports:
      - "6006:6006"
      - "4317:4317"

volumes:
  redis-data:
```

### Service Separation Rationale

Each agent runs as a separate Docker container with its own:
- NAT YAML configuration (agent type, tools, system prompt)
- NeMo Guardrails configuration (domain-specific rails)
- A2A endpoint for inter-agent communication
- Independent health check and restart policy

This separation enables:
- Independent scaling (scale Code Analysis Agent separately from Documentation Agent)
- Independent deployment (update one agent without restarting others)
- Fault isolation (one agent crashing does not take down the system)
- Independent resource limits (Code Analysis Agent may need more memory for large codebases)

---

## 8. Observability

### Distributed Tracing

The core observability challenge in a multi-agent system is **trace correlation**. A single user request touches 4 agents across 4 services. Without correlation, debugging is impossible.

**Implementation**:
1. Router generates a unique `trace_id` for each incoming request.
2. `trace_id` is propagated via A2A message headers to all downstream agents.
3. Each agent's NAT observability configuration exports spans to the shared Phoenix instance.
4. All spans for a single request are correlated by `trace_id`.

**Trace structure for a "full" request**:
```
[trace_id: abc-123]
├── router.classify (200ms)
├── router.dispatch_code_analysis (50ms)
│   └── code_agent.analyze (8s)
│       ├── code_agent.tool.static_analysis (2s)
│       ├── code_agent.tool.security_scan (1.5s)
│       └── code_agent.llm.generate_report (4s)
├── router.dispatch_documentation (50ms)
│   └── doc_agent.generate (6s)
│       ├── doc_agent.llm.generate_api_docs (3s)
│       ├── doc_agent.llm.generate_readme (2s)
│       └── doc_agent.guardrail.secret_scan (200ms)
├── router.dispatch_review (50ms)
│   └── review_agent.review (5s)
│       ├── review_agent.llm.review_analysis (2.5s)
│       └── review_agent.llm.review_docs (2.5s)
└── router.aggregate_results (100ms)
```

### Key Dashboards

| Dashboard | Metrics | Purpose |
|-----------|---------|---------|
| System Overview | Total requests, success rate, end-to-end latency distribution | High-level health |
| Agent Performance | Per-agent latency, throughput, error rate | Identify which agent is the bottleneck |
| Routing Accuracy | Classification distribution, dispatch success rate | Verify router is making correct decisions |
| A2A Health | Inter-agent latency, message delivery rate, timeout rate | Monitor communication infrastructure |
| Human Oversight | Approval queue depth, average approval time, approval/rejection ratio | Monitor the human governance bottleneck |
| Cost Tracking | Token usage per agent, per request type, total NIM cost | Budget monitoring |

### Key Alerts

- **A2A timeout rate > 5%**: Agent communication degradation. Check network and agent health.
- **Review agent rejection rate > 30%**: Quality issues in upstream agents. Review recent prompt or configuration changes.
- **Approval queue depth > 50**: Human reviewers cannot keep up. Consider relaxing auto-approval thresholds for low-risk items.
- **Single agent error rate > 10%**: Agent health issue. Check logs, restart if necessary.

---

## 9. Scaling Concerns

### Independent Scaling Strategy

| Agent | Scaling Trigger | Scaling Action |
|-------|----------------|----------------|
| Router | Request rate > 100/min | Add router replicas behind load balancer. Router is lightweight (classification only). |
| Code Analysis | Code review queue depth > 20 | Add code-agent replicas. This is typically the most compute-intensive agent. |
| Documentation | Documentation queue depth > 10 | Add doc-agent replicas. Moderate compute needs. |
| Review | Review queue depth > 15 | Add review-agent replicas. Review is faster than generation but serializes the pipeline. |
| NIM LLM | GPU utilization > 80% on any NIM instance | Add NIM replicas. Requires additional GPU allocation. |

### When Code Reviews Spike

A CI/CD pipeline triggering 50 concurrent code reviews creates a different load profile than normal usage:

1. **First bottleneck**: Code Analysis Agent. Scale to 5-10 replicas.
2. **Second bottleneck**: NIM LLM endpoint (all code analysis requests hit the LLM). Scale NIM or implement request batching.
3. **Router**: Lightweight, handles 50 concurrent dispatches easily. No scaling needed.
4. **Review Agent**: Receives results sequentially as code analyses complete. Minor scaling may help.
5. **Documentation Agent**: Idle during a code-review-only spike.

### When Documentation Requests Spike

A major release requiring documentation for 100 new APIs:

1. **First bottleneck**: Documentation Agent. Scale to 5-10 replicas.
2. **Second bottleneck**: Review Agent (every doc goes through review). Scale to 3-5 replicas.
3. **Third bottleneck**: Human approval queue. This is the true bottleneck. Technical scaling cannot fix it — need to either batch approvals or delegate approval authority.
4. **Code Analysis Agent**: Idle during a documentation-only spike.

### Shared Resource Contention

All agents share the NIM LLM endpoint and Redis. Under combined load:
- NIM LLM: Implement per-agent request quotas to prevent one agent from starving others.
- Redis: Low risk at moderate scale. Redis Cluster at 1,000+ concurrent requests.

---

## 10. Human Oversight

### 10.1 Approval Workflow

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│ Review Agent │────►│ Approval     │────►│ Human       │
│ passes      │     │ Queue        │     │ Reviewer    │
│ quality     │     │ (Redis)      │     │ Dashboard   │
│ check       │     └──────────────┘     └──────┬──────┘
└─────────────┘                                  │
                                          ┌──────┴──────┐
                                          │             │
                                     ┌────▼───┐   ┌────▼───┐
                                     │Approve │   │Reject  │
                                     │+ Notes │   │+ Notes │
                                     └────┬───┘   └────┬───┘
                                          │             │
                                     ┌────▼───┐   ┌────▼────┐
                                     │Publish │   │Return   │
                                     │        │   │feedback │
                                     └────────┘   │to user  │
                                                  └─────────┘
```

**Auto-approval rules** (configurable):
- Style-only suggestions with high confidence → auto-approve
- Internal-only documentation for non-sensitive code → auto-approve
- Security findings with confidence > 0.95 and severity "low" → auto-approve

**Mandatory human review**:
- Any documentation marked for external publication
- Security findings with severity "high" or "critical"
- Any analysis where agent confidence < 0.7
- First 50 items from a new codebase (calibration period)

### 10.2 Escalation Paths

| Trigger | Escalation Action | Who Receives |
|---------|-------------------|--------------|
| Code Analysis confidence < 0.5 on security finding | Immediate escalation | Security team lead |
| Code Analysis confidence 0.5-0.7 on any finding | Queue for senior review | Senior developer |
| Documentation references internal APIs not in scope | Flag and hold | Documentation lead |
| Review Agent disagrees with Code Analysis Agent | Both outputs sent for human arbitration | Technical lead |
| Any agent produces an error on a request | Log, retry once, then escalate | On-call engineer |

### 10.3 Feedback Loop

Human decisions feed back into the system:

1. **Approval/rejection data** is logged with the reviewer's notes.
2. Weekly aggregation identifies patterns: which agents produce the most rejections? Which types of code produce low-confidence analyses?
3. Feedback is used to:
   - Refine agent system prompts (if Documentation Agent consistently gets format wrong, update prompt).
   - Adjust auto-approval thresholds (if human reviewers approve 99% of style suggestions, widen auto-approval).
   - Identify training data for potential fine-tuning via NAT Finetuning (DPO with NeMo Customizer).
4. NAT eval framework is used to measure whether feedback-driven changes actually improve quality metrics.

---

## 11. What Makes This Weak vs. Strong

### Weak Submission Characteristics

- **All agents in one process**: Four "agents" are four functions in a single Python script. No A2A protocol, no service separation, no independent scaling. This is a pipeline, not a multi-agent system.
- **No A2A communication**: Agents share memory directly. No protocol, no serialization, no network boundary. Impossible to deploy as separate services.
- **No evaluation**: "The documentation looks right" is not evaluation. No precision/recall on code analysis, no LLM-as-Judge on documentation, no trajectory evaluation.
- **No approval workflow**: Generated documentation is returned directly to the user. No human gate, no review queue, no escalation for uncertain findings.
- **No distributed tracing**: When something goes wrong, there is no way to trace the request across agents. Debugging requires reading logs from four services and manually correlating timestamps.
- **No domain isolation**: Code Analysis Agent can be prompted to generate documentation. Documentation Agent can be prompted to analyze code. No Colang rules enforce boundaries.
- **Single NIM endpoint with no resource management**: All agents compete for the same LLM endpoint with no quotas or priority.

### Strong Submission Characteristics

- **True service separation via A2A**: Each agent is a separate Docker service with its own NAT A2A Server endpoint. Communication uses the A2A protocol with structured messages and typed payloads.
- **Comprehensive trajectory evaluation**: Test suite of 30+ scenarios with expected agent sequences. Trajectory evaluation verifies correct routing, tool use order, and inter-agent data passing.
- **Human approval gates**: Configurable approval workflow with auto-approve rules for low-risk items and mandatory human review for high-risk items. Approval queue with dashboard.
- **Distributed observability**: OpenTelemetry traces propagated across all services via trace ID. Single Phoenix dashboard shows end-to-end request flow across all agents. Per-agent performance dashboards.
- **Independent scaling**: Docker Compose with per-service resource limits and replica configuration. Demonstrated ability to scale Code Analysis Agent independently of Documentation Agent.
- **Documented coordination patterns**: README explains why the Router pattern was chosen over hierarchical, how the sequential dependency between analysis and documentation is managed, and what happens when an agent fails mid-request.
- **Domain isolation via Colang**: Each agent has Colang rules restricting it to its domain. Demonstrated with test cases showing domain boundary enforcement.
- **Feedback loop**: Human approval data logged and aggregated. At least one example of a prompt refinement driven by approval feedback, with before/after evaluation metrics.

---

## 12. Deliverables Checklist

- [ ] Source code for all four agents (NAT YAML configs, Guardrails configs, application code)
- [ ] Docker Compose file that brings up all services (router + 3 specialists + NIM + Redis + Phoenix)
- [ ] A2A communication protocol implementation (message schemas, routing logic)
- [ ] NeMo Guardrails configuration for each agent (domain isolation, secret detection, execution rails)
- [ ] Evaluation suite: code analysis accuracy tests, documentation quality tests (LLM-as-Judge), trajectory evaluation tests
- [ ] Evaluation report showing all metrics
- [ ] Human approval workflow: queue implementation, dashboard mockup or implementation, auto-approve rules
- [ ] Observability: distributed tracing configuration, trace correlation demonstration, dashboard definitions
- [ ] Architecture diagram (can be the ASCII diagram above, refined)
- [ ] README documenting: setup instructions, architecture decisions, coordination patterns, scaling strategy, known limitations
- [ ] 5-minute demo showing: a code submission flowing through all agents, distributed trace in Phoenix, a human approval decision, domain isolation blocking an out-of-scope request
