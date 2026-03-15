# Module 6: Agent Development — Orchestration with NeMo Agent Toolkit

**Primary Exam Domains:** Agent Development (15%), NVIDIA Platform Implementation (7%)
**NVIDIA Tools:** NAT YAML configuration format, nat.builder workflow construction, middleware (caching, defense, logging, red teaming, dynamic dispatch), plugins (LangChain, LlamaIndex, CrewAI, AutoGen, Semantic Kernel, Agno, Strands, Google ADK), Interactive Workflows, nat.authentication (API Key, OAuth2, Bearer, HTTP Basic, MCP service accounts)

---

## A. Module Title

**Agent Development — Orchestration with NeMo Agent Toolkit**

---

## B. Why This Module Matters in Real Systems

Building an individual agent is a solved problem. Orchestrating agents — connecting them to tools, chaining them into workflows, securing them, observing them, and deploying them reliably — is where production systems live and die. A single agent answering questions is a demo. A workflow that ingests a customer request, routes it to the right agent, applies safety checks, logs every decision for audit, caches expensive operations, and handles authentication across multiple services — that is a production system. Orchestration is the layer that turns agent capabilities into operational services.

NeMo Agent Toolkit (NAT) provides two orchestration interfaces: declarative YAML configuration for standard workflows and the programmatic `nat.builder` API for complex or dynamic workflows. Both share the same runtime but serve different development patterns. YAML is readable, diffable, and deployable as configuration; the Python API is flexible, composable, and testable with standard tooling. Production teams typically use YAML for deployment configuration and the Python API for testing and custom middleware development.

The middleware system deserves particular attention. Middleware in NAT intercepts every request and response in the workflow, enabling cross-cutting concerns (caching, safety, logging, adversarial testing) without modifying agent logic. This is the same architectural pattern as HTTP middleware in web frameworks, applied to agent interactions. Understanding how middleware chains compose — and how ordering affects behavior — is essential for building secure, observable, and performant agentic systems. The NCP-AAI exam tests this understanding directly.

---

## C. Learning Objectives

By the end of this module, the student will be able to:

1. Write a complete NAT YAML configuration file from scratch, specifying agents, LLM providers, functions, memory, middleware, and observability settings.
2. Explain when to use YAML configuration versus `nat.builder` programmatic construction and justify the choice for a given scenario.
3. Describe the middleware chain execution model and correctly order middleware for a production deployment.
4. Configure caching, defense, logging, and red teaming middleware with appropriate parameters.
5. Integrate external framework tools (LangChain, LlamaIndex, CrewAI) into a NAT workflow using the plugin system.
6. Implement an Interactive Workflow using bidirectional WebSocket communication.
7. Configure multi-provider authentication (API Key, OAuth2, Bearer, HTTP Basic, MCP service accounts) in NAT.
8. Write deterministic unit tests for agent workflows using `nat_test_llm`.

---

## D. Required Concepts

Before starting this module, students must understand:

- **Module 5 content**: NAT functions, function groups, tool design, prompt engineering. Module 6 builds directly on Module 5.
- **YAML syntax**: Indentation-based structure, mappings, sequences, anchors/aliases. NAT configuration is entirely YAML-driven.
- **Middleware pattern**: The concept of intercepting a request, optionally modifying it, passing it to the next handler, and optionally modifying the response. Familiarity with middleware in web frameworks (Express.js, Django, FastAPI) is helpful.
- **OAuth2 basics**: Authorization code flow, client credentials, access tokens, refresh tokens. Required for understanding enterprise authentication configuration.
- **WebSocket fundamentals**: Persistent bidirectional connections vs request-response HTTP. Required for Interactive Workflows.

---

## E. Core Lesson Content

### NAT YAML Configuration Deep Dive [BEGINNER]

NAT workflows are defined declaratively in YAML. Every aspect of the agent system — which LLM to use, which tools to expose, how to handle memory, what middleware to apply — is specified in a single configuration file.

Below is a complete annotated YAML configuration. Every field is explained.

