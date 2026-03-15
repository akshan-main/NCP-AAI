# Module 5: Agent Development — Tools and Prompt Engineering

**Primary Exam Domain:** Agent Development (15%)
**NVIDIA Tools:** NAT Functions, Function Groups, Per-User Functions, MCP Client, tool registry, Parameter Optimization, built-in tools (code execution sandbox, datetime, document search, GitHub, memory, text-to-SQL/Vanna), Agent Hyperparameter Optimizer

---

## A. Module Title

**Agent Development — Tools and Prompt Engineering**

---

## B. Why This Module Matters in Real Systems

An LLM without tools is a text generator. An LLM with well-designed tools is an agent that can query databases, execute code, search documents, call APIs, and take actions in the real world. The quality of tool design — how tools are described, what parameters they accept, what contracts they enforce — directly determines whether an agent successfully completes tasks or fails in subtle, hard-to-debug ways. In production agentic systems, tool design is where most of the engineering effort goes, and where most of the reliability improvement comes from.

Prompt engineering for agents is not the same as prompt engineering for chatbots. Agent prompts must encode behavioral policies (when to use which tool, when to ask for clarification, when to stop), output format contracts (JSON schemas, structured responses), and safety boundaries (what the agent must not do). A poorly prompted agent will use tools at random, retry failed operations indefinitely, or produce outputs that downstream systems cannot parse. The difference between a 60% success rate and a 95% success rate often comes down to prompt quality, not model quality.

NVIDIA's NeMo Agent Toolkit (NAT) provides a structured framework for tool definition, registration, and management. It includes built-in tools for common operations, a plugin system for external tool ecosystems, MCP client support for remote tool access, and automated parameter optimization. Understanding these systems is essential for the Agent Development section of the NCP-AAI exam, which represents 15% of the total score.

---

## C. Learning Objectives

By the end of this module, the student will be able to:

1. Design agent tools with clear descriptions, typed parameters, and well-defined return value contracts.
2. Implement a custom NAT function with proper registration and input/output typing.
3. Organize tools into NAT Function Groups and explain when grouping improves agent behavior.
4. Configure NAT Per-User Functions for multi-tenant tool isolation in production.
5. Explain the MCP (Model Context Protocol) and configure NAT's MCP client to connect to remote tool servers.
6. Select appropriate built-in NAT tools for common tasks (code execution, document search, text-to-SQL).
7. Write agent system prompts that encode behavioral policies, output contracts, and safety boundaries.
8. Use NAT Parameter Optimization to automatically tune prompt phrasing, LLM selection, and inference parameters.

---

## D. Required Concepts

Before starting this module, students must understand:

- **Function calling in LLMs**: How modern LLMs are trained to emit structured function calls (tool use) and how the runtime executes those calls and returns results.
- **JSON Schema**: How parameter schemas define types, required fields, enums, and descriptions. The LLM uses these schemas to generate valid function call arguments.
- **Python type hints**: `typing` module basics — `str`, `int`, `List[str]`, `Optional[int]`, `Dict[str, Any]`. NAT uses type hints to generate schemas.
- **HTTP APIs**: REST basics — endpoints, methods, request/response bodies, status codes. Required for understanding MCP and remote tool access.
- **Basic prompt engineering**: System messages, user messages, few-shot examples. Module 3 covers foundational prompting.

---

## E. Core Lesson Content

### Tool Design Principles [BEGINNER]

A tool is only as useful as the LLM's ability to understand when and how to use it. The three elements that determine tool usability are:

**1. Tool description**: This is the most important field. The LLM reads the description to decide whether to call the tool. A vague description ("does stuff with data") leads to incorrect tool selection. A precise description ("Queries the PostgreSQL inventory database and returns current stock levels for specified product SKUs") tells the LLM exactly when this tool is appropriate.

**2. Parameter schema**: Each parameter needs a name, type, description, and whether it is required or optional. The descriptions matter — `product_id: str` tells the LLM nothing about format, but `product_id: str — The 8-digit alphanumeric product SKU, e.g., 'PRD-12345'` gives it enough to generate valid arguments.

**3. Return value contract**: What the tool returns, in what format, and what error conditions look like. If the tool returns JSON, document the schema. If it can fail, document the error format. The LLM needs to know what to expect so it can parse results and decide next steps.

