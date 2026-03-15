# Module 9: Safety, Guardrails, and Governance

**Primary exam domains:** Safety, Ethics, and Compliance (5%)
**NVIDIA tools:** NeMo Guardrails (Colang 2.0 DSL, 5 rail types, LLMRails API, RailsConfig, CLI, FastAPI server, Docker, LangChain/LangGraph integration, built-in guardrails library), Content Safety NIM, Topic Control NIM, Jailbreak Detection NIM, NeMo Auditor, NAT defense middleware, NAT red-teaming middleware, NAT redaction processors

---

## A. Module Title

**Safety, Guardrails, and Governance: Protecting Agentic Systems in Production**

---

## B. Why This Module Matters in Real Systems

An agent that can call tools, access databases, and take actions on behalf of users is a security-critical system. Unlike a static web page or a read-only API, an agentic system can execute code, send messages, modify data, and make purchases. A single prompt injection that bypasses safety checks can cause real-world damage — financial, reputational, or regulatory. Safety is not a feature you add after the system works. It is a structural requirement that shapes every architectural decision.

The challenge is that safety for agents is harder than safety for chatbots. A chatbot's attack surface is its input prompt. An agent's attack surface includes input prompts, retrieved documents (indirect injection), tool parameters (tool injection), multi-turn conversation manipulation, and the tool outputs themselves. Each of these vectors requires distinct defensive measures.

NVIDIA provides a layered safety toolkit: NeMo Guardrails for programmable input/output/execution/dialog/topical rails, Safety NIMs for specialized detection (content safety, jailbreak, topic control), NeMo Auditor for pre-deployment vulnerability assessment, and NAT middleware for defense and red-teaming within agent workflows. This module teaches you to layer these tools into a defense-in-depth architecture where no single failure compromises the system.

---

## C. Learning Objectives

After completing this module, you will be able to:

1. Describe the 4-stage safety lifecycle (Evaluate, Post-Train, Deploy, Runtime) as a conceptual framework and map current NVIDIA tools to each stage.
2. Configure NeMo Guardrails with all five rail types: input, output, dialog, retrieval, and execution rails.
3. Write Colang 2.0 code for content safety, topic control, and tool validation guardrails.
4. Deploy NeMo Guardrails via Python API, CLI, and FastAPI server.
5. Integrate Content Safety NIM, Jailbreak Detection NIM, and Topic Control NIM into an agent safety architecture.
6. Configure NAT defense middleware and red-teaming middleware for agent workflows.
7. Implement PII detection and redaction using NeMo Guardrails integrations and NAT redaction processors.
8. Use NeMo Auditor for pre-deployment safety assessment and vulnerability scanning.

---

## D. Required Concepts

- Completion of Module 5 (Agentic Reasoning) — you must understand how agents select and execute tools.
- Completion of Module 6 (NAT Development) — you must understand NAT configuration and middleware chains.
- Basic understanding of prompt injection and adversarial attacks on LLMs.
- Familiarity with LLM inference flow (prompt in, response out) and where interception points exist.

---

## E. Core Lesson Content

### [BEGINNER] The 4-Stage Safety Lifecycle (Conceptual Framework)

*This subsection presents a conceptual framing for understanding where safety measures apply in the agent lifecycle. It is not a prescriptive implementation guide — current NVIDIA production tooling is covered in subsequent sections.*

```
4-STAGE SAFETY LIFECYCLE
========================
+------------+    +------------+    +----------+    +-----------+
|  Stage 1   |    |  Stage 2   |    | Stage 3  |    |  Stage 4  |
|  EVALUATE  |--->| POST-TRAIN |--->|  DEPLOY  |--->|  RUNTIME  |
| Vuln scan, |    | Safety     |    | Trusted  |    | Inference |
| benchmarks |    | alignment  |    | model    |    | guardrails|
+------------+    +------------+    +----------+    +-----------+
  NeMo Auditor     DPO/SFT with      NIM deploy     NeMo Guardrails
                   safety data                       Safety NIMs
                                                     NAT middleware
```