```yaml
# NAT Workflow Configuration — Complete Example
# Note: Field names and structure inferred from NAT documentation patterns.
# Verify exact syntax against current NAT SDK documentation.

workflow:
  name: "customer_support_v2"
  description: "Multi-turn customer support agent with order management tools"
  version: "2.1.0"

# LLM provider configuration
llm:
  provider: "nim"                          # NVIDIA NIM inference endpoint
  model: "meta/llama-3.3-70b-instruct"    # Model identifier
  endpoint: "https://integrate.api.nvidia.com/v1"
  temperature: 0.1                         # Low temperature for factual tasks
  max_tokens: 2048                         # Max response length
  top_p: 0.9                               # Nucleus sampling threshold
  api_key_env: "NVIDIA_API_KEY"            # Environment variable for API key

# Agent definition
agent:
  type: "react"                            # ReAct reasoning-action loop
  system_prompt: |
    You are a customer support agent for TechCorp.
    You have access to order tracking and refund tools.
    Always verify the customer's identity before taking action.
    If unsure, ask for clarification. Do not guess.
    For refunds over $500, escalate to human support.
  max_iterations: 10                       # Maximum reasoning-action cycles
  stop_conditions:                         # When to stop the loop
    - "task_complete"
    - "escalation_requested"
    - "max_iterations_reached"

# Tool/function registration
functions:
  - name: "order_lookup"
    source: "tools/order_tools.py"         # Local Python file
    group: "order_management"
  - name: "process_refund"
    source: "tools/order_tools.py"
    group: "order_management"
  - builtin: "datetime"                    # NAT built-in tool
  - builtin: "memory"                      # Persistent memory across sessions
  - mcp_server:                            # Remote tools via MCP
      url: "https://tools.internal.com/mcp"
      auth: "bearer"
      token_env: "MCP_SERVER_TOKEN"

# Memory backend
memory:
  backend: "redis"                         # Redis for production persistence
  host: "redis.internal.com"
  port: 6379
  ttl: 86400                               # 24-hour TTL for session memory
  max_history: 50                          # Maximum conversation turns to retain

# Middleware chain (executed in order)
middleware:
  - type: "logging"                        # Audit logging — first in chain
    config:
      level: "INFO"
      destination: "stdout"
      include_tool_calls: true
      include_llm_responses: true

  - type: "defense"                        # Safety checks — before LLM call
    config:
      input_guardrail: "nemo_guardrails"
      output_guardrail: "nemo_guardrails"
      block_pii: true
      max_input_length: 4096

  - type: "caching"                        # Cache expensive operations
    config:
      backend: "redis"
      ttl: 3600
      cache_tool_results: true
      cache_llm_responses: false           # Don't cache LLM — context-dependent

  - type: "dynamic_dispatch"               # Route to different agents based on intent
    config:
      classifier: "intent_classifier_v1"
      routes:
        order_inquiry: "order_agent"
        technical_support: "tech_agent"
        billing: "billing_agent"

# Observability
observability:
  exporter: "opentelemetry"
  endpoint: "http://otel-collector:4317"
  service_name: "customer_support_agent"
  trace_tool_calls: true
  trace_llm_latency: true
  metrics:
    - "task_success_rate"
    - "tool_call_count"
    - "average_latency"
```

**Key configuration sections:**

- **`llm`**: Specifies the inference provider. NAT supports NIM endpoints, OpenAI-compatible APIs, and local model servers. The `api_key_env` pattern keeps secrets out of configuration files.
- **`agent`**: Defines the reasoning strategy (ReAct is the most common), the system prompt, and stopping conditions. `max_iterations` prevents infinite loops.
- **`functions`**: Mixes local tools, built-in tools, and remote MCP tools in a single registry.
- **`memory`**: Configures where and how conversation history and persistent state are stored.
- **`middleware`**: Ordered list of middleware. Order matters — logging should come first (to capture everything), defense before the LLM (to block bad inputs).
- **`observability`**: OpenTelemetry integration for tracing, metrics, and monitoring.

### nat.builder: Programmatic Workflow Construction [INTERMEDIATE]

For workflows that cannot be expressed declaratively — conditional tool registration, dynamic middleware, runtime configuration changes — `nat.builder` provides a Python API.