```
Good Tool Design Checklist:

  [x] Description answers: "When should the agent use this tool?"
  [x] Each parameter has a type AND a human-readable description
  [x] Examples of valid parameter values are included
  [x] Return format is documented (JSON schema or description)
  [x] Error conditions are documented
  [x] The tool does ONE thing well (single responsibility)
```

### NAT Functions [BEGINNER]

NAT Functions are the primary mechanism for giving agents custom capabilities. A NAT function is a Python function that is registered with the toolkit and made available to the agent.

```python
# Example: Writing a custom NAT function
# Note: Exact decorator syntax is inferred from NAT documentation patterns.
# Verify against current NAT SDK documentation.

from nat_sdk import function, FunctionResult

@function(
    name="get_stock_price",
    description="Retrieves the current stock price for a given ticker symbol. "
                "Use this when the user asks about stock prices, market data, "
                "or financial quotes. Returns price in USD.",
)
def get_stock_price(
    ticker: str,  # Stock ticker symbol, e.g., 'NVDA', 'AAPL'
    include_change: bool = False  # Whether to include 24h price change
) -> FunctionResult:
    """Fetch current stock price from market data API."""
    # Implementation here
    price = fetch_price(ticker)
    result = {"ticker": ticker, "price": price, "currency": "USD"}
    if include_change:
        result["change_24h"] = fetch_change(ticker)
    return FunctionResult(data=result)
```

Key design points for NAT functions:
- **Type hints are mandatory**: NAT uses them to generate the JSON Schema that the LLM sees. Missing type hints mean the LLM has no schema guidance.
- **Docstrings supplement descriptions**: The function docstring and the `description` parameter both contribute to the LLM's understanding.
- **Return `FunctionResult`**: Structured return type that NAT can serialize and pass back to the LLM.
- **Error handling**: Functions should catch exceptions and return meaningful error messages rather than crashing. The LLM can often recover from a descriptive error but not from a raw traceback.

### NAT Function Groups [INTERMEDIATE]

Function Groups compose related tools into logical units. Instead of registering 20 individual functions, you group them by domain.

```python
# Example: Function group for database operations
# Inferred API pattern — verify against NAT SDK docs.

from nat_sdk import function_group

@function_group(
    name="inventory_db",
    description="Tools for querying and updating the product inventory database. "
                "Use these tools for any inventory-related questions.",
)
class InventoryTools:

    @function(name="search_products", description="Search products by name or category")
    def search_products(self, query: str, category: str = None) -> FunctionResult:
        ...

    @function(name="get_stock_level", description="Get current stock level for a product SKU")
    def get_stock_level(self, sku: str) -> FunctionResult:
        ...

    @function(name="update_stock", description="Update stock level after a transaction")
    def update_stock(self, sku: str, quantity_change: int, reason: str) -> FunctionResult:
        ...
```

**When to use groups vs individual functions:**
- Use groups when tools share a domain context (all database tools, all email tools, all file tools). The group description helps the LLM understand the domain before selecting a specific tool.
- Use individual functions for standalone capabilities that don't belong to a larger domain.
- Groups also enable batch enablement/disablement — you can activate the "inventory_db" group for warehouse agents and deactivate it for customer-facing agents.

### NAT Per-User Functions [INTERMEDIATE]

In multi-tenant systems, different users require different tool access. A finance team member should access financial reporting tools; an engineering team member should access deployment tools. Exposing all tools to all users creates security risks and confuses the agent with irrelevant options.

NAT Per-User Functions allow the tool set to vary based on user identity. The runtime resolves user context (from authentication tokens, session data, or external identity providers) and activates only the tools that user is authorized to use.

This is a production-critical feature. Without it, you either build separate agents per user group (expensive, hard to maintain) or expose a single over-permissioned agent to everyone (insecure, unreliable).

### NAT Built-in Tools [INTERMEDIATE]

NAT includes pre-built tools for common agent operations:

| Built-in Tool | Purpose | When to Use |
|---|---|---|
| **Code execution** | Runs Python code in a sandboxed environment (local or remote) | Data analysis, calculations, transformations |
| **Datetime** | Current time, timezone conversions, date arithmetic | Any time-sensitive queries |
| **Document search** | Searches indexed documents via RAG pipeline | Knowledge base queries (connects to Module 4) |
| **GitHub** | Repository operations — issues, PRs, code search | Developer-facing agents |
| **Memory** | Persistent key-value store for agent state across sessions | Agents that need to remember user preferences or prior context |
| **Text-to-SQL (Vanna)** | Converts natural language questions to SQL queries | Database-connected agents for non-technical users |

**Code execution sandboxing**: NAT supports both local and remote sandboxes. Local sandboxes use process isolation. Remote sandboxes execute code in a separate container, providing stronger isolation. For production systems handling untrusted user input, remote sandboxing is required. The sandbox prevents agents from executing arbitrary system commands, accessing the filesystem beyond designated paths, or making unauthorized network calls.

**Text-to-SQL via Vanna**: Vanna is an open-source framework that fine-tunes on your database schema to generate accurate SQL. NAT integrates Vanna as a built-in tool, allowing agents to answer data questions without the user writing SQL. The agent translates the question to SQL, executes it, and interprets the results.

### MCP (Model Context Protocol) [INTERMEDIATE]

MCP is a standard protocol for connecting LLM applications to external tool servers. It defines how a client (the agent) discovers tools, calls them, and processes results from a server (the tool provider).

**NAT's MCP client** connects to MCP-compatible servers, automatically discovering available tools and making them callable by the agent. This enables:
- Using tools hosted by third-party providers without writing custom integration code
- Accessing remote tools behind network boundaries via MCP server proxies
- Sharing tool implementations across multiple agent frameworks

```
MCP Architecture:

  ┌──────────────────┐         ┌──────────────────────┐
  │  NAT Agent       │         │  MCP Server          │
  │                  │         │                      │
  │  ┌────────────┐  │  JSON   │  ┌────────────────┐  │
  │  │ MCP Client │◄─┼────────►│  │ Tool Registry  │  │
  │  │            │  │  RPC    │  │                │  │
  │  │ - discover │  │         │  │ - list_tools() │  │
  │  │ - call     │  │         │  │ - call_tool()  │  │
  │  │ - parse    │  │         │  │ - get_schema() │  │
  │  └────────────┘  │         │  └────────────────┘  │
  └──────────────────┘         └──────────────────────┘
```

### NAT Tool Registry [INTERMEDIATE]

The tool registry is where all available tools — custom functions, function groups, built-in tools, and MCP-discovered tools — are managed. NAT supports multiple registry handlers:

- **Local file**: Tools defined in Python files in a local directory
- **PyPI**: Tools packaged as Python packages and installed from PyPI
- **REST**: Tools exposed as REST endpoints that conform to NAT's tool API

The registry provides discovery (what tools are available?), schema inspection (what parameters does a tool accept?), and lifecycle management (enabling/disabling tools at runtime).

### Prompt Engineering for Agents [INTERMEDIATE]

Agent prompt engineering differs from conversational prompt engineering in three critical ways:

**1. Behavioral policies**: The system prompt must encode when to use tools, when to ask for clarification, when to give up, and when to escalate. Without explicit policies, agents exhibit erratic behavior — sometimes asking for clarification, sometimes guessing, sometimes retrying failures indefinitely.

```
# Example: Behavioral policy in a system prompt

You are a customer support agent with access to order tracking and refund tools.

BEHAVIORAL RULES:
1. Always verify the customer's order ID before taking any action.
2. If the customer's request is ambiguous, ask ONE clarifying question. Do not guess.
3. If a tool call fails, retry ONCE. If it fails again, apologize and escalate to human support.
4. Never disclose internal system details, error messages, or tool names to the customer.
5. For refund requests over $500, collect the reason and escalate — do not process directly.
```

**2. Output format contracts**: When agents produce structured outputs consumed by downstream systems, the prompt must specify the exact format. Use JSON schemas in the prompt and provide examples.

**3. Chain-of-thought prompting for tool selection**: For complex tasks requiring multiple tool calls, instruct the agent to reason about which tool to use before calling it. This is especially important when the agent has many tools and must select the right sequence.

```
# Example: Chain-of-thought tool selection prompt

Before calling any tool, briefly state:
- What information you need
- Which tool will provide it
- What you will do with the result

Then call the tool.
```