**Stage 1 — Evaluate:** Before deployment, scan models and agents for vulnerabilities. Identify safety gaps through adversarial testing and safety benchmarks. NVIDIA tool: NeMo Auditor.

**Stage 2 — Post-Train:** Align models with safety objectives through supervised finetuning (SFT) or reinforcement learning with safety-focused datasets. This embeds safety behaviors into the model itself.

**Stage 3 — Deploy:** Use trusted, validated models deployed through secure infrastructure (NIM). Ensure the deployed model matches what was evaluated and aligned.

**Stage 4 — Runtime:** Apply inference-time guardrails that intercept and validate inputs, outputs, tool calls, and conversational flows. NVIDIA tools: NeMo Guardrails, Safety NIMs, NAT defense middleware.

**Historical note:** The Safety for Agentic AI Blueprint (deprecated April 22, 2026) was the original NVIDIA implementation of this lifecycle. It provided an integrated pipeline spanning evaluation through runtime guardrails. NVIDIA now recommends using NeMo Microservices (NeMo Guardrails, NeMo Auditor, Safety NIMs) as the current production tooling. The lifecycle model remains useful as a conceptual frame, but all labs and operational guidance in this module use current tools.

### [BEGINNER] NeMo Guardrails Architecture Overview

NeMo Guardrails intercepts the LLM interaction at multiple points:

```
GUARDRAILS ARCHITECTURE
=======================

User Input
    |
    v
+---+---+---+---+---+---+---+---+---+
|       INPUT RAILS                  |  <-- Validate/modify input
|  Content safety, jailbreak detect, |      before LLM sees it
|  PII detection, topic check        |
+---+---+---+---+---+---+---+---+---+
    |
    v
+---+---+---+---+---+---+---+---+---+
|       DIALOG RAILS                 |  <-- Enforce conversation
|  Pre-defined flows, patterns,      |      patterns (cross-cutting)
|  canonical forms                   |
+---+---+---+---+---+---+---+---+---+
    |
    v
+---+---+---+---+---+---+---+---+---+
|       LLM PROCESSING               |
|  + RETRIEVAL RAILS                 |  <-- Filter/modify retrieved
|    (filter chunks, validate        |      chunks in RAG scenarios
|     retrieved context)             |
+---+---+---+---+---+---+---+---+---+
    |
    v
+---+---+---+---+---+---+---+---+---+
|       EXECUTION RAILS              |  <-- Validate tool calls
|  Check tool inputs, validate       |      before execution
|  tool outputs, access control      |
+---+---+---+---+---+---+---+---+---+
    |
    v
+---+---+---+---+---+---+---+---+---+
|       OUTPUT RAILS                 |  <-- Validate/modify LLM
|  Content safety, fact-check,       |      response before user
|  PII redaction, format check       |      sees it
+---+---+---+---+---+---+---+---+---+
    |
    v
Response to User
```

**The five rail types:**

1. **Input Rails:** Intercept user input before the LLM processes it. Use for content safety screening, jailbreak detection, PII detection, and topic validation.
2. **Output Rails:** Intercept LLM responses before returning to the user. Use for content safety, fact-checking, PII redaction, and format enforcement.
3. **Dialog Rails:** Define expected conversational patterns and enforce them. Use for keeping conversations on-track and handling sensitive topics with scripted flows.
4. **Retrieval Rails:** Filter or modify retrieved chunks in RAG scenarios. Use for removing irrelevant, outdated, or unsafe content from retrieval results.
5. **Execution Rails:** Validate tool call inputs and outputs. Use for preventing tool misuse, injection through tool parameters, and unauthorized tool access.

### [INTERMEDIATE] Content Safety Implementation

**NeMo Guardrails Input Rails** screen user messages before the LLM processes them. Configure input rails in `config.yml`:

```yaml
# config.yml
models:
  - type: main
    engine: nim
    model: meta/llama-3.1-70b-instruct

rails:
  input:
    flows:
      - content safety check input
      - jailbreak detection
      - pii detection input
  output:
    flows:
      - content safety check output
      - pii redaction output
```

**Content Safety NIM** provides specialized toxic/biased content detection. It runs as a separate microservice and can be called from within NeMo Guardrails as an action or integrated as an input/output rail.