```python
# Programmatic workflow construction with nat.builder
# Inferred API — verify against NAT SDK docs.

from nat_sdk import builder, middleware, agents

workflow = (
    builder.Workflow("dynamic_support")
    .with_llm(
        provider="nim",
        model="meta/llama-3.3-70b-instruct",
        temperature=0.1
    )
    .with_agent(
        type="react",
        system_prompt=load_prompt("prompts/support_v2.txt"),
        max_iterations=10
    )
    .with_functions([
        "tools/order_tools.py",
        "tools/billing_tools.py",
    ])
    .with_builtin("datetime")
    .with_builtin("memory")
    .with_middleware([
        middleware.Logging(level="INFO"),
        middleware.Defense(guardrail="nemo_guardrails"),
        middleware.Caching(backend="redis", ttl=3600),
    ])
    .with_observability(exporter="opentelemetry")
    .build()
)

# Run the workflow
result = workflow.run("I need to return my order #12345")
```

**When to use YAML vs Python API:**

| Criterion | YAML | nat.builder |
|---|---|---|
| Static workflows | Preferred | Works but verbose |
| Dynamic tool registration | Not possible | Preferred |
| CI/CD deployment | Easy (config-as-code) | Requires packaging |
| Testing | Needs YAML loader | Standard pytest |
| Version control | Clean diffs | Code diffs |
| Non-developer configuration | Accessible | Requires Python |

In practice, most teams use YAML for production deployment configuration and `nat.builder` for integration tests and prototyping.

### Middleware Chains [INTERMEDIATE]

Middleware intercepts every interaction in the workflow. The chain executes in order: each middleware sees the request, optionally modifies it, passes it to the next middleware (or the agent), and then processes the response on the way back.

```
NAT Middleware Chain — Request/Response Flow

  User Request
       │
       ▼
  ┌─────────────────┐
  │  1. LOGGING     │  ──> Records incoming request
  │     Middleware   │
  └────────┬────────┘
           ▼
  ┌─────────────────┐
  │  2. DEFENSE     │  ──> Checks input safety, PII, length
  │     Middleware   │      Can BLOCK request (returns error)
  └────────┬────────┘
           ▼
  ┌─────────────────┐
  │  3. CACHING     │  ──> Checks cache for matching request
  │     Middleware   │      If HIT: returns cached response (skips agent)
  └────────┬────────┘      If MISS: continues to agent
           ▼
  ┌─────────────────┐
  │  4. DYNAMIC     │  ──> Routes to appropriate agent
  │     DISPATCH    │      based on intent classification
  └────────┬────────┘
           ▼
  ┌═════════════════┐
  ║  AGENT CORE     ║  ──> ReAct loop: Reason → Tool Call → Observe → Repeat
  ║  (LLM + Tools)  ║
  └════════┬════════┘
           │
           ▼  (Response flows back up through middleware)
  ┌─────────────────┐
  │  4. DYNAMIC     │  ──> (pass-through on response)
  │     DISPATCH    │
  └────────┬────────┘
           ▼
  ┌─────────────────┐
  │  3. CACHING     │  ──> Stores response in cache (if cacheable)
  │     Middleware   │
  └────────┬────────┘
           ▼
  ┌─────────────────┐
  │  2. DEFENSE     │  ──> Checks output safety, removes PII
  │     Middleware   │      Can BLOCK response (returns safe alternative)
  └────────┬────────┘
           ▼
  ┌─────────────────┐
  │  1. LOGGING     │  ──> Records outgoing response
  │     Middleware   │
  └────────┬────────┘
           ▼
      User Response
```

**Middleware ordering matters.** If defense middleware comes after caching, a cached response bypasses safety checks. If logging comes after defense, blocked requests are not logged. The recommended order is:

1. **Logging** (captures everything, including blocked requests)
2. **Defense** (blocks unsafe inputs before they reach expensive processing)
3. **Caching** (avoids redundant processing for safe, validated requests)
4. **Red teaming** (applied in testing/staging, often disabled in production)
5. **Dynamic dispatch** (routing, closest to the agent core)

**Individual middleware details:**

**Caching middleware**: Caches tool call results (expensive API calls, database queries) to reduce latency and cost. Configurable TTL. Should cache tool results but generally not LLM responses (which are context-dependent). Cache key is typically the tool name + hashed parameters.

**Defense middleware**: Integrates with NeMo Guardrails for input/output safety filtering. Blocks prompt injection attempts, PII leakage, off-topic requests, and unsafe content. Can be configured with different guardrail profiles for different sensitivity levels.