### NAT Parameter Optimization [ADVANCED]

Manual prompt tuning is time-consuming and often leaves performance on the table. NAT Parameter Optimization automates this process.

**What can be optimized:**
- **LLM selection**: Which model performs best for this task (e.g., Llama 3.3 70B vs Nemotron Super 49B)
- **Temperature**: Higher for creative tasks, lower for factual tasks, but the optimal value varies
- **Max tokens**: Prevents over-generation or under-generation
- **Prompt phrasing**: Automated rephrasing of system prompts and few-shot examples
- **Tool descriptions**: Automated refinement of tool descriptions to improve selection accuracy

**How it works** (inferred from NAT documentation — verify exact API):
You define an evaluation dataset (inputs with expected outputs or a scoring function), specify which parameters to optimize, and run the optimizer. It systematically tests parameter combinations and reports the best configuration.

```python
# Conceptual example — verify against NAT SDK docs
from nat_sdk import ParameterOptimizer

optimizer = ParameterOptimizer(
    workflow="customer_support",
    eval_dataset="support_eval_100.jsonl",
    parameters_to_optimize=["llm_model", "temperature", "system_prompt"],
    scoring_function=task_success_rate,
    max_iterations=50
)

best_config = optimizer.run()
# Returns: {"llm_model": "llama-3.3-70b", "temperature": 0.1, "system_prompt": "..."}
```

### Agent Hyperparameter Optimizer [ADVANCED]

The Agent Hyperparameter Optimizer extends parameter optimization beyond individual parameters to the full agent configuration space. This includes:
- Middleware chain ordering
- Retrieval parameters (top-K, similarity threshold)
- Tool selection thresholds
- Retry policies
- Memory configuration

This is a system-level optimizer that treats the entire agent configuration as a search space and finds the combination that maximizes a given objective (task success rate, latency, cost, or a weighted combination).

**When to use**: After you have a working agent with reasonable baseline performance. Hyperparameter optimization refines a working system; it does not fix a broken one. Fix tool design, prompt quality, and retrieval pipeline first.

---

## F. Terminology Box

| Term | Definition |
|---|---|
| **NAT Function** | A Python function registered with NeMo Agent Toolkit, made available to agents as a callable tool. |
| **Function Group** | A logical collection of related NAT functions that share a domain context. |
| **Per-User Functions** | NAT feature that restricts tool availability based on authenticated user identity. |
| **MCP** | Model Context Protocol. A standard protocol for LLM applications to discover and call tools on remote servers. |
| **Tool Registry** | NAT's central catalog of available tools, supporting local, PyPI, and REST registry handlers. |
| **Parameter Optimization** | Automated tuning of agent configuration parameters (model, temperature, prompts) against an evaluation dataset. |
| **Few-shot prompting** | Including example input-output pairs in the prompt to demonstrate desired behavior. |
| **Chain-of-thought (CoT)** | Prompting technique that instructs the model to reason step-by-step before producing a final answer. |
| **Structured output** | Agent responses formatted as JSON, XML, or other machine-parseable formats for downstream consumption. |
| **Sandbox** | An isolated execution environment for running agent-generated code without risking the host system. |
| **Vanna** | An open-source text-to-SQL framework integrated as a NAT built-in tool. |

---

## G. Common Misconceptions

1. **"The LLM can figure out how to use a tool from the function name alone."** LLMs rely heavily on tool descriptions and parameter descriptions. A function named `process_data` with no description is nearly useless. A function named `pd` with a detailed description works fine. Always invest in descriptions, not clever naming.

2. **"More tools means a more capable agent."** Adding tools beyond what the agent needs increases the probability of incorrect tool selection. Each tool competes for the LLM's attention. An agent with 5 well-described tools will outperform one with 50 poorly described tools. Curate the tool set for the task.

3. **"Prompt engineering is just about being polite or adding 'please.'"** For agents, prompt engineering is systems engineering. It defines behavioral policies, error handling strategies, output formats, and safety boundaries. A production agent prompt is a specification document, not a conversation starter.

4. **"MCP replaces the need for custom tool implementations."** MCP provides a protocol for accessing remote tools, but someone must still implement those tools on the server side. MCP standardizes the interface; it does not generate tool implementations.