**NAT defense middleware** provides built-in safety checks within the NAT middleware chain. These checks execute as part of the agent's request processing pipeline, catching safety issues at the agent framework level.

### [INTERMEDIATE] Tool Safety (Agentic Security)

Tool safety is where agentic systems diverge from chatbot safety. An agent that can execute code, query databases, or call APIs must validate every tool interaction.

**NeMo Guardrails Execution Rails** validate tool call inputs before execution and outputs before the agent processes them:

```colang
# Execution rail for tool validation
define flow validate tool call
  when tool_call
    if tool_call.name == "execute_sql"
      if contains(tool_call.parameters.query, "DROP") or \
         contains(tool_call.parameters.query, "DELETE")
        bot refuse dangerous operation
        stop
    execute tool_call
    when tool_result
      if tool_result.status == "error"
        bot report tool error gracefully
```

**Key tool safety concerns:**
- **Tool parameter injection:** A manipulated prompt causes the agent to pass malicious parameters to a tool (e.g., SQL injection through a database query tool).
- **Unauthorized tool access:** An agent calls a tool it should not have access to for the current user or context.
- **Tool output poisoning:** A tool returns malicious content (e.g., from a web search) that influences subsequent agent behavior.

**NAT per-user functions** provide tool access control, restricting which tools are available based on user identity or role.

### [INTERMEDIATE] Policy Enforcement with Dialog and Topical Rails

**Dialog Rails** pre-define conversational flows. When a conversation matches a defined pattern, the guardrails system follows the scripted flow instead of allowing free LLM generation:

```colang
# Colang 2.0: Dialog rail for handling refund requests
define user ask for refund
  "I want a refund"
  "Can I get my money back"
  "How do I return this"

define bot explain refund policy
  "Our refund policy allows returns within 30 days of purchase.
   I can help you start a return. Would you like to proceed?"

define flow handle refund request
  user ask for refund
  bot explain refund policy
```

**Topical Rails** constrain the agent to approved discussion topics:

```colang
# Colang 2.0: Topical rail
define user ask about competitors
  "What do you think about [competitor]?"
  "Is [competitor] better than you?"
  "Compare yourself to [competitor]"

define bot deflect competitor question
  "I'm here to help with questions about our products and services.
   How can I assist you today?"

define flow block competitor discussion
  user ask about competitors
  bot deflect competitor question
```

**Topic Control NIM** provides business domain compliance checking as a specialized microservice. It can be integrated as a rail action for domain-specific topic enforcement.

### [INTERMEDIATE] Colang 2.0 Deep Dive

Colang 2.0 is the domain-specific language for NeMo Guardrails. Key syntax elements:

```colang
# DEFINING USER INTENTS (canonical forms with examples)
define user greet
  "hello"
  "hi there"
  "good morning"

# DEFINING BOT RESPONSES
define bot greet back
  "Hello! How can I help you today?"

# CONDITIONAL FLOWS
define flow handle sensitive topic
  user ask about medical advice
  if user.verified_medical_professional
    bot provide medical information
  else
    bot redirect to medical professional
    stop

# GUARDRAIL LOGIC: Input content safety rail
define flow content safety check input
  $is_safe = execute content_safety_check(text=$user_message)
  if not $is_safe
    bot refuse unsafe content
    stop

# GUARDRAIL LOGIC: Execution rail for tool validation
define flow check tool authorization
  when tool_call
    $authorized = execute check_user_permissions(
      user=$context.user_id,
      tool=$tool_call.name
    )
    if not $authorized
      bot report unauthorized tool access
      stop
    execute tool_call
```

**Colang 2.0 vs. Colang 1.0:** Version 2.0 introduces proper control flow (if/else, while), variable scoping, event-driven patterns (when keyword), and more expressive guardrail logic. New projects should use Colang 2.0.

### [INTERMEDIATE] Jailbreak and Adversarial Defense

**Common attack patterns:**

- **Direct prompt injection:** "Ignore previous instructions and..." — attempting to override the system prompt.
- **Indirect prompt injection:** Malicious content embedded in retrieved documents or tool outputs that manipulates agent behavior.
- **Multi-turn manipulation:** Gradually steering the conversation across multiple turns to circumvent single-turn safety checks.
- **Encoding attacks:** Obfuscating harmful requests through base64, Unicode tricks, or language switching.