**Logging middleware**: Records all requests, responses, tool calls, and LLM interactions. Essential for debugging and compliance. Supports structured logging to stdout, files, or external log aggregation systems.

**Red teaming middleware**: Runs adversarial test inputs against the agent to identify vulnerabilities. Typically enabled in staging/testing pipelines, not in production. Tests for prompt injection susceptibility, guardrail bypass attempts, and tool misuse patterns.

**Dynamic function dispatch**: Routes requests to different agents or function sets based on runtime conditions (user intent, request metadata, load balancing). Enables a single endpoint to serve multiple specialized agents.

### NAT Plugin System [ADVANCED]

> **Install note**: LangChain integration requires `pip install nvidia-nat[langchain] langchain-nvidia-ai-endpoints`. If LangChain plugin imports fail, verify the extra name in current NAT docs.

NAT's plugin system bridges external agent frameworks, allowing you to use tools, chains, retrievers, and agents from other ecosystems within NAT workflows.

**Supported frameworks:**
- **LangChain**: Use LangChain tools, chains, and retrievers. NAT wraps LangChain tools as NAT functions.
- **LlamaIndex**: Use LlamaIndex retrievers and query engines. Particularly useful for RAG integration (complements Module 4).
- **CrewAI**: Import CrewAI crew definitions as NAT sub-workflows. Useful for multi-agent collaboration patterns.
- **AutoGen**: Import AutoGen agent configurations. Conversation patterns map to NAT interactive workflows.
- **Semantic Kernel**: Use Semantic Kernel skills as NAT functions. Plugin conversion handles type mapping.
- **Agno, Strands, Google ADK**: Additional framework integrations with varying levels of support.

**How plugins work:**

Each plugin provides:
- **LLM wrapper**: Adapts the framework's LLM interface to NAT's provider configuration, so the framework uses NAT's configured LLM.
- **Tool conversion**: Converts framework-specific tool definitions to NAT functions (description, parameters, return types).
- **Callback handlers**: Maps framework events (tool calls, LLM completions) to NAT's middleware and observability pipeline.
- **Parser adapters**: Converts framework-specific output formats to NAT's response format.

```python
# Example: Using a LangChain tool within NAT
# Inferred API — verify against NAT SDK docs.

from nat_sdk import builder
from nat_sdk.plugins import langchain_adapter

# Import a LangChain tool
from langchain_community.tools import WikipediaQueryRun
wiki_tool = WikipediaQueryRun()

# Convert to NAT function
nat_wiki_function = langchain_adapter.convert_tool(wiki_tool)

# Use in NAT workflow
workflow = (
    builder.Workflow("research_agent")
    .with_llm(provider="nim", model="meta/llama-3.3-70b-instruct")
    .with_functions([nat_wiki_function])
    .build()
)
```

**When to use plugins vs native NAT functions**: Use plugins when an existing framework has a mature tool implementation you want to reuse. Use native NAT functions when you need full control over the tool interface, performance is critical (plugins add conversion overhead), or the tool is simple enough that wrapping is more work than reimplementing.

### Interactive Workflows [INTERMEDIATE]

Standard workflows are request-response: user sends a message, agent processes it, returns a result. Interactive Workflows support bidirectional WebSocket communication for real-time human-agent interaction during task execution.

**Use cases:**
- The agent needs clarification mid-task and asks the user
- Long-running tasks with progress updates
- Collaborative workflows where the human and agent take turns
- Approval gates where the agent pauses for human authorization before a sensitive action (e.g., executing a large financial transaction)

```
Interactive Workflow — WebSocket Flow

  User                         NAT Server                    Agent
   │                              │                            │
   │── WebSocket Connect ────────>│                            │
   │                              │── Initialize Agent ───────>│
   │── "Check order #12345" ─────>│                            │
   │                              │── Forward to Agent ───────>│
   │                              │                            │── [Tool: order_lookup]
   │                              │                            │── Result: order found
   │<─ "I found order #12345.   ─┤<─────────────────────────── │
   │    Refund is $750, which    │   Agent requests approval   │
   │    exceeds $500. Approve?" ─┤                             │
   │                              │                            │── [Waiting for user]
   │── "Yes, approved" ─────────>│                            │
   │                              │── Forward approval ───────>│
   │                              │                            │── [Tool: process_refund]
   │<─ "Refund processed." ──────┤<────────────────────────────│
   │                              │                            │
```