5. **"Parameter optimization will fix a poorly designed agent."** Optimization finds the best configuration within a search space. If the fundamental design is wrong (bad tool descriptions, missing tools, incorrect behavioral policies), no amount of temperature tuning will fix it. Fix design first, then optimize.

6. **"Code execution tools are too dangerous for production."** With proper sandboxing (remote container execution, filesystem restrictions, network isolation, resource limits, timeout enforcement), code execution tools are safely deployable. The risk comes from inadequate sandboxing, not from code execution itself.

---

## H. Failure Modes / Anti-Patterns

1. **Vague tool descriptions**: Description says "handles customer data" without specifying what operations are supported, what input formats are accepted, or what is returned. The LLM guesses wrong 40% of the time. Fix: write descriptions that answer "when should the agent use this tool?" with specifics.

2. **Missing parameter validation**: The tool accepts any string for a parameter that should be a constrained enum (e.g., `status` should be one of `["active", "inactive", "pending"]` but accepts anything). The LLM generates invalid values, the tool fails, the agent retries with different invalid values. Fix: use enums, regex patterns, and range constraints in parameter schemas.

3. **Tool does too many things**: A single tool that searches, filters, sorts, and exports data. The LLM must provide many parameters correctly for one call. Break it into focused tools: `search_products`, `filter_results`, `export_results`. Single-responsibility tools are more reliably used.

4. **No error context in tool responses**: Tool returns `{"error": true}` with no message. The LLM cannot diagnose the problem or communicate it to the user. Fix: return descriptive error messages like `{"error": true, "message": "Product SKU 'XYZ-999' not found in inventory database", "suggestion": "Verify the SKU format: expected pattern is 'PRD-NNNNN'"}`.

5. **System prompt doesn't specify failure behavior**: Agent retries a failing tool infinitely, or silently drops the task, or hallucinates an answer when the tool fails. Fix: explicitly state retry limits and escalation behavior in the system prompt.

6. **Over-optimizing on a narrow eval set**: Running parameter optimization on 10 examples and deploying the result. The optimized configuration overfits to those 10 examples and degrades on real traffic. Fix: use at least 100 diverse evaluation examples and hold out a test set.

---

## I. Hands-On Lab

**Lab: Build and Optimize a Multi-Tool Agent**

Build a NAT agent with three custom tools: a product catalog search tool, a stock level checker, and an order placement tool. Write a system prompt with behavioral policies. Test the agent on 15 predefined scenarios covering happy path, error cases, and ambiguous queries. Measure task success rate. Then run NAT Parameter Optimization to tune the prompt and temperature. Compare before/after success rates.

Full lab specification is in a separate file.

---

## J. Stretch Lab

**Stretch: MCP Integration and Dynamic Tool Discovery**

Set up an MCP server exposing a set of tools (e.g., weather API, calendar API). Configure NAT's MCP client to connect to the server. Build an agent that discovers available tools at runtime and uses them to answer user queries. Test with tool server restarts and new tool additions to verify dynamic discovery. Implement Per-User Functions so that different test users see different tool subsets.

---

## K. Review Quiz

**1.** What are the three elements that determine an agent tool's usability by an LLM?
**Answer:** Tool description, parameter schema (with types and descriptions), and return value contract. The description tells the LLM when to use the tool; the parameter schema tells it how to call it; the return contract tells it what to expect.

**2.** Why does NAT require type hints on function parameters?
**Answer:** NAT uses Python type hints to auto-generate the JSON Schema that the LLM receives. Without type hints, the LLM has no schema to guide its argument generation, leading to type errors and invalid calls.

**3.** When should you use NAT Function Groups instead of individual functions?
**Answer:** When tools share a domain context (e.g., all database operations, all email operations). Groups provide a domain-level description that helps the LLM understand the category before selecting a specific tool, and enable batch enablement/disablement for different agent roles.

**4.** What is the purpose of NAT Per-User Functions?
**Answer:** Multi-tenant tool isolation. Different users receive different tool sets based on their authenticated identity, preventing unauthorized access and reducing irrelevant tool clutter that degrades agent accuracy.

**5.** How does NAT's MCP client discover available tools on a remote server?
**Answer:** Through MCP's standard discovery protocol — the client queries the server's tool registry endpoint, which returns tool names, descriptions, and parameter schemas. Tools are then registered in NAT's tool registry and made available to the agent.