**Defense layers:**

1. **Jailbreak Detection NIM:** Specialized model trained to detect jailbreak attempts. Runs as a microservice, called from NeMo Guardrails input rails.
2. **NeMo Guardrails built-in jailbreak detection:** Self-check methods that use the main LLM to evaluate whether an input is a jailbreak attempt. Heuristic pattern matching for known attack templates. NemoGuard integration for enhanced detection.
3. **NAT red-teaming middleware:** Adversarial testing during development. Simulates attack patterns against the agent to find vulnerabilities before deployment.

```yaml
# config.yml - Jailbreak detection configuration
rails:
  input:
    flows:
      - jailbreak detection heuristics
      - jailbreak detection nim
      - content safety check input
```

**Defense in depth:** No single detection method catches all attacks. Layer multiple methods: heuristic patterns catch known attacks fast and cheap, NIM-based detection catches novel attacks, and self-check provides a last line of defense.

### [INTERMEDIATE] PII Detection and Data Privacy

**NeMo Guardrails PII detection** integrates with multiple providers:
- **GLiNER:** Open-source named entity recognition for PII detection.
- **Presidio:** Microsoft's PII detection and anonymization engine.
- **PrivateAI:** Commercial PII detection service.
- **AutoAlign:** Automated alignment-based PII handling.

**NAT redaction processors** handle PII at the agent framework level, redacting sensitive data from inputs, tool parameters, and outputs.