Interactive mode is configured in YAML:

```yaml
workflow:
  name: "interactive_support"
  interactive: true
  websocket:
    path: "/ws/support"
    ping_interval: 30          # Keep-alive ping (seconds)
    max_idle: 300              # Disconnect after 5 minutes idle
```

### Authentication [ADVANCED]

Production agent systems must authenticate with multiple services: the LLM provider, tool APIs, MCP servers, databases, and external services. NAT provides a multi-provider authentication architecture.

**Supported auth methods:**

| Method | Use Case | Configuration |
|---|---|---|
| **API Key** | Simple service auth (NIM, third-party APIs) | Key in env variable |
| **OAuth2** | Enterprise SSO, delegated authorization | Client ID/secret, token endpoint, scopes |
| **Bearer Token** | Service-to-service with pre-issued tokens | Token in env variable or vault |
| **HTTP Basic** | Legacy system integration | Username/password (use only over TLS) |
| **MCP Service Account** | MCP server authentication | Service account credentials |

```yaml
# Authentication configuration example
authentication:
  providers:
    nvidia_nim:
      type: "api_key"
      key_env: "NVIDIA_API_KEY"            # Reads from environment variable

    enterprise_sso:
      type: "oauth2"
      client_id_env: "OAUTH_CLIENT_ID"
      client_secret_env: "OAUTH_CLIENT_SECRET"
      token_endpoint: "https://auth.corp.com/oauth2/token"
      scopes: ["agent.read", "agent.write"]

    mcp_tools_server:
      type: "mcp_service_account"
      account_id_env: "MCP_ACCOUNT_ID"
      secret_env: "MCP_ACCOUNT_SECRET"
```

**Secure token storage**: NAT uses environment variable references (`_env` suffix) throughout configuration to keep secrets out of YAML files. For production deployments, environment variables are populated from secret management systems (Vault, AWS Secrets Manager, Kubernetes secrets). NAT never logs authentication tokens; the logging middleware redacts them automatically.

### NAT Cron Integration [INTERMEDIATE]

Agents are not always reactive. Some workflows run on schedules: daily report generation, periodic data validation, scheduled notifications.

NAT cron integration allows scheduling workflow execution:

```yaml
cron:
  - schedule: "0 8 * * MON-FRI"          # 8 AM weekdays
    workflow: "daily_report_generator"
    input: "Generate the daily sales report for yesterday"
  - schedule: "0 */4 * * *"              # Every 4 hours
    workflow: "data_quality_checker"
    input: "Run data quality checks on the inventory database"
```

This is particularly useful for monitoring and maintenance agents that run without human initiation.

### Testing with nat_test_llm [ADVANCED]

Testing agent workflows is challenging because LLM responses are non-deterministic. NAT provides `nat_test_llm`, a mock LLM that returns deterministic, pre-configured responses. This enables:

- **Unit tests**: Test that a specific input triggers the correct tool call sequence.
- **Integration tests**: Test the full middleware chain with predictable LLM behavior.
- **Regression tests**: Ensure that configuration changes don't break existing behavior.

```python
# Testing with nat_test_llm
# Inferred API — verify against NAT SDK docs.

from nat_sdk.testing import nat_test_llm, WorkflowTestHarness

# Configure mock LLM with deterministic responses
mock_llm = nat_test_llm(responses=[
    # First LLM call: agent decides to look up order
    {"tool_call": {"name": "order_lookup", "args": {"order_id": "12345"}}},
    # Second LLM call: agent generates response from tool result
    {"text": "Your order #12345 shipped on March 10th and arrives March 15th."},
])

# Build workflow with mock LLM
harness = WorkflowTestHarness(
    config="configs/support_workflow.yaml",
    llm_override=mock_llm
)

# Run test
result = harness.run("Where is my order 12345?")

# Assertions
assert result.tool_calls[0].name == "order_lookup"
assert result.tool_calls[0].args == {"order_id": "12345"}
assert "March 15" in result.final_response
assert result.middleware_logs["defense"].blocked == False
```