**6.** Name three NAT built-in tools and state when each is appropriate.
**Answer:** (Any three of) Code execution — for data analysis and computation; Document search — for knowledge base queries; Text-to-SQL via Vanna — for database queries from non-technical users; Memory — for persisting state across sessions; Datetime — for time-sensitive operations; GitHub — for developer workflow automation.

**7.** What is the difference between local and remote code execution sandboxes in NAT?
**Answer:** Local sandboxes use process-level isolation on the same machine. Remote sandboxes execute code in a separate container, providing stronger isolation (filesystem, network, resource limits). Remote sandboxing is required for production systems handling untrusted input.

**8.** Why is specifying retry and escalation behavior in the system prompt important?
**Answer:** Without explicit failure behavior, the agent defaults to unpredictable strategies — infinite retries, silent failures, or hallucinated answers. Specifying "retry once, then escalate to human support" creates deterministic, user-friendly failure handling.

**9.** What does NAT Parameter Optimization require as inputs?
**Answer:** An evaluation dataset (inputs with expected outputs or a scoring function), a specification of which parameters to optimize (model, temperature, prompt, etc.), and a scoring function that measures task success.

**10.** A team adds 30 tools to their agent and finds that task success rate drops from 90% to 65%. What is the most likely cause and fix?
**Answer:** Tool selection confusion — the LLM has too many options and selects the wrong tool more often. Fix: reduce the active tool set to only what is needed for the current task (using Function Groups and Per-User Functions), and improve tool descriptions to make each tool's purpose unambiguous.

---

## L. Mini Project

**Project: Tool Design Audit and Optimization Report**

Take an existing set of 10+ poorly designed tool definitions (provided as a JSON file with vague descriptions, missing parameter docs, overlapping functionality). Redesign them: rewrite descriptions, add parameter documentation, split multi-purpose tools, group related tools into Function Groups. Build an agent using both the original and redesigned tools. Run the same 20 test scenarios against both. Produce a comparison report showing task success rate, average tool calls per task, and error rate. Include specific examples of failures caused by poor tool design and how the redesign fixed them.

---

## M. How This May Appear on the Exam

1. **Tool design evaluation**: Given a tool definition (description, parameters, return type), identify what is wrong with it and how to fix it. Expect questions testing whether descriptions are specific enough and whether parameter schemas are properly typed.

2. **NAT feature selection**: "A company needs different agents for different departments, but wants to maintain a single codebase. Which NAT feature enables this?" Expect questions about Function Groups and Per-User Functions.

3. **MCP understanding**: "What protocol does NAT use to connect to remote tool servers?" and "What does the MCP client discover from the server?" Know that MCP provides tool discovery, schema retrieval, and remote invocation.

4. **Prompt engineering for reliability**: Given a scenario where an agent behaves incorrectly (e.g., retries indefinitely, uses wrong tool), identify the prompt engineering fix. Expect questions about behavioral policies in system prompts.

5. **Parameter optimization scope**: "Which of the following can NAT Parameter Optimization tune?" Know the list: LLM model, temperature, max_tokens, prompt phrasing, tool descriptions.

---

## N. Checklist for Mastery

The student should be able to:

- [ ] Write a NAT function with proper description, typed parameters, and FunctionResult return
- [ ] Organize related functions into a Function Group with a domain-level description
- [ ] Explain Per-User Functions and describe a production scenario where they are necessary
- [ ] Describe MCP and draw the client-server interaction for tool discovery and invocation
- [ ] List all NAT built-in tools and state the use case for each
- [ ] Write an agent system prompt with explicit behavioral policies, output format, and failure handling
- [ ] Explain why tool description quality has more impact on agent reliability than model size
- [ ] Configure and run NAT Parameter Optimization with an evaluation dataset
- [ ] Diagnose agent failures caused by poor tool design (vague descriptions, missing validation, overlapping tools)
- [ ] Explain the difference between local and remote code execution sandboxes and when each is appropriate
- [ ] Describe how the Agent Hyperparameter Optimizer differs from single-parameter optimization
- [ ] Write few-shot examples that demonstrate correct tool usage for a multi-step task