**When to mask vs. block:**
- **Mask (redact):** Replace PII with placeholders ("[EMAIL]", "[SSN]") when the query can still be processed without the actual PII values. Useful for logging and analytics.
- **Block:** Refuse to process the request entirely when PII presence indicates a safety or compliance violation (e.g., user submitting someone else's SSN).

### [ADVANCED] Fact-Checking and Hallucination Prevention

**NeMo Guardrails Retrieval Rails** filter and modify chunks in RAG scenarios before the LLM uses them for generation. This prevents the LLM from being influenced by irrelevant or contradictory retrieved content.

**Fact-checking rails** verify generated content against retrieved context or external knowledge:

```colang
define flow fact check output
  when bot_message
    $is_grounded = execute fact_check(
      response=$bot_message,
      context=$retrieved_chunks
    )
    if not $is_grounded
      bot provide verified response
```

**Hallucination prevention strategy:**
1. Retrieval rails filter irrelevant chunks before generation.
2. System prompt instructs the model to only use provided context.
3. Output rails fact-check the response against the context.
4. Confidence thresholds trigger human escalation when uncertainty is high.

### [ADVANCED] NeMo Auditor (Pre-Deployment Auditing)

NeMo Auditor provides pre-deployment safety assessment:

- **Audit jobs:** Automated vulnerability and safety assessment runs against models and agents.
- **Vulnerability scanning:** Identifies known vulnerability patterns in model behavior.
- **Safety scoring:** Produces quantitative safety scores across multiple dimensions (toxicity, bias, jailbreak susceptibility).

**Where it fits:** NeMo Auditor operates between evaluation and deployment in the lifecycle. It is the recommended replacement for the evaluation stage of the deprecated Safety for Agentic AI Blueprint. Run auditor jobs as a gate in CI/CD — agents that fail safety audits do not deploy.

### [ADVANCED / OPTIONAL] Safe Synthesizer

**Purpose:** Privacy-preserving synthetic data generation for safe model training. When you need to train models on sensitive data patterns without exposing real user data.

**Status:** Early Access. Not recommended as a backbone for safety implementation. Use when you specifically need to generate safety training data (adversarial examples, safety-aligned response pairs) without exposing real user data.

**When useful:** Organizations with strict data governance requirements that need to train safety-aligned models but cannot use production data directly.

### [INTERMEDIATE] NeMo Guardrails Deployment

NeMo Guardrails has two deployment tiers:

- **Library mode (easy)**: `pip install nemoguardrails langchain-nvidia-ai-endpoints` + `NVIDIA_API_KEY` + NVIDIA-hosted models. Runs on a CPU laptop. This is what all labs in this course use. Includes the Python API and CLI chat. The built-in FastAPI server requires `pip install nemoguardrails[server]`.
- **Microservice mode (heavy)**: NGC container deployment (`nvcr.io/nvidia/nemoguardrails`). Requires NGC API key, Docker, potentially GPU for Safety NIM models (Content Safety, Jailbreak Detection), and Kubernetes for production. This is optional and advanced — only relevant if you need a standalone guardrails service in a containerized production stack.

All examples below use **library mode** unless explicitly noted.

**Python API:**
```python
from nemoguardrails import LLMRails, RailsConfig

config = RailsConfig.from_path("./config")
rails = LLMRails(config)

response = rails.generate(
    messages=[{"role": "user", "content": "Hello, how can I get a refund?"}]
)
```

**CLI:**
```bash
# Interactive chat
nemoguardrails chat --config ./config

# Evaluate guardrails against test cases
nemoguardrails evaluate --config ./config --dataset ./test_cases.json

# Start server (requires: pip install nemoguardrails[server])
nemoguardrails server --config ./config --port 8000
```

**FastAPI server:** Exposes `/v1/chat/completions` endpoint compatible with OpenAI API format. Enables drop-in replacement for existing LLM API calls with guardrails wrapping.

**Docker containerization (microservice mode):** Package the guardrails configuration and server into a Docker container for production deployment. This is the **microservice mode** — it requires NGC access, Docker, and potentially GPU for Safety NIM models. Library mode (above) is sufficient for development and all labs in this course.

**Configuration structure:**
```
config/
  config.yml          # Models, rails mapping, settings
  config.py           # Custom Python actions and logic
  actions.py          # Custom action implementations
  content_safety.co   # Colang files defining rails
  topic_control.co    # Colang files for topical rails
  tool_validation.co  # Colang files for execution rails
```

### [INTERMEDIATE] config.yml Walkthrough

```yaml
# Full config.yml example
colang_version: "2.x"
models:
  - type: main
    engine: nim
    model: meta/llama-3.1-70b-instruct
  - type: content_safety
    engine: nim
    model: nvidia/content-safety

rails:
  input:
    flows:
      - content safety check input
      - jailbreak detection
      - pii detection input
      - topic check input
  output:
    flows:
      - content safety check output
      - pii redaction output
      - fact check output
  dialog:
    user_messages:
      embeddings_model: nvidia/nv-embedqa-e5-v5
  retrieval:
    flows:
      - filter irrelevant chunks
  execution:
    flows:
      - validate tool permissions

# Entity configuration for PII detection
sensitive_data_detection:
  input:
    entities:
      - email
      - phone_number
      - ssn
    provider: presidio
```

---

## F. Terminology Box

| Term | Definition |
|------|-----------|
| **Input Rails** | Guardrails that intercept and validate user input before LLM processing |
| **Output Rails** | Guardrails that validate and modify LLM responses before returning to the user |
| **Dialog Rails** | Pre-defined conversational flows that override free LLM generation for specific patterns |
| **Retrieval Rails** | Guardrails that filter or modify retrieved chunks in RAG scenarios |
| **Execution Rails** | Guardrails that validate tool call inputs and outputs |
| **Colang 2.0** | Domain-specific language for defining NeMo Guardrails logic |
| **Canonical Form** | In Colang, a normalized representation of user intent with example utterances |
| **Prompt Injection** | Attack that manipulates the prompt to override system instructions |
| **Indirect Injection** | Attack embedded in retrieved documents or tool outputs rather than user input |
| **PII Redaction** | Replacing personally identifiable information with placeholders |
| **NeMo Auditor** | Pre-deployment safety auditing tool for vulnerability and safety assessment |
| **Defense in Depth** | Layering multiple safety mechanisms so no single failure compromises the system |
| **Red Teaming** | Adversarial testing to find safety vulnerabilities before production |

---

## G. Common Misconceptions

1. **"Safety is just content filtering."** Content filtering (blocking toxic outputs) is one layer. Agentic safety also requires tool validation, dialog control, jailbreak defense, PII handling, and pre-deployment auditing. Content filtering alone misses most agent-specific attack vectors.

2. **"If the model is safety-aligned, you don't need guardrails."** Model alignment is Stage 2 of the lifecycle. Aligned models still fail under adversarial pressure, novel attack patterns, and edge cases. Runtime guardrails (Stage 4) are always required.

3. **"NeMo Guardrails replaces safety training."** Guardrails and safety training are complementary. Guardrails catch issues at runtime; safety training reduces the frequency of issues at the model level. Both are needed.

4. **"One jailbreak detection method is sufficient."** No single method catches all attacks. Production systems layer heuristic patterns, NIM-based detection, and self-check methods for defense in depth.

5. **"PII detection only matters for user inputs."** PII can appear in LLM outputs (hallucinated or retrieved), tool call parameters, tool outputs, and logged data. PII handling must cover all these vectors.

6. **"Safety slows the system down too much for production."** Well-configured guardrails add 50-200ms of latency. For most applications, this is acceptable. Profile with NAT Profiler to identify which rails are expensive and optimize selectively.

---

## H. Failure Modes / Anti-Patterns

1. **Output-only guardrails.** Filtering only the final response while leaving tool calls unguarded. An agent could execute a dangerous tool call and then have its response filtered — the damage is already done. Always include execution rails.

2. **Overly restrictive guardrails.** Guardrails so aggressive that they block 30% of legitimate requests. Users abandon the system or find workarounds. Measure false positive rates and tune thresholds.

3. **Static guardrails without red-teaming.** Deploying guardrails once and never testing them adversarially. Attack techniques evolve. Regular red-teaming (using NAT red-teaming middleware) is required.

4. **Missing indirect injection defense.** Protecting against user prompt injection but not against malicious content in retrieved documents or tool outputs. RAG systems are particularly vulnerable — a poisoned document in the knowledge base can manipulate the agent.

5. **Logging PII in guardrails audit trails.** Guardrails that detect and redact PII from responses but log the original un-redacted content for debugging. The logs become a PII leak. Redact before logging.

6. **No graceful degradation.** When a guardrail service (e.g., Content Safety NIM) is unavailable, the system either crashes or bypasses safety entirely. Design fallback behavior: degrade to a more restrictive mode, not a less restrictive one.

---

## I. Hands-On Lab

**Lab 9: NeMo Guardrails Configuration**

Set up a NeMo Guardrails configuration with input rails (content safety, jailbreak detection), output rails (content safety, PII redaction), and a topical rail that restricts the agent to a specific business domain. Write Colang 2.0 flows for each rail. Deploy via the Python API and test with both legitimate queries and adversarial inputs. Measure latency impact of the guardrails stack.

---

## J. Stretch Lab

**Stretch Lab 9: Full Safety Pipeline with Auditing**

Extend Lab 9 with: (1) Execution rails that validate tool call parameters for a database query tool (block SQL injection patterns). (2) Integration with Jailbreak Detection NIM as an input rail. (3) NAT red-teaming middleware to run adversarial probes. (4) NeMo Auditor pre-deployment audit job. Produce a safety report showing: guardrail trigger rates, false positive rates, red-teaming findings, and auditor safety scores.

---

## K. Review Quiz

**Q1:** Name the five types of rails in NeMo Guardrails.
**Answer:** Input rails, output rails, dialog rails, retrieval rails, and execution rails.

**Q2:** An agent with output rails but no execution rails executes a SQL DROP TABLE command before the output is filtered. Which rail type would have prevented this?
**Answer:** Execution rails, which validate tool call inputs before the tool is executed.

**Q3:** What is the difference between direct and indirect prompt injection?
**Answer:** Direct injection is in the user's input ("Ignore previous instructions..."). Indirect injection is embedded in retrieved documents, tool outputs, or other data sources that the agent processes.

**Q4:** In Colang 2.0, what does the `when` keyword do?
**Answer:** It defines event-driven patterns that trigger when a specific event occurs (e.g., `when tool_call` triggers when the agent attempts a tool call, allowing the flow to validate or block it).

**Q5:** What endpoint does the NeMo Guardrails FastAPI server expose?
**Answer:** `/v1/chat/completions`, compatible with the OpenAI API format.

**Q6:** Why was the Safety for Agentic AI Blueprint deprecated?
**Answer:** NVIDIA now recommends using NeMo Microservices (NeMo Guardrails, NeMo Auditor, Safety NIMs) as the current production tooling. The blueprint's lifecycle model remains useful as a conceptual framework.

**Q7:** When should you mask PII versus block the request entirely?
**Answer:** Mask when the query can still be processed without the actual PII values (replace with placeholders). Block when PII presence indicates a safety or compliance violation that should not proceed.

**Q8:** A guardrails configuration blocks 25% of legitimate user requests. What is this failure mode called, and how do you fix it?
**Answer:** Overly restrictive guardrails / high false positive rate. Fix by tuning detection thresholds, adding more examples to canonical forms in Colang, and measuring false positive rates systematically.

**Q9:** What is NeMo Auditor's role in the safety lifecycle?
**Answer:** Pre-deployment safety assessment. It runs audit jobs for vulnerability scanning and safety scoring, serving as a deployment gate between evaluation and production.

**Q10:** A Content Safety NIM goes down in production. What should happen?
**Answer:** The system should degrade to a more restrictive mode (e.g., block uncertain requests, fall back to heuristic detection), not bypass safety. Guardrail failure should never result in less safety.

---

## L. Mini Project

**Project: Defense-in-Depth Agent**

Build an agent with a complete safety stack: (1) Input rails: content safety + jailbreak detection + PII detection. (2) Dialog rails: scripted flow for one sensitive topic (e.g., account deletion). (3) Execution rails: parameter validation for at least one tool. (4) Output rails: content safety + PII redaction. (5) Topical rails: restrict to a specific business domain. Write all Colang 2.0 flows. Deploy via FastAPI server. Create a test suite of 20 inputs: 10 legitimate, 5 adversarial, 5 edge cases. Document guardrail trigger rates for each input category.

---

## M. How This May Appear on the Exam

1. **Rail type selection:** Given a specific safety scenario (e.g., "prevent SQL injection through a database tool"), identify which rail type addresses it (execution rails).

2. **Colang 2.0 interpretation:** Given a Colang code snippet, explain what safety behavior it implements and identify any gaps.

3. **Safety architecture design:** Given an agent description, design a guardrails configuration specifying which rail types are needed and why.

4. **Lifecycle mapping:** Map NVIDIA tools (NeMo Auditor, NeMo Guardrails, Safety NIMs, NAT middleware) to the correct stage of the 4-stage safety lifecycle.

5. **Failure analysis:** Given a safety breach scenario, identify which defensive layer was missing or misconfigured and prescribe the fix.

---

## N. Checklist for Mastery

- [ ] I can describe the 4-stage safety lifecycle and map current NVIDIA tools to each stage.
- [ ] I can explain why the Safety for Agentic AI Blueprint was deprecated and what replaced it.
- [ ] I can configure NeMo Guardrails with all five rail types in config.yml.
- [ ] I can write Colang 2.0 flows for content safety, topic control, and execution validation.
- [ ] I can explain the difference between Colang 1.0 and 2.0 and why new projects should use 2.0.
- [ ] I can deploy NeMo Guardrails via Python API, CLI, and FastAPI server.
- [ ] I can integrate Content Safety NIM, Jailbreak Detection NIM, and Topic Control NIM into a guardrails configuration.
- [ ] I can configure PII detection with at least one provider (Presidio, GLiNER, PrivateAI, or AutoAlign).
- [ ] I can explain direct vs. indirect prompt injection and the distinct defenses for each.
- [ ] I can configure NAT defense middleware and red-teaming middleware.
- [ ] I can use NeMo Auditor for pre-deployment safety assessment.
- [ ] I can design a defense-in-depth architecture where no single guardrail failure compromises the system.
- [ ] I can measure and optimize guardrail latency impact using the NAT Profiler.
- [ ] I can distinguish when to mask PII versus block requests entirely based on compliance requirements.