**Testing patterns:**
- **Happy path**: Configure mock to follow the expected tool call sequence. Verify correct tools are called with correct arguments.
- **Error handling**: Configure mock tool to return an error. Verify the agent retries or escalates as specified in the system prompt.
- **Middleware testing**: Inject an unsafe input. Verify defense middleware blocks it before it reaches the agent.
- **Performance testing**: Run many test cases and measure middleware overhead, tool call latency, and end-to-end latency.

### The Full Development Lifecycle in NAT [INTERMEDIATE]

```
NAT Development Lifecycle

  1. DESIGN                2. IMPLEMENT             3. TEST
  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
  │ Write YAML       │    │ Write NAT        │    │ nat_test_llm     │
  │ configuration    │───>│ functions        │───>│ mock LLM tests   │
  │ (agents, LLM,   │    │ (tools, groups,  │    │ Unit tests       │
  │  middleware)     │    │  per-user)       │    │ Integration tests│
  └──────────────────┘    └──────────────────┘    └────────┬─────────┘
                                                           │
  6. MONITOR               5. DEPLOY                4. CONFIGURE
  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
  │ OpenTelemetry    │    │ Docker / K8s     │    │ Middleware chain  │
  │ traces & metrics │<───│ deployment       │<───│ Authentication   │
  │ Log aggregation  │    │ NIM endpoints    │    │ Observability    │
  │ Alert on errors  │    │ Scaling config   │    │ Parameter opt    │
  └──────────────────┘    └──────────────────┘    └──────────────────┘
```

---

## F. Terminology Box

| Term | Definition |
|---|---|
| **NAT YAML configuration** | Declarative file format for defining NAT workflows, including agents, LLMs, tools, middleware, memory, and observability. |
| **nat.builder** | Programmatic Python API for constructing NAT workflows when declarative YAML is insufficient. |
| **Middleware** | A processing layer that intercepts requests and responses, enabling cross-cutting concerns (logging, security, caching) without modifying agent logic. |
| **Middleware chain** | An ordered sequence of middleware components. Order determines processing priority. |
| **Defense middleware** | Middleware that filters unsafe inputs and outputs, integrating with NeMo Guardrails. |
| **Dynamic dispatch** | Middleware that routes requests to different agents or function sets based on runtime conditions. |
| **Red teaming middleware** | Middleware that runs adversarial inputs against the agent to identify vulnerabilities. Used in testing. |
| **Plugin** | An adapter that converts tools, agents, or workflows from external frameworks (LangChain, LlamaIndex, etc.) into NAT-compatible components. |
| **Interactive Workflow** | A NAT workflow mode using bidirectional WebSocket communication for real-time human-agent interaction. |
| **nat_test_llm** | A mock LLM provided by NAT for deterministic testing of agent workflows. |
| **MCP service account** | An authentication method for NAT to authenticate with MCP servers using service-level credentials. |
| **Cron integration** | NAT's ability to schedule workflow execution at specified intervals using cron syntax. |

---

## G. Common Misconceptions

1. **"YAML configuration and nat.builder produce different runtime behavior."** They produce identical runtime behavior. YAML is parsed into the same internal representation that `nat.builder` constructs. The choice is purely about development workflow preference, not runtime differences.

2. **"Middleware order doesn't matter as long as all middleware is present."** Middleware order is critical. Defense after caching means cached responses bypass safety checks. Logging after defense means blocked requests are not logged. The order defines the security and observability guarantees of the system.

3. **"Plugins give you the full power of the external framework within NAT."** Plugins adapt specific components (tools, retrievers, parsers) but do not replicate the full runtime of the external framework. A LangChain agent running inside NAT is not the same as a LangChain agent running natively — the orchestration, middleware, and memory are handled by NAT.

4. **"Interactive Workflows are just chat."** Interactive Workflows support structured interactions: approval gates, multi-step forms, progress updates, and collaborative decision-making. They are designed for task-oriented human-agent collaboration, not open-ended conversation.

5. **"You can test agents adequately by running them with a real LLM and checking outputs."** Real LLM responses are non-deterministic. A test that passes today may fail tomorrow with the same input. `nat_test_llm` provides deterministic behavior for assertions. Use real LLMs for evaluation (measuring quality), not for testing (verifying correctness).

6. **"Authentication configuration should be in the YAML file for simplicity."** Secrets (API keys, tokens, passwords) must never be stored in YAML configuration files. NAT uses `_env` suffix references to read from environment variables, which are populated from secret management systems at deployment time.

