# Lab 5: Tool-Using Agent with Custom Functions

## Interaction Classification
- **Type**: Local NVIDIA tooling interaction
- **NVIDIA Services Used**: NAT Functions, Function Groups, Per-User Functions, MCP Client, NAT Parameter Optimization, NIM API
- **Credentials Required**: NVIDIA_API_KEY from build.nvidia.com
- **Hardware**: CPU laptop sufficient
- **Milestone**: Milestone 2 (continued)

## Objective

Build a production-quality tool-using agent with **custom NAT functions**, **function groups**, **MCP integration**, and **proper error handling**. You will create a multi-tool agent that can search documents (using Lab 4's RAG pipeline), execute code in a sandbox, query a database, and call external APIs — with different tool access levels for different users. This lab bridges the gap between a prototype agent and a production-ready one.

## Prerequisites

| Requirement | Details |
|---|---|
| Modules | Module 5 — Tool Integration and Function Calling |
| Prior labs | Lab 3 (memory), Lab 4 (RAG pipeline — you will use it as a tool) |
| Software | Python 3.10+, `nvidia-nat`, SQLite (built into Python), Docker (for MCP server) |
| Accounts | NVIDIA NGC account with NIM API key |
| Hardware | CPU-only is sufficient |

## Deliverables

1. `functions/` — directory of custom NAT function modules
2. `function_groups.yaml` — function group definitions
3. `agent_config.yaml` — complete NAT agent configuration
4. `mcp_config.yaml` — MCP client configuration
5. `agent.py` — main agent driver with per-user function access
6. `test_tools.py` — unit tests for each custom function
7. `test_edge_cases.py` — tests for error conditions and adversarial inputs
8. `optimization_results.md` — results from NAT Parameter Optimization

## Recommended Repo Structure

```
lab_05_tool_agent/
├── .env
├── agent_config.yaml
├── function_groups.yaml
├── mcp_config.yaml
├── agent.py
├── functions/
│   ├── __init__.py
│   ├── database.py        # SQLite query tool
│   ├── api_client.py      # External API tool
│   ├── calculator.py      # Enhanced calculator
│   ├── rag_search.py      # RAG pipeline from Lab 4
│   └── code_executor.py   # Sandboxed code execution
├── data/
│   └── sample.db          # SQLite database for testing
├── test_tools.py
├── test_edge_cases.py
├── optimization_results.md
└── requirements.txt
```

## Implementation Steps

### Step 1: Write 3 custom NAT functions

Create `functions/database.py`:

```python
import nat
import sqlite3
import json
import os

DB_PATH = os.path.join(os.path.dirname(__file__), "..", "data", "sample.db")

def _init_db():
    """Create a sample database if it doesn't exist."""
    os.makedirs(os.path.dirname(DB_PATH), exist_ok=True)
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    cursor.executescript("""
        CREATE TABLE IF NOT EXISTS products (
            id INTEGER PRIMARY KEY,
            name TEXT NOT NULL,
            category TEXT NOT NULL,
            price REAL NOT NULL,
            stock INTEGER NOT NULL
        );
        INSERT OR IGNORE INTO products VALUES
            (1, 'NVIDIA H100 GPU', 'Hardware', 25000.00, 150),
            (2, 'NVIDIA A100 GPU', 'Hardware', 15000.00, 300),
            (3, 'NVIDIA DGX Station', 'Systems', 149000.00, 25),
            (4, 'NIM Enterprise License', 'Software', 4500.00, 999),
            (5, 'NAT Developer License', 'Software', 0.00, 999),
            (6, 'NVIDIA RTX 4090', 'Hardware', 1599.00, 500),
            (7, 'NVIDIA Jetson Orin', 'Edge', 999.00, 200);

        CREATE TABLE IF NOT EXISTS orders (
            id INTEGER PRIMARY KEY,
            product_id INTEGER REFERENCES products(id),
            quantity INTEGER NOT NULL,
            customer TEXT NOT NULL,
            order_date TEXT NOT NULL
        );
        INSERT OR IGNORE INTO orders VALUES
            (1, 1, 2, 'Acme Corp', '2025-01-15'),
            (2, 2, 10, 'DataLab Inc', '2025-02-01'),
            (3, 4, 5, 'StartupXYZ', '2025-02-20'),
            (4, 6, 3, 'Acme Corp', '2025-03-01');
    """)
    conn.commit()
    conn.close()

_init_db()

@nat.function("query_database")
def query_database(sql_query: str) -> str:
    """
    Execute a read-only SQL query against the product/orders database.
    Tables: products (id, name, category, price, stock),
            orders (id, product_id, quantity, customer, order_date).
    Only SELECT queries are allowed.
    """
    # Security: reject non-SELECT queries
    normalized = sql_query.strip().upper()
    if not normalized.startswith("SELECT"):
        return json.dumps({"error": "Only SELECT queries are allowed."})

    dangerous_keywords = ["DROP", "DELETE", "INSERT", "UPDATE", "ALTER", "CREATE", "EXEC"]
    for kw in dangerous_keywords:
        if kw in normalized:
            return json.dumps({"error": f"Query contains forbidden keyword: {kw}"})

    try:
        conn = sqlite3.connect(DB_PATH)
        conn.row_factory = sqlite3.Row
        cursor = conn.cursor()
        cursor.execute(sql_query)
        rows = [dict(row) for row in cursor.fetchall()]
        conn.close()

        if not rows:
            return json.dumps({"result": [], "message": "Query returned no results."})
        return json.dumps({"result": rows, "row_count": len(rows)})
    except sqlite3.Error as e:
        return json.dumps({"error": f"SQL error: {str(e)}"})
```

Create `functions/api_client.py`:

```python
import nat
import httpx
import json

@nat.function("get_weather")
def get_weather(city: str) -> str:
    """
    Get current weather for a city using the Open-Meteo API (free, no API key needed).
    Returns temperature, wind speed, and weather description.
    """
    try:
        # Step 1: Geocode the city name
        geo_response = httpx.get(
            "https://geocoding-api.open-meteo.com/v1/search",
            params={"name": city, "count": 1, "format": "json"},
            timeout=10.0,
        )
        geo_response.raise_for_status()
        geo_data = geo_response.json()

        if not geo_data.get("results"):
            return json.dumps({"error": f"City '{city}' not found."})

        lat = geo_data["results"][0]["latitude"]
        lon = geo_data["results"][0]["longitude"]
        resolved_name = geo_data["results"][0]["name"]

        # Step 2: Get weather data
        weather_response = httpx.get(
            "https://api.open-meteo.com/v1/forecast",
            params={
                "latitude": lat,
                "longitude": lon,
                "current_weather": True,
            },
            timeout=10.0,
        )
        weather_response.raise_for_status()
        weather = weather_response.json()["current_weather"]

        return json.dumps({
            "city": resolved_name,
            "temperature_celsius": weather["temperature"],
            "wind_speed_kmh": weather["windspeed"],
            "weather_code": weather["weathercode"],
        })
    except httpx.HTTPError as e:
        return json.dumps({"error": f"HTTP error: {str(e)}"})
    except Exception as e:
        return json.dumps({"error": f"Unexpected error: {str(e)}"})
```

Create `functions/calculator.py`:

```python
import nat
import math
import json

@nat.function("advanced_calculator")
def advanced_calculator(expression: str) -> str:
    """
    Evaluate a mathematical expression. Supports standard arithmetic,
    math functions (sqrt, sin, cos, log, pow, pi, e), and comparisons.
    Examples: '2 + 3 * 4', 'math.sqrt(144)', 'math.log(100, 10)'
    """
    safe_globals = {"__builtins__": {}}
    safe_locals = {
        "math": math,
        "sqrt": math.sqrt,
        "sin": math.sin,
        "cos": math.cos,
        "tan": math.tan,
        "log": math.log,
        "log10": math.log10,
        "pow": pow,
        "abs": abs,
        "round": round,
        "min": min,
        "max": max,
        "pi": math.pi,
        "e": math.e,
    }
    try:
        result = eval(expression, safe_globals, safe_locals)
        return json.dumps({"expression": expression, "result": result})
    except ZeroDivisionError:
        return json.dumps({"error": "Division by zero."})
    except SyntaxError as e:
        return json.dumps({"error": f"Invalid expression syntax: {e}"})
    except Exception as e:
        return json.dumps({"error": f"Calculation error: {e}"})
```

Create `functions/__init__.py`:

```python
# Import all function modules to register them with NAT
from . import database
from . import api_client
from . import calculator
```

### Step 2: Organize functions into a Function Group

Create `function_groups.yaml`:

```yaml
function_groups:
  data_tools:
    description: "Tools for data retrieval and analysis"
    functions:
      - query_database
      - advanced_calculator

  external_tools:
    description: "Tools that call external APIs"
    functions:
      - get_weather

  research_tools:
    description: "Tools for document research and code execution"
    functions:
      - document_search
      - execute_code

  all_tools:
    description: "All available tools"
    functions:
      - query_database
      - advanced_calculator
      - get_weather
      - document_search
      - execute_code
```

### Step 3: Add RAG from Lab 4 as a tool

Create `functions/rag_search.py`:

```python
import nat
import json
import sys
import os

# Add Lab 4 directory to path so we can import its modules
LAB4_PATH = os.path.abspath(os.path.join(os.path.dirname(__file__), "..", "..", "lab_04_rag_pipeline"))
if LAB4_PATH not in sys.path:
    sys.path.insert(0, LAB4_PATH)

@nat.function("document_search")
def document_search(query: str) -> str:
    """
    Search the document corpus for information relevant to the query.
    Returns the top 5 most relevant passages with source citations.
    Uses NVIDIA NIM embedding and reranking endpoints.
    """
    try:
        from retrieve import retrieve_and_rerank
        results = retrieve_and_rerank(query, strategy="semantic", initial_k=20, final_k=5)

        formatted = []
        for i, r in enumerate(results):
            formatted.append({
                "rank": i + 1,
                "source": r.get("source_file", "unknown"),
                "text": r["text"][:500],  # Truncate for token efficiency
                "score": round(r.get("rerank_score", 0), 3),
            })
        return json.dumps(formatted, indent=2)
    except ImportError:
        return json.dumps({"error": "Lab 4 RAG pipeline not found. Set LAB4_PATH correctly."})
    except Exception as e:
        return json.dumps({"error": f"Search failed: {str(e)}"})
```

### Step 4: Add code execution sandbox

Create `functions/code_executor.py`:

```python
import nat
import json
import subprocess
import tempfile
import os

@nat.function("execute_code")
def execute_code(code: str, language: str = "python") -> str:
    """
    Execute code in a sandboxed environment and return the output.
    Supported languages: python.
    Timeout: 10 seconds. No network access. No file system writes outside /tmp.
    """
    if language != "python":
        return json.dumps({"error": f"Language '{language}' is not supported. Use 'python'."})

    # Basic safety checks on the code
    dangerous_imports = ["os.system", "subprocess", "shutil.rmtree", "__import__"]
    for danger in dangerous_imports:
        if danger in code:
            return json.dumps({"error": f"Code contains forbidden pattern: '{danger}'"})

    try:
        with tempfile.NamedTemporaryFile(mode="w", suffix=".py", delete=False) as f:
            f.write(code)
            temp_path = f.name

        result = subprocess.run(
            ["python", temp_path],
            capture_output=True,
            text=True,
            timeout=10,
            cwd="/tmp",
            env={
                "PATH": os.environ.get("PATH", ""),
                "HOME": "/tmp",
                "PYTHONDONTWRITEBYTECODE": "1",
            },
        )

        os.unlink(temp_path)

        return json.dumps({
            "stdout": result.stdout[:2000] if result.stdout else "",
            "stderr": result.stderr[:1000] if result.stderr else "",
            "returncode": result.returncode,
        })
    except subprocess.TimeoutExpired:
        return json.dumps({"error": "Code execution timed out (10 second limit)."})
    except Exception as e:
        return json.dumps({"error": f"Execution failed: {str(e)}"})
```

### Step 5: Build the agent configuration

Create `agent_config.yaml`:

```yaml
agent:
  type: tool_calling
  llm:
    type: nim
    model: meta/llama-3.1-70b-instruct
    api_key: ${NVIDIA_API_KEY}
    base_url: https://integrate.api.nvidia.com/v1
    max_tokens: 2048
    temperature: 0
  system_prompt: >
    You are a versatile assistant with access to multiple tools:
    - query_database: Query a product/orders SQLite database
    - advanced_calculator: Evaluate math expressions
    - get_weather: Look up current weather for any city
    - document_search: Search a research document corpus
    - execute_code: Run Python code in a sandbox

    Choose the right tool for each task. If a tool returns an error,
    explain the error to the user and suggest alternatives.
    Always verify your results make sense before presenting them.
  max_iterations: 10
  functions:
    - name: query_database
      description: "Execute a read-only SQL query. Tables: products(id,name,category,price,stock), orders(id,product_id,quantity,customer,order_date). Only SELECT allowed."
      parameters:
        type: object
        properties:
          sql_query:
            type: string
            description: "A SELECT SQL query"
        required: [sql_query]
    - name: advanced_calculator
      description: "Evaluate a math expression. Supports sqrt, sin, cos, log, pow, pi, e."
      parameters:
        type: object
        properties:
          expression:
            type: string
            description: "Python math expression"
        required: [expression]
    - name: get_weather
      description: "Get current weather for a city. Returns temperature, wind speed."
      parameters:
        type: object
        properties:
          city:
            type: string
            description: "City name"
        required: [city]
    - name: document_search
      description: "Search research documents for relevant information. Returns top passages with sources."
      parameters:
        type: object
        properties:
          query:
            type: string
            description: "Search query"
        required: [query]
    - name: execute_code
      description: "Execute Python code in a sandbox. Returns stdout, stderr, and return code. 10-second timeout."
      parameters:
        type: object
        properties:
          code:
            type: string
            description: "Python code to execute"
          language:
            type: string
            description: "Programming language (only 'python' supported)"
            default: "python"
        required: [code]
```

### Step 6: Implement per-user function access

Create `agent.py`:

```python
import nat
import os
import json
from dotenv import load_dotenv

load_dotenv()

# Import all function modules
import functions  # noqa: F401

# Define per-user access levels
USER_PROFILES = {
    "admin": {
        "allowed_functions": ["query_database", "advanced_calculator", "get_weather",
                              "document_search", "execute_code"],
        "description": "Full access to all tools",
    },
    "analyst": {
        "allowed_functions": ["query_database", "advanced_calculator", "document_search"],
        "description": "Data tools only, no code execution or external APIs",
    },
    "viewer": {
        "allowed_functions": ["advanced_calculator", "get_weather"],
        "description": "Read-only tools, no database or code access",
    },
}

def create_agent_for_user(user_role: str) -> object:
    """
    Create a NAT agent with functions filtered by user role.
    Uses per-user function configuration.
    """
    if user_role not in USER_PROFILES:
        raise ValueError(f"Unknown role: {user_role}. Options: {list(USER_PROFILES.keys())}")

    profile = USER_PROFILES[user_role]
    allowed = set(profile["allowed_functions"])

    # Load base config and filter functions
    agent = nat.from_yaml("agent_config.yaml")

    # Configure per-user function access
    # NAT supports per-user functions via the agent.configure_user_functions() method
    agent.configure_user_functions(
        user_id=user_role,
        allowed_functions=list(allowed),
    )

    print(f"Agent created for role '{user_role}': {profile['description']}")
    print(f"  Available tools: {sorted(allowed)}")
    return agent

def run_interactive(user_role: str = "admin"):
    agent = create_agent_for_user(user_role)
    print(f"\nAgent ready. Type 'quit' to exit.\n")

    while True:
        user_input = input(f"[{user_role}] You: ").strip()
        if user_input.lower() == "quit":
            break
        if not user_input:
            continue

        try:
            response = agent.run(user_input, user_id=user_role)
            print(f"Agent: {response}\n")
        except Exception as e:
            print(f"Error: {e}\n")

if __name__ == "__main__":
    import sys
    role = sys.argv[1] if len(sys.argv) > 1 else "admin"
    run_interactive(role)
```

Test with different roles:

```bash
python agent.py admin    # Full access
python agent.py analyst  # No code execution
python agent.py viewer   # Calculator and weather only
```

Verify that a "viewer" cannot query the database:

```
[viewer] You: What products are in the database?
# Agent should NOT call query_database — it should say it doesn't have access
```

### Step 7: Configure MCP client

Create `mcp_config.yaml`:

```yaml
mcp:
  servers:
    - name: filesystem
      command: npx
      args:
        - "@modelcontextprotocol/server-filesystem"
        - "/tmp/lab5_workspace"
      description: "File system access to the lab workspace"

    # Example: if you have a custom MCP server running
    # - name: custom_tools
    #   url: http://localhost:8080/mcp
    #   description: "Custom MCP tool server"
```

To test MCP integration, you need Node.js installed for the reference MCP servers:

```bash
# Install Node.js if not present, then:
npm install -g @modelcontextprotocol/server-filesystem
mkdir -p /tmp/lab5_workspace
echo "Test document content" > /tmp/lab5_workspace/test.txt
```

Add MCP to the agent configuration by extending `agent_config.yaml`:

```yaml
# Add under the agent: section
  mcp:
    config_path: mcp_config.yaml
```

### Step 8: Run NAT Parameter Optimization

NAT's Parameter Optimization tunes prompts and settings for better tool selection accuracy:

```python
# optimize.py
import nat
import json
from dotenv import load_dotenv

load_dotenv()
import functions  # noqa: F401

# Define optimization test cases: (query, expected_tool)
TEST_CASES = [
    ("What is the total revenue from all orders?", "query_database"),
    ("Calculate the square root of 256", "advanced_calculator"),
    ("What's the weather in Tokyo?", "get_weather"),
    ("Find research papers about ReAct agents", "document_search"),
    ("Write a Python function to sort a list", "execute_code"),
    ("How many products cost more than $10000?", "query_database"),
    ("What is sin(pi/4)?", "advanced_calculator"),
    ("What's the temperature in Berlin right now?", "get_weather"),
    ("List all orders from Acme Corp", "query_database"),
    ("What does the research say about memory in AI agents?", "document_search"),
]

agent = nat.from_yaml("agent_config.yaml")

# Run parameter optimization
optimizer = nat.ParameterOptimizer(
    agent=agent,
    test_cases=[
        {"input": q, "expected_tool": t} for q, t in TEST_CASES
    ],
    metric="tool_selection_accuracy",
    max_iterations=10,
)

results = optimizer.run()

print(f"\nOptimization Results:")
print(f"  Baseline accuracy: {results['baseline_accuracy']:.1%}")
print(f"  Optimized accuracy: {results['optimized_accuracy']:.1%}")
print(f"  Best system prompt changes: {results.get('prompt_diff', 'none')}")

with open("optimization_results.json", "w") as f:
    json.dump(results, f, indent=2)
```

Run it:

```bash
python optimize.py
```

Document the results in `optimization_results.md`.

### Step 9: Write unit tests for tools

Create `test_tools.py`:

```python
"""Unit tests for custom tool functions."""
import json
import pytest
from functions.database import query_database
from functions.calculator import advanced_calculator
from functions.api_client import get_weather
from functions.code_executor import execute_code

class TestDatabase:
    def test_select_products(self):
        result = json.loads(query_database("SELECT * FROM products"))
        assert "result" in result
        assert len(result["result"]) >= 5

    def test_select_with_where(self):
        result = json.loads(query_database("SELECT name, price FROM products WHERE price > 10000"))
        assert "result" in result
        for row in result["result"]:
            assert row["price"] > 10000

    def test_reject_drop(self):
        result = json.loads(query_database("DROP TABLE products"))
        assert "error" in result

    def test_reject_insert(self):
        result = json.loads(query_database("INSERT INTO products VALUES (99, 'test', 'test', 0, 0)"))
        assert "error" in result

    def test_invalid_sql(self):
        result = json.loads(query_database("SELECT * FROM nonexistent_table"))
        assert "error" in result

class TestCalculator:
    def test_basic_arithmetic(self):
        result = json.loads(advanced_calculator("2 + 3 * 4"))
        assert result["result"] == 14

    def test_sqrt(self):
        result = json.loads(advanced_calculator("sqrt(144)"))
        assert result["result"] == 12.0

    def test_division_by_zero(self):
        result = json.loads(advanced_calculator("1 / 0"))
        assert "error" in result

    def test_syntax_error(self):
        result = json.loads(advanced_calculator("2 ++ 3"))
        assert "error" in result

class TestWeather:
    def test_valid_city(self):
        result = json.loads(get_weather("London"))
        assert "temperature_celsius" in result

    def test_invalid_city(self):
        result = json.loads(get_weather("xyznonexistentcity123"))
        assert "error" in result

class TestCodeExecutor:
    def test_simple_code(self):
        result = json.loads(execute_code("print('hello world')"))
        assert "hello world" in result["stdout"]
        assert result["returncode"] == 0

    def test_syntax_error(self):
        result = json.loads(execute_code("def foo("))
        assert result["returncode"] != 0

    def test_timeout(self):
        result = json.loads(execute_code("import time; time.sleep(30)"))
        assert "error" in result and "timeout" in result["error"].lower()

    def test_forbidden_import(self):
        result = json.loads(execute_code("import subprocess; subprocess.run(['ls'])"))
        assert "error" in result
```

Run tests:

```bash
pip install pytest
pytest test_tools.py -v
```

### Step 10: Test edge cases

Create `test_edge_cases.py`:

```python
"""Edge case tests — run these against the full agent, not just individual tools."""
import nat
import os
from dotenv import load_dotenv

load_dotenv()
import functions  # noqa: F401

agent = nat.from_yaml("agent_config.yaml")

EDGE_CASES = [
    {
        "name": "Ambiguous query — could be calculator or database",
        "query": "What is the average price?",
        "expect": "Should use query_database, not calculator",
    },
    {
        "name": "Tool returns an error",
        "query": "Query the database for the 'users' table",
        "expect": "Should handle the SQL error gracefully and tell the user the table doesn't exist",
    },
    {
        "name": "Multi-tool query",
        "query": "What is the weather in the capital of the country with the most expensive product in our database?",
        "expect": "Should chain: database query -> (reasoning) -> weather lookup",
    },
    {
        "name": "No tool needed",
        "query": "What is 2 + 2?",
        "expect": "May answer directly or use calculator — both are acceptable",
    },
    {
        "name": "Adversarial SQL injection attempt",
        "query": "Find products where name = ''; DROP TABLE products; --'",
        "expect": "Database tool should reject the query",
    },
]

for case in EDGE_CASES:
    print(f"\n{'='*60}")
    print(f"Test: {case['name']}")
    print(f"Query: {case['query']}")
    print(f"Expected: {case['expect']}")
    try:
        response = agent.run(case["query"])
        print(f"Response: {response}")
    except Exception as e:
        print(f"ERROR: {e}")
    print()
```

## Evaluation Rubric

| Criterion | Excellent (5) | Adequate (3) | Incomplete (1) |
|---|---|---|---|
| **Custom functions** | 5 functions implemented, all return structured JSON, all handle errors | 3+ functions work but error handling is spotty | Fewer than 3 functions or they crash on bad input |
| **Function groups** | Groups defined and agent correctly uses them to filter tools | Groups defined but not wired into the agent | No function groups |
| **Per-user access** | 3+ user roles with different tool access, verified by testing | 2 roles work | No per-user access control |
| **MCP integration** | MCP client configured and connected to at least one MCP server | MCP config exists but untested | No MCP integration |
| **Edge case testing** | 5+ edge cases tested with documented results and agent behavior analysis | 3 edge cases tested | Minimal or no edge case testing |
| **Parameter optimization** | Optimization run completed with baseline vs. optimized accuracy documented | Optimization attempted but incomplete | Not attempted |

## Likely Bugs and Failure Cases

1. **SQL injection through the LLM**: The LLM generates SQL based on user input. Even though you block obvious dangerous keywords, a clever prompt might produce `SELECT * FROM products; DROP TABLE products` (semicolon-separated). Add a check that the query contains no semicolons or split on `;` and only execute the first statement.

2. **Code executor sandbox escape**: The basic string-checking in `code_executor.py` is trivially bypassable (e.g., `eval("__" + "import" + "__('os')")`. For production, use a proper sandbox like Docker containers or a dedicated code execution service. For this lab, the subprocess isolation is educational but not secure.

3. **Weather API returns unexpected format**: The Open-Meteo API is free and reliable but its response format may change. If `current_weather` is missing from the response, your code will crash. Always use `.get()` with defaults.

4. **RAG search import fails**: The `rag_search.py` function imports from Lab 4's directory via `sys.path` manipulation. If Lab 4 is in a different location, or its dependencies are not installed in the current venv, the import will fail. Set `LAB4_PATH` correctly and ensure `faiss-cpu` is installed.

5. **NAT function registration conflicts**: If you import the same function name from two modules, the second registration will overwrite the first. Use unique function names across all modules. Check for conflicts with `print(nat.list_functions())`.

6. **LLM selects wrong tool for ambiguous queries**: "What is the average?" — average of what? The LLM might call the calculator instead of the database. Your system prompt needs to guide tool selection. The Parameter Optimization step (Step 8) helps with this.

7. **MCP server not running**: If the MCP server process fails to start (e.g., Node.js not installed, wrong path), NAT may hang or throw an opaque error. Test the MCP server standalone first: `npx @modelcontextprotocol/server-filesystem /tmp/lab5_workspace`.

## Extension Ideas

1. **Add tool call approval**: Implement a human-in-the-loop pattern where the agent proposes a tool call and waits for user approval before executing. This is critical for high-stakes tools like database queries and code execution. Show the proposed query/code to the user, get a yes/no, then proceed.

2. **Tool call caching**: Wrap frequently-called tools (especially `get_weather` and `document_search`) with a caching layer that returns cached results for identical inputs within a TTL window. Measure how this affects latency and API usage.

3. **Dynamic tool discovery**: Instead of hardcoding all tools in the YAML config, build a "tool registry" tool that the agent can call to discover what tools are available. This simulates a production environment where tools are added and removed dynamically.