---

## H. Failure Modes / Anti-Patterns

1. **Defense middleware after caching**: Cached responses bypass safety checks. A response that was safe when cached may be served to a different user in a different context where it is unsafe. Fix: always place defense middleware before caching in the chain.

2. **No max_iterations on ReAct agents**: The agent enters an infinite reasoning loop, calling tools repeatedly without converging on an answer. Costs accumulate rapidly. Fix: set `max_iterations` to a reasonable limit (5-15) and define explicit stop conditions.

3. **Caching LLM responses**: LLM responses depend on the full conversation context. Caching a response to "What's the status?" returns the wrong answer when the context changes. Cache tool results (which are stateless) but not LLM responses (which are context-dependent). Fix: set `cache_llm_responses: false`.

4. **Monolithic YAML with no environment separation**: The same configuration file is used for development, staging, and production, with secrets and endpoints hardcoded. Fix: use environment variable references and separate overlay files for each environment.

5. **Using plugins without understanding conversion overhead**: Each plugin call involves type conversion, schema adaptation, and callback translation. For latency-sensitive paths, this overhead matters. Fix: benchmark plugin-wrapped tools against native NAT functions. Use native implementations for hot paths.

6. **Testing only happy paths**: Tests verify that the agent handles correct inputs correctly but never test error handling, tool failures, ambiguous inputs, or adversarial inputs. Fix: write test cases for every failure mode documented in the system prompt's behavioral policies.

7. **Ignoring WebSocket connection management in Interactive Workflows**: Not handling disconnections, reconnections, or idle timeouts. Users lose mid-conversation state when connections drop. Fix: configure `ping_interval` and `max_idle`, implement reconnection logic on the client side, and persist conversation state in the memory backend.

---

## I. Hands-On Lab

**Lab: Build a Complete NAT Workflow with Middleware**

Write a YAML configuration for a customer support agent. Implement three custom NAT functions (order lookup, refund processing, FAQ search). Configure the full middleware chain: logging, defense (using NeMo Guardrails or a mock guardrail), caching (for FAQ search results only), and dynamic dispatch (route billing questions to a billing sub-agent). Deploy the workflow and test with 10 predefined scenarios. Write three unit tests using `nat_test_llm` covering happy path, tool error handling, and defense middleware blocking.

Full lab specification is in a separate file.

---

## J. Stretch Lab

**Stretch: Multi-Framework Plugin Integration**

Build a NAT workflow that uses tools from three different frameworks: a LangChain web search tool, a LlamaIndex document retriever, and a custom NAT function for database queries. Configure the workflow with authentication to all three tool sources. Write integration tests that verify the agent correctly selects tools from different frameworks based on query type. Measure and report the latency overhead of each plugin versus a native NAT function equivalent.

---

## K. Review Quiz

**1.** In a NAT YAML configuration, what does the `api_key_env` field do?
**Answer:** It specifies the name of an environment variable that contains the API key. NAT reads the key from the environment at runtime rather than storing it in the YAML file. This keeps secrets out of version-controlled configuration.

**2.** What happens if defense middleware is placed after caching middleware in the chain?
**Answer:** Cached responses bypass defense checks entirely. A request that hits the cache returns directly without being checked for safety, PII, or policy compliance. This is a security anti-pattern. Defense should always precede caching.

**3.** When should you use `nat.builder` instead of YAML configuration?
**Answer:** When the workflow requires dynamic behavior that cannot be expressed declaratively — such as conditional tool registration based on runtime data, dynamic middleware configuration, or programmatic workflow composition. Also preferred for writing tests using standard Python testing frameworks.

**4.** What is the purpose of `nat_test_llm`?
**Answer:** It is a mock LLM that returns pre-configured, deterministic responses. It enables writing repeatable unit and integration tests for agent workflows by removing LLM non-determinism. Tests can assert that specific tool calls are made with specific arguments.

**5.** How does NAT's plugin system convert a LangChain tool for use in a NAT workflow?
**Answer:** The plugin adapter converts the LangChain tool's description, parameter schema, and return type into NAT function format. It also wraps the execution call so that the LangChain tool uses NAT's configured LLM and the results flow through NAT's middleware pipeline.

**6.** What is the difference between caching tool results and caching LLM responses?
**Answer:** Tool results are typically stateless (same inputs produce same outputs) and safe to cache. LLM responses depend on the full conversation context and vary with context changes. Caching LLM responses can return contextually incorrect answers. Best practice: cache tool results, not LLM responses.

**7.** Name three authentication methods supported by NAT and give a use case for each.
**Answer:** API Key — simple service authentication (e.g., NIM endpoints); OAuth2 — enterprise SSO with delegated authorization (e.g., corporate identity providers); MCP Service Account — authenticating with remote MCP tool servers using service-level credentials.

**8.** What is dynamic dispatch middleware and when is it used?
**Answer:** Dynamic dispatch routes requests to different agents or function sets based on runtime conditions (typically an intent classifier). It is used when a single endpoint must serve multiple specialized agents — for example, routing billing questions to a billing agent and technical questions to a support agent.

**9.** Why is setting `max_iterations` on a ReAct agent important?
**Answer:** Without a limit, the agent can enter infinite reasoning loops, repeatedly calling tools without converging on an answer. This wastes compute resources and accumulates API costs. `max_iterations` forces termination after a fixed number of reasoning-action cycles.

**10.** A team deploys an Interactive Workflow but users report losing conversation context after brief network interruptions. What is the most likely cause?
**Answer:** The WebSocket connection drops during the interruption and is not properly reconnected. If conversation state is stored only in the WebSocket session (not persisted to the memory backend), it is lost on disconnect. Fix: configure the memory backend (e.g., Redis) for state persistence and implement client-side reconnection logic.

---

## L. Mini Project

**Project: Production-Ready NAT Workflow with Full Middleware Stack**

Design and implement a complete NAT workflow for a domain of your choice (e.g., IT helpdesk, e-commerce support, research assistant). Requirements: (1) At least 5 custom NAT functions organized into 2+ function groups. (2) Full middleware chain: logging, defense, caching, and at least one custom middleware. (3) Multi-provider authentication (at least 2 auth methods). (4) 10+ unit tests using `nat_test_llm` covering happy paths, error handling, and middleware behavior. (5) YAML configuration file with inline documentation explaining every field. (6) A one-page architecture document showing the middleware chain, tool registry, and authentication flow.

---

## M. How This May Appear on the Exam

1. **YAML configuration reading**: Given a NAT YAML configuration snippet, identify what is misconfigured (e.g., middleware in wrong order, missing auth, no max_iterations). Expect questions that test your ability to read and critique configuration.

2. **Middleware ordering**: "Place the following middleware in the correct order for a production deployment: caching, defense, logging, red teaming." Know the canonical ordering and the reasoning behind it.

3. **Plugin framework knowledge**: "Which NAT feature allows using LangChain tools within a NAT workflow?" Know that the plugin system provides framework adapters. May test specific frameworks supported.

4. **Testing methodology**: "How do you write deterministic tests for an agent workflow that calls an LLM?" Know that `nat_test_llm` provides mock responses. May test understanding of what can and cannot be tested deterministically.

5. **Authentication architecture**: "A NAT workflow must authenticate with NIM for LLM inference, a corporate OAuth2 provider for user identity, and an MCP server for remote tools. How is this configured?" Know the multi-provider authentication model and the `_env` pattern for secrets.

---

## N. Checklist for Mastery

The student should be able to:

- [ ] Write a complete NAT YAML configuration file from scratch without referencing documentation
- [ ] Explain every field in a NAT YAML configuration to a colleague
- [ ] Build the same workflow using both YAML and `nat.builder` and explain when each is appropriate
- [ ] Order middleware correctly for a production deployment and justify the ordering
- [ ] Configure each middleware type (logging, defense, caching, red teaming, dynamic dispatch) with appropriate parameters
- [ ] Use the plugin system to integrate at least one external framework tool into a NAT workflow
- [ ] Implement an Interactive Workflow with WebSocket communication and approval gates
- [ ] Configure multi-provider authentication with secrets stored in environment variables
- [ ] Write unit tests with `nat_test_llm` for happy path, error handling, and middleware behavior
- [ ] Set up scheduled workflow execution using NAT cron integration
- [ ] Diagnose common configuration errors: wrong middleware order, missing max_iterations, cached LLM responses
- [ ] Describe the full NAT development lifecycle from design through monitoring
