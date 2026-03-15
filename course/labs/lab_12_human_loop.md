# Lab 12: Human-in-the-Loop Integration

## Interaction Classification
- **Type**: Local NVIDIA tooling + Safety stack interaction
- **NVIDIA Services Used**: NAT interactive workflows (WebSocket), NeMo Guardrails execution rails, NAT authentication, NAT logging middleware, NIM API
- **Credentials Required**: NVIDIA_API_KEY from build.nvidia.com
- **Hardware**: CPU laptop sufficient
- **Milestone**: Milestone 8 (Integrated capstone, oversight component)

> **Package note**: This lab uses `nvidia-nat` (installed via `pip install nvidia-nat`). Python imports use `from nat...`. If you see references to `nvidia-agent-toolkit` or `nvidia_agent_toolkit` elsewhere, those are the same toolkit under a different name that may appear in some NVIDIA examples. This course standardizes on `nvidia-nat`.

## Objective

Add human oversight to the agent system: **approval workflows** for high-risk actions, **escalation** when agent confidence is low, **feedback collection** for continuous improvement, and **complete audit trails** for compliance. By the end of this lab you will have an agent that knows when to ask for human permission, routes uncertain cases to human operators, learns from human feedback, and produces a tamper-evident log of every decision.

## Prerequisites

| Requirement | Detail |
|---|---|
| Module 12 | Human-in-the-loop patterns, approval workflows, feedback loops, audit requirements |
| Lab 9 | Guardrails configuration (execution rails for tool approval) |
| Lab 10 | Deployed agent system (FastAPI server, Docker Compose) |
| Lab 11 | Observability setup (tracing for audit trail) |
| Software | Python 3.10+, `nvidia-nat[server]`, `nemoguardrails`, `websockets`, `fastapi` |

## Deliverables

1. NAT interactive workflow with WebSocket bidirectional communication
2. Execution rail for human approval of high-risk tool calls
3. Approval UI (terminal-based or simple web)
4. Per-user authentication with role-based tool permissions
5. Confidence-based escalation to human operators
6. Feedback collection system connected to the evaluation pipeline (Lab 8)
7. Complete audit trail with logging middleware and redaction
8. End-to-end test demonstrating the full human-in-the-loop cycle
9. Audit trail completeness verification

## Recommended Repo Structure

```
lab_12_human_loop/
├── workflows/
│   ├── interactive_workflow.py      # NAT interactive WebSocket workflow
│   ├── approval_workflow.py         # Human approval request/response logic
│   └── escalation.py               # Confidence-based escalation
├── guardrails/
│   ├── execution_approval.co        # Colang 2.0 execution rail for approvals
│   └── config_update.yml            # Updated guardrails config
├── auth/
│   ├── user_auth.py                 # Per-user authentication
│   ├── role_permissions.py          # Role-based tool permissions
│   └── users.yaml                   # User/role definitions
├── ui/
│   ├── approval_ui.py               # Terminal-based approval interface
│   └── web_ui.py                    # Optional: simple web approval UI
├── feedback/
│   ├── feedback_collector.py        # Collect human ratings
│   ├── feedback_store.py            # Persist feedback
│   └── feedback_to_eval.py          # Connect feedback to Lab 8 eval pipeline
├── audit/
│   ├── audit_logger.py              # Comprehensive audit logging
│   ├── redaction_processor.py       # PII redaction for audit logs
│   └── audit_verifier.py            # Verify audit trail completeness
├── tests/
│   ├── test_approval_flow.py        # Approval workflow tests
│   ├── test_escalation.py           # Escalation tests
│   ├── test_feedback.py             # Feedback collection tests
│   ├── test_audit_trail.py          # Audit completeness tests
│   └── test_end_to_end.py           # Full cycle integration test
├── requirements.txt
└── README.md
```

## Implementation Steps

### Step 1: Configure NAT Interactive Workflow with WebSocket

Set up a bidirectional WebSocket connection so the agent can pause execution and ask the human for input at any point.

```python
# workflows/interactive_workflow.py
from nat.server import FastAPIServer
from nat.workflows import InteractiveWorkflow
from fastapi import WebSocket, WebSocketDisconnect
import json
import asyncio

class HumanInLoopWorkflow(InteractiveWorkflow):
    """
    An agent workflow that can pause and request human input via WebSocket.
    """

    def __init__(self, agent, websocket: WebSocket):
        super().__init__(agent=agent)
        self.ws = websocket
        self.pending_approvals: dict[str, asyncio.Future] = {}

    async def request_human_input(self, prompt: str, input_type: str = "text") -> str:
        """Pause agent execution and request input from the human via WebSocket."""
        request_id = f"input-{len(self.pending_approvals)}"
        future = asyncio.get_event_loop().create_future()
        self.pending_approvals[request_id] = future

        await self.ws.send_json({
            "type": "human_input_request",
            "request_id": request_id,
            "prompt": prompt,
            "input_type": input_type,  # "text", "approval", "choice"
        })

        # Block until the human responds
        result = await asyncio.wait_for(future, timeout=300)  # 5 min timeout
        del self.pending_approvals[request_id]
        return result

    async def request_approval(self, action: str, details: dict) -> bool:
        """Request human approval for a specific action."""
        response = await self.request_human_input(
            prompt=json.dumps({
                "action": action,
                "details": details,
                "message": f"The agent wants to execute: {action}. Approve?",
            }),
            input_type="approval",
        )
        return response.lower() in ("yes", "approve", "y", "true")

    async def handle_human_response(self, data: dict):
        """Process incoming WebSocket messages from the human."""
        request_id = data.get("request_id")
        if request_id in self.pending_approvals:
            self.pending_approvals[request_id].set_result(data.get("response"))

# WebSocket endpoint
async def websocket_agent_endpoint(websocket: WebSocket):
    await websocket.accept()

    try:
        # Receive initial configuration
        init_msg = await websocket.receive_json()
        session_id = init_msg.get("session_id", "default")
        user_id = init_msg.get("user_id")

        # Create the interactive workflow
        from app.agent_config import create_agent_pipeline
        agent = create_agent_pipeline()
        workflow = HumanInLoopWorkflow(agent=agent, websocket=websocket)

        # Main message loop
        while True:
            message = await websocket.receive_json()

            if message["type"] == "user_query":
                # Run the agent with human-in-the-loop capability
                asyncio.create_task(
                    process_with_hitl(workflow, message, websocket, session_id, user_id)
                )

            elif message["type"] == "human_response":
                # Route human responses to pending approval requests
                await workflow.handle_human_response(message)

            elif message["type"] == "feedback":
                # Collect feedback on the previous response
                from feedback.feedback_collector import collect_feedback
                await collect_feedback(
                    session_id=session_id,
                    user_id=user_id,
                    rating=message.get("rating"),
                    comment=message.get("comment"),
                )

    except WebSocketDisconnect:
        print(f"Client disconnected: {session_id}")

async def process_with_hitl(workflow, message, websocket, session_id, user_id):
    """Process a user query with human-in-the-loop."""
    try:
        # Stream agent steps to the client
        async for step in workflow.run_stream(
            query=message["content"],
            session_id=session_id,
            user_id=user_id,
        ):
            await websocket.send_json({
                "type": "agent_step",
                "step_type": step.type,    # "thinking", "tool_call", "tool_result", "response"
                "content": step.content,
            })

    except asyncio.TimeoutError:
        await websocket.send_json({
            "type": "error",
            "message": "Human approval timed out. The action was not executed.",
        })
```

### Step 2: Implement Execution Rail for Human Approval

Extend the NeMo Guardrails execution rails from Lab 9 to require human approval for high-risk tool calls.

```colang
# guardrails/execution_approval.co

# Define high-risk actions that require human approval
define flow check tool call
  """Validate tool calls and require approval for high-risk operations."""

  # High-risk tools always require approval
  if $tool_name == "send_email"
    $approved = execute request_human_approval(
      action="send_email",
      details=$tool_args,
      risk_level="high"
    )
    if not $approved
      bot inform action rejected
      stop

  if $tool_name == "database_write"
    $approved = execute request_human_approval(
      action="database_write",
      details=$tool_args,
      risk_level="high"
    )
    if not $approved
      bot inform action rejected
      stop

  if $tool_name == "file_write"
    $approved = execute request_human_approval(
      action="file_write",
      details=$tool_args,
      risk_level="medium"
    )
    if not $approved
      bot inform action rejected
      stop

  # Code execution: require approval if the code modifies the filesystem or network
  if $tool_name == "code_interpreter"
    $has_side_effects = execute check_code_side_effects(code=$tool_args.code)
    if $has_side_effects
      $approved = execute request_human_approval(
        action="code_interpreter (with side effects)",
        details=$tool_args,
        risk_level="medium"
      )
      if not $approved
        bot inform action rejected
        stop

define bot inform action rejected
  "The action was not approved. I'll find an alternative approach or provide the information without executing that operation."
```

Register the approval action with NeMo Guardrails:

```python
# workflows/approval_workflow.py
from nemoguardrails import LLMRails

async def request_human_approval(action: str, details: dict, risk_level: str) -> bool:
    """
    Request human approval for a tool call.
    This function is called by the Colang execution rail.
    It communicates with the active WebSocket workflow to pause and ask the human.
    """
    # Get the active workflow for the current session
    # (stored in a session registry by the WebSocket handler)
    workflow = get_active_workflow()

    if workflow is None:
        # No interactive session -- check role-based auto-approval
        user_role = get_current_user_role()
        if user_role == "admin" and risk_level != "high":
            return True  # Admins can auto-approve medium/low risk
        # If no human available and no auto-approval, deny by default
        return False

    # Request approval via WebSocket
    approved = await workflow.request_approval(action=action, details=details)
    return approved

async def check_code_side_effects(code: str) -> bool:
    """Check if code has filesystem, network, or database side effects."""
    side_effect_patterns = [
        "open(", "write(", "os.remove", "os.unlink", "shutil.",
        "subprocess.", "os.system(", "requests.", "httpx.",
        "urllib.", "socket.", "sqlite3.", "psycopg",
        "pymongo.", "redis.", "import smtplib",
    ]
    return any(pattern in code for pattern in side_effect_patterns)

# Register with guardrails
def register_approval_actions(rails: LLMRails):
    rails.register_action(request_human_approval, name="request_human_approval")
    rails.register_action(check_code_side_effects, name="check_code_side_effects")
```

### Step 3: Build the Approval UI

**Terminal-based UI** for development and testing:

```python
# ui/approval_ui.py
import asyncio
import websockets
import json
from datetime import datetime

class TerminalApprovalUI:
    """Terminal-based UI for human-in-the-loop interactions."""

    def __init__(self, server_url: str = "ws://localhost:8000/v1/interactive"):
        self.server_url = server_url
        self.session_id = f"terminal-{datetime.now().strftime('%H%M%S')}"

    async def run(self, user_id: str = "operator-1"):
        """Main loop: send queries, handle approval requests, provide feedback."""
        async with websockets.connect(self.server_url) as ws:
            # Initialize session
            await ws.send(json.dumps({
                "type": "init",
                "session_id": self.session_id,
                "user_id": user_id,
            }))

            # Start listener for agent messages
            listener = asyncio.create_task(self._listen(ws))

            # Main input loop
            print("\n=== Agent Terminal (type 'quit' to exit) ===\n")
            while True:
                user_input = await asyncio.get_event_loop().run_in_executor(
                    None, input, "You: "
                )
                if user_input.lower() == "quit":
                    break

                await ws.send(json.dumps({
                    "type": "user_query",
                    "content": user_input,
                }))

            listener.cancel()

    async def _listen(self, ws):
        """Listen for messages from the agent."""
        try:
            async for raw in ws:
                msg = json.loads(raw)

                if msg["type"] == "agent_step":
                    step_type = msg["step_type"]
                    if step_type == "thinking":
                        print(f"  [Thinking] {msg['content'][:80]}...")
                    elif step_type == "tool_call":
                        print(f"  [Tool] {msg['content']}")
                    elif step_type == "tool_result":
                        print(f"  [Result] {msg['content'][:120]}...")
                    elif step_type == "response":
                        print(f"\nAgent: {msg['content']}\n")

                elif msg["type"] == "human_input_request":
                    await self._handle_approval_request(ws, msg)

                elif msg["type"] == "error":
                    print(f"\n  [ERROR] {msg['message']}\n")

        except asyncio.CancelledError:
            pass

    async def _handle_approval_request(self, ws, msg):
        """Display an approval request and collect the human's decision."""
        request_id = msg["request_id"]
        details = json.loads(msg["prompt"]) if isinstance(msg["prompt"], str) else msg["prompt"]

        print("\n" + "=" * 60)
        print("  APPROVAL REQUIRED")
        print("=" * 60)
        print(f"  Action:  {details.get('action', 'Unknown')}")
        print(f"  Details: {json.dumps(details.get('details', {}), indent=4)}")
        print(f"  Message: {details.get('message', '')}")
        print("=" * 60)

        response = await asyncio.get_event_loop().run_in_executor(
            None, input, "  Approve? (yes/no): "
        )

        await ws.send(json.dumps({
            "type": "human_response",
            "request_id": request_id,
            "response": response.strip(),
        }))

        status = "APPROVED" if response.strip().lower() in ("yes", "y") else "REJECTED"
        print(f"  [{status}]\n")

        # Also prompt for feedback after the response
        if status == "APPROVED":
            await self._collect_feedback(ws)

    async def _collect_feedback(self, ws):
        """Optionally collect feedback after a response."""
        rate = await asyncio.get_event_loop().run_in_executor(
            None, input, "  Rate this interaction (1-5, or skip): "
        )
        if rate.strip().isdigit():
            comment = await asyncio.get_event_loop().run_in_executor(
                None, input, "  Comment (or Enter to skip): "
            )
            await ws.send(json.dumps({
                "type": "feedback",
                "rating": int(rate.strip()),
                "comment": comment.strip() or None,
            }))
            print("  [Feedback recorded]\n")

# Entry point
if __name__ == "__main__":
    ui = TerminalApprovalUI()
    asyncio.run(ui.run(user_id="operator-1"))
```

### Step 4: Configure Per-User Authentication

```yaml
# auth/users.yaml
users:
  admin-alice:
    role: admin
    name: "Alice (Admin)"
    auto_approve:
      - low_risk
      - medium_risk
    # Admins can auto-approve low and medium risk; high risk still needs manual approval

  operator-bob:
    role: operator
    name: "Bob (Operator)"
    auto_approve:
      - low_risk
    # Operators auto-approve low risk only

  user-charlie:
    role: user
    name: "Charlie (User)"
    auto_approve: []
    # Regular users require approval for everything

roles:
  admin:
    description: "Full access, auto-approve most operations"
    tool_access:
      - web_search
      - code_interpreter
      - file_read
      - file_write
      - send_email
      - database_write

  operator:
    description: "Operational access, limited auto-approval"
    tool_access:
      - web_search
      - code_interpreter
      - file_read
      - file_write
      # Cannot use send_email or database_write

  user:
    description: "Basic access, no auto-approval"
    tool_access:
      - web_search
      - code_interpreter
      - file_read
      # Cannot use file_write, send_email, or database_write
```

```python
# auth/user_auth.py
import yaml
from dataclasses import dataclass

@dataclass
class User:
    user_id: str
    role: str
    name: str
    auto_approve: list[str]
    tool_access: list[str]

class UserAuthenticator:
    def __init__(self, config_path: str = "auth/users.yaml"):
        with open(config_path) as f:
            config = yaml.safe_load(f)
        self.users = config["users"]
        self.roles = config["roles"]

    def authenticate(self, user_id: str, token: str = None) -> User | None:
        """Authenticate a user and return their profile."""
        # In production, verify the token (JWT, API key, etc.)
        user_config = self.users.get(user_id)
        if not user_config:
            return None

        role_config = self.roles.get(user_config["role"], {})
        return User(
            user_id=user_id,
            role=user_config["role"],
            name=user_config["name"],
            auto_approve=user_config.get("auto_approve", []),
            tool_access=role_config.get("tool_access", []),
        )

    def can_use_tool(self, user: User, tool_name: str) -> bool:
        """Check if a user's role allows access to a specific tool."""
        return tool_name in user.tool_access

    def needs_approval(self, user: User, risk_level: str) -> bool:
        """Check if a user needs manual approval for a given risk level."""
        return risk_level not in user.auto_approve
```

```python
# auth/role_permissions.py
from nat.tools import BaseTool
from auth.user_auth import UserAuthenticator, User

auth = UserAuthenticator()

def filter_tools_for_user(all_tools: list[BaseTool], user: User) -> list[BaseTool]:
    """Return only the tools the user's role is allowed to access."""
    return [tool for tool in all_tools if auth.can_use_tool(user, tool.name)]

# Example: in the WebSocket handler
# user = auth.authenticate(user_id)
# agent.tools = filter_tools_for_user(all_tools, user)
```

### Step 5: Implement Confidence-Based Escalation

When the agent is uncertain, escalate to a human operator rather than guessing.

```python
# workflows/escalation.py
import re

class ConfidenceEscalation:
    """Escalate to human when agent confidence is below threshold."""

    def __init__(self, threshold: float = 0.6):
        self.threshold = threshold

    async def check_confidence(self, agent_response: str, workflow) -> str:
        """
        Check if the agent's response indicates low confidence.
        If so, escalate to a human.
        """
        confidence = self._estimate_confidence(agent_response)

        if confidence < self.threshold:
            # Escalate: ask the human to review or provide the answer
            human_input = await workflow.request_human_input(
                prompt=(
                    f"The agent is not confident in its response "
                    f"(estimated confidence: {confidence:.0%}).\n\n"
                    f"Agent's draft response:\n{agent_response}\n\n"
                    f"Please either:\n"
                    f"1. Type 'approve' to send the agent's response as-is\n"
                    f"2. Type a corrected response to send instead"
                ),
                input_type="text",
            )

            if human_input.lower() == "approve":
                return agent_response
            else:
                return f"[Human-provided response] {human_input}"

        return agent_response

    def _estimate_confidence(self, response: str) -> float:
        """
        Estimate agent confidence from the response text.
        In production, use the LLM's logprobs or a dedicated confidence model.
        """
        # Heuristic: look for uncertainty markers
        uncertainty_phrases = [
            "i'm not sure", "i don't know", "i think", "possibly",
            "it might be", "i believe", "approximately", "not certain",
            "hard to say", "unclear", "don't have enough information",
            "may not be accurate", "i cannot confirm",
        ]

        response_lower = response.lower()
        uncertainty_count = sum(
            1 for phrase in uncertainty_phrases if phrase in response_lower
        )

        # More uncertainty phrases = lower confidence
        confidence = max(0.0, 1.0 - (uncertainty_count * 0.15))

        # Also check response length -- very short responses may indicate uncertainty
        if len(response.split()) < 10:
            confidence *= 0.8

        return confidence

# Integration with the workflow
escalation = ConfidenceEscalation(threshold=0.6)

# In the agent's post-processing:
# final_response = await escalation.check_confidence(agent_response, workflow)
```

For a more robust approach using LLM logprobs (if available from the NIM endpoint):

```python
async def estimate_confidence_from_logprobs(llm_response) -> float:
    """Use LLM logprobs to estimate response confidence."""
    if not hasattr(llm_response, "logprobs") or llm_response.logprobs is None:
        return 0.5  # Unknown confidence

    # Average token probability
    token_probs = [
        math.exp(lp.logprob) for lp in llm_response.logprobs.content
        if lp.logprob is not None
    ]

    if not token_probs:
        return 0.5

    avg_prob = sum(token_probs) / len(token_probs)
    # Low-confidence tokens (probability < 0.3) count against overall confidence
    low_conf_ratio = sum(1 for p in token_probs if p < 0.3) / len(token_probs)

    confidence = avg_prob * (1 - low_conf_ratio * 0.5)
    return confidence
```

### Step 6: Implement Feedback Collection

```python
# feedback/feedback_collector.py
from datetime import datetime
from feedback.feedback_store import FeedbackStore

store = FeedbackStore()

async def collect_feedback(
    session_id: str,
    user_id: str,
    rating: int,          # 1-5
    comment: str = None,
    query: str = None,
    response: str = None,
    trace_id: str = None,
):
    """Collect human feedback on an agent response."""
    feedback_entry = {
        "session_id": session_id,
        "user_id": user_id,
        "timestamp": datetime.utcnow().isoformat(),
        "rating": rating,
        "comment": comment,
        "query": query,
        "response": response,
        "trace_id": trace_id,  # Link to observability trace
    }

    await store.save(feedback_entry)

    # If rating is low (1-2), flag for review
    if rating <= 2:
        await store.flag_for_review(feedback_entry)
        print(f"  [FLAGGED] Low rating ({rating}/5) from {user_id}: {comment}")
```

```python
# feedback/feedback_store.py
import json
import redis.asyncio as redis
from datetime import datetime

class FeedbackStore:
    """Persist feedback in Redis with export capability."""

    def __init__(self, redis_url: str = "redis://localhost:6379/2"):
        self.redis = redis.from_url(redis_url)

    async def save(self, entry: dict) -> str:
        """Save a feedback entry. Returns the entry ID."""
        entry_id = f"feedback:{entry['session_id']}:{datetime.utcnow().timestamp()}"
        await self.redis.set(entry_id, json.dumps(entry))
        # Also add to a sorted set for chronological retrieval
        await self.redis.zadd(
            "feedback:timeline",
            {entry_id: datetime.utcnow().timestamp()},
        )
        return entry_id

    async def flag_for_review(self, entry: dict):
        """Add low-rated entries to a review queue."""
        await self.redis.lpush("feedback:review_queue", json.dumps(entry))

    async def get_recent(self, count: int = 50) -> list[dict]:
        """Get the most recent feedback entries."""
        entry_ids = await self.redis.zrevrange("feedback:timeline", 0, count - 1)
        entries = []
        for eid in entry_ids:
            raw = await self.redis.get(eid)
            if raw:
                entries.append(json.loads(raw))
        return entries

    async def export_for_evaluation(self, output_path: str):
        """Export all feedback as JSONL for use in the evaluation pipeline (Lab 8)."""
        entries = await self.get_recent(count=10000)
        with open(output_path, "w") as f:
            for entry in entries:
                f.write(json.dumps(entry) + "\n")
        return len(entries)
```

### Step 7: Connect Feedback to Lab 8 Evaluation Pipeline

```python
# feedback/feedback_to_eval.py
import json

async def convert_feedback_to_eval_dataset(
    feedback_path: str = "feedback_export.jsonl",
    output_path: str = "eval_dataset_from_feedback.jsonl",
):
    """
    Convert human feedback into evaluation dataset entries.
    Low-rated responses become test cases for regression testing.
    """
    eval_cases = []

    with open(feedback_path) as f:
        for line in f:
            entry = json.loads(line)

            if entry.get("rating") and entry["rating"] <= 2:
                # Low-rated: this is a failure case to test against
                eval_case = {
                    "query": entry.get("query", ""),
                    "expected_answer": entry.get("comment", ""),  # Human correction
                    "category": "feedback_regression",
                    "difficulty": "hard",   # It failed before
                    "source": "human_feedback",
                    "original_response": entry.get("response", ""),
                    "original_rating": entry["rating"],
                }
                eval_cases.append(eval_case)

            elif entry.get("rating") and entry["rating"] >= 4:
                # High-rated: this is a known-good case for regression testing
                eval_case = {
                    "query": entry.get("query", ""),
                    "expected_answer": entry.get("response", ""),  # Agent got it right
                    "category": "feedback_golden",
                    "difficulty": "known_good",
                    "source": "human_feedback",
                }
                eval_cases.append(eval_case)

    with open(output_path, "w") as f:
        for case in eval_cases:
            f.write(json.dumps(case) + "\n")

    print(f"Converted {len(eval_cases)} feedback entries to evaluation dataset.")
    print(f"  Regression cases (low-rated): {sum(1 for c in eval_cases if c['category'] == 'feedback_regression')}")
    print(f"  Golden cases (high-rated):    {sum(1 for c in eval_cases if c['category'] == 'feedback_golden')}")
```

### Step 8: Configure Logging Middleware for Audit Trail

```python
# audit/audit_logger.py
import json
import logging
import hashlib
from datetime import datetime
from audit.redaction_processor import redact_pii

# Configure structured audit logger
audit_logger = logging.getLogger("audit")
audit_logger.setLevel(logging.INFO)

# File handler with rotation
handler = logging.handlers.RotatingFileHandler(
    "audit/audit_trail.jsonl",
    maxBytes=50 * 1024 * 1024,  # 50MB per file
    backupCount=10,
)
handler.setFormatter(logging.Formatter("%(message)s"))
audit_logger.addHandler(handler)

class AuditMiddleware:
    """
    Log every agent decision for compliance and debugging.
    Each log entry includes a hash chain for tamper detection.
    """

    def __init__(self):
        self.previous_hash = "GENESIS"

    def log_event(self, event: dict):
        """Log an audit event with hash chain."""
        entry = {
            "timestamp": datetime.utcnow().isoformat() + "Z",
            "event_type": event.get("type"),
            "session_id": event.get("session_id"),
            "user_id": event.get("user_id"),
            "trace_id": event.get("trace_id"),
            **event,
        }

        # Redact PII from the audit entry
        entry = redact_pii(entry)

        # Hash chain: each entry includes the hash of the previous entry
        entry_str = json.dumps(entry, sort_keys=True)
        entry["previous_hash"] = self.previous_hash
        entry["entry_hash"] = hashlib.sha256(
            (self.previous_hash + entry_str).encode()
        ).hexdigest()
        self.previous_hash = entry["entry_hash"]

        audit_logger.info(json.dumps(entry))
        return entry

    # Convenience methods for specific event types
    def log_query(self, session_id, user_id, query, trace_id):
        return self.log_event({
            "type": "user_query",
            "session_id": session_id,
            "user_id": user_id,
            "query": query,
            "trace_id": trace_id,
        })

    def log_tool_call(self, session_id, tool_name, tool_args, trace_id):
        return self.log_event({
            "type": "tool_call",
            "session_id": session_id,
            "tool_name": tool_name,
            "tool_args": tool_args,
            "trace_id": trace_id,
        })

    def log_approval_request(self, session_id, action, details, trace_id):
        return self.log_event({
            "type": "approval_request",
            "session_id": session_id,
            "action": action,
            "details": details,
            "trace_id": trace_id,
        })

    def log_approval_decision(self, session_id, action, approved, approver_id, trace_id):
        return self.log_event({
            "type": "approval_decision",
            "session_id": session_id,
            "action": action,
            "approved": approved,
            "approver_id": approver_id,
            "trace_id": trace_id,
        })

    def log_response(self, session_id, response, trace_id):
        return self.log_event({
            "type": "agent_response",
            "session_id": session_id,
            "response": response,
            "trace_id": trace_id,
        })

    def log_feedback(self, session_id, user_id, rating, comment, trace_id):
        return self.log_event({
            "type": "feedback",
            "session_id": session_id,
            "user_id": user_id,
            "rating": rating,
            "comment": comment,
            "trace_id": trace_id,
        })

    def log_escalation(self, session_id, reason, confidence, trace_id):
        return self.log_event({
            "type": "escalation",
            "session_id": session_id,
            "reason": reason,
            "confidence": confidence,
            "trace_id": trace_id,
        })
```

### Step 9: Configure Redaction for Audit Logs

```python
# audit/redaction_processor.py
import re
from copy import deepcopy

# PII patterns
PII_PATTERNS = [
    (re.compile(r'\b\d{3}-\d{2}-\d{4}\b'), '[SSN_REDACTED]'),
    (re.compile(r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b'), '[EMAIL_REDACTED]'),
    (re.compile(r'\b\d{3}[-.]?\d{3}[-.]?\d{4}\b'), '[PHONE_REDACTED]'),
    (re.compile(r'\b\d{4}[- ]?\d{4}[- ]?\d{4}[- ]?\d{4}\b'), '[CARD_REDACTED]'),
]

# Fields that should be redacted
SENSITIVE_FIELDS = {"query", "response", "comment", "tool_args", "details"}

def redact_pii(entry: dict) -> dict:
    """Redact PII from audit log entries."""
    redacted = deepcopy(entry)

    for key, value in redacted.items():
        if key in SENSITIVE_FIELDS and isinstance(value, str):
            for pattern, replacement in PII_PATTERNS:
                value = pattern.sub(replacement, value)
            redacted[key] = value
        elif isinstance(value, dict):
            redacted[key] = redact_pii(value)

    return redacted
```

### Step 10: Test the Full Human-in-the-Loop Cycle

Write an end-to-end test that exercises every component.

```python
# tests/test_end_to_end.py
import asyncio
import json
import pytest
from unittest.mock import AsyncMock
from workflows.interactive_workflow import HumanInLoopWorkflow
from audit.audit_logger import AuditMiddleware
from feedback.feedback_collector import collect_feedback

@pytest.mark.asyncio
async def test_full_hitl_cycle():
    """
    Test the complete flow:
    User query -> Agent processes -> Approval needed -> Human approves
    -> Agent completes -> Feedback collected -> Audit logged
    """
    # Setup
    mock_ws = AsyncMock()
    audit = AuditMiddleware()

    # Simulate the responses the "human" will give
    human_responses = asyncio.Queue()
    await human_responses.put({"type": "human_response", "request_id": "input-0", "response": "yes"})

    mock_ws.send_json = AsyncMock()
    mock_ws.receive_json = AsyncMock(side_effect=lambda: human_responses.get())

    # Step 1: Log user query
    audit.log_query(
        session_id="e2e-test",
        user_id="operator-bob",
        query="Send an email summary of today's metrics to the team",
        trace_id="trace-e2e-001",
    )

    # Step 2: Agent processes and requests approval for send_email
    audit.log_tool_call(
        session_id="e2e-test",
        tool_name="send_email",
        tool_args={"to": "team@company.com", "subject": "Daily Metrics"},
        trace_id="trace-e2e-001",
    )

    audit.log_approval_request(
        session_id="e2e-test",
        action="send_email",
        details={"to": "team@company.com", "subject": "Daily Metrics"},
        trace_id="trace-e2e-001",
    )

    # Step 3: Human approves
    approved = True  # Simulated from the queue above
    audit.log_approval_decision(
        session_id="e2e-test",
        action="send_email",
        approved=approved,
        approver_id="operator-bob",
        trace_id="trace-e2e-001",
    )
    assert approved is True

    # Step 4: Agent completes
    response = "Email sent successfully to team@company.com with daily metrics summary."
    audit.log_response(
        session_id="e2e-test",
        response=response,
        trace_id="trace-e2e-001",
    )

    # Step 5: Feedback collected
    audit.log_feedback(
        session_id="e2e-test",
        user_id="operator-bob",
        rating=5,
        comment="Perfect, exactly what I needed",
        trace_id="trace-e2e-001",
    )

    # Step 6: Verify audit trail
    # Read the audit log and verify all events are present
    with open("audit/audit_trail.jsonl") as f:
        events = [json.loads(line) for line in f if "e2e-test" in line]

    event_types = [e["event_type"] for e in events]
    assert "user_query" in event_types
    assert "tool_call" in event_types
    assert "approval_request" in event_types
    assert "approval_decision" in event_types
    assert "agent_response" in event_types
    assert "feedback" in event_types

    # Verify hash chain integrity
    for i in range(1, len(events)):
        assert events[i]["previous_hash"] == events[i - 1]["entry_hash"], (
            f"Hash chain broken at event {i}"
        )

    print("End-to-end test PASSED: all 6 event types logged with intact hash chain")
```

### Step 11: Verify Audit Trail Completeness

Build a verification tool that checks whether every agent decision can be reconstructed from the audit log.

```python
# audit/audit_verifier.py
import json
from collections import defaultdict

class AuditVerifier:
    """Verify that the audit trail is complete and consistent."""

    def __init__(self, audit_log_path: str = "audit/audit_trail.jsonl"):
        self.log_path = audit_log_path
        self.events = self._load_events()

    def _load_events(self) -> list[dict]:
        events = []
        with open(self.log_path) as f:
            for line in f:
                if line.strip():
                    events.append(json.loads(line))
        return events

    def verify_hash_chain(self) -> tuple[bool, list[str]]:
        """Verify the hash chain is intact (no tampering)."""
        errors = []
        for i in range(1, len(self.events)):
            expected_prev = self.events[i - 1]["entry_hash"]
            actual_prev = self.events[i]["previous_hash"]
            if expected_prev != actual_prev:
                errors.append(
                    f"Hash chain broken at event {i}: "
                    f"expected {expected_prev[:12]}..., got {actual_prev[:12]}..."
                )
        return len(errors) == 0, errors

    def verify_session_completeness(self, session_id: str) -> dict:
        """
        Verify that a session has all required events.
        Every session should have: query -> [tool_calls] -> response.
        If approval was needed: approval_request -> approval_decision.
        """
        session_events = [e for e in self.events if e.get("session_id") == session_id]
        event_types = [e["event_type"] for e in session_events]

        result = {
            "session_id": session_id,
            "total_events": len(session_events),
            "has_query": "user_query" in event_types,
            "has_response": "agent_response" in event_types,
            "tool_calls": event_types.count("tool_call"),
            "approval_requests": event_types.count("approval_request"),
            "approval_decisions": event_types.count("approval_decision"),
            "has_feedback": "feedback" in event_types,
            "issues": [],
        }

        # Check: every query should have a response
        if result["has_query"] and not result["has_response"]:
            result["issues"].append("Query without response (possible crash)")

        # Check: every approval request should have a decision
        if result["approval_requests"] != result["approval_decisions"]:
            result["issues"].append(
                f"Mismatched approvals: {result['approval_requests']} requests vs "
                f"{result['approval_decisions']} decisions"
            )

        # Check: events are in chronological order
        timestamps = [e["timestamp"] for e in session_events]
        if timestamps != sorted(timestamps):
            result["issues"].append("Events not in chronological order")

        result["complete"] = len(result["issues"]) == 0
        return result

    def verify_all_sessions(self) -> dict:
        """Verify completeness across all sessions."""
        sessions = set(e.get("session_id") for e in self.events if e.get("session_id"))
        results = {}
        for session_id in sessions:
            results[session_id] = self.verify_session_completeness(session_id)

        complete = sum(1 for r in results.values() if r["complete"])
        total = len(results)

        print(f"=== Audit Trail Verification ===")
        print(f"  Total sessions: {total}")
        print(f"  Complete: {complete} ({complete/total:.0%})")
        print(f"  Incomplete: {total - complete}")

        # Verify hash chain
        hash_ok, hash_errors = self.verify_hash_chain()
        print(f"  Hash chain: {'INTACT' if hash_ok else 'BROKEN'}")
        for err in hash_errors:
            print(f"    {err}")

        # Show incomplete sessions
        for sid, result in results.items():
            if not result["complete"]:
                print(f"\n  Incomplete session: {sid}")
                for issue in result["issues"]:
                    print(f"    - {issue}")

        return results

# Run verification
if __name__ == "__main__":
    verifier = AuditVerifier()
    verifier.verify_all_sessions()
```

## Evaluation Rubric

| Criterion | Excellent (5) | Satisfactory (3) | Needs Work (1) |
|---|---|---|---|
| **WebSocket workflow** | Bidirectional communication working, agent pauses and resumes correctly | WebSocket connects but flow is fragile | No WebSocket integration |
| **Approval workflow** | High-risk tool calls blocked until approved, medium-risk role-dependent | Some tool calls require approval | No approval mechanism |
| **Approval UI** | Functional terminal or web UI that displays details and collects decisions | Basic UI that works for simple cases | No UI |
| **Per-user auth** | 3 roles with different permissions, tool filtering per role, tested | 2 roles with some permission differences | No role-based access |
| **Escalation** | Confidence estimation working, low-confidence responses routed to human | Escalation exists but threshold not tuned | No escalation |
| **Feedback collection** | Ratings collected, stored, and exportable to evaluation dataset | Ratings collected but not connected to eval | No feedback collection |
| **Audit trail** | Complete log with hash chain, all event types, redaction, verified | Most events logged but no hash chain or redaction | Minimal or no audit logging |
| **End-to-end test** | Full cycle tested and passing with all 6 event types verified | Partial cycle tested | No integration test |
| **Audit verification** | Verification tool checks completeness and hash chain integrity | Manual verification only | No verification |

**Minimum passing score**: 27/45 (average 3 across all criteria)

## Likely Bugs / Failure Cases

1. **WebSocket connection drops during approval wait**: If the human takes too long or the network blips, the WebSocket disconnects and the agent hangs waiting for a response that will never come. Fix: implement a timeout on `request_human_input` (as shown: 5 minutes) and handle `WebSocketDisconnect` gracefully by defaulting to "deny".

2. **Race condition between agent steps and human responses**: The agent sends an approval request and immediately sends the next step before the human responds. Fix: ensure the agent workflow truly blocks on `await future` before proceeding. The `asyncio.Future` pattern handles this, but verify with tests.

3. **Hash chain breaks on concurrent writes**: If two sessions write audit entries simultaneously, the hash chain can get out of order. Fix: use a lock or write audit entries through a single-writer queue (`asyncio.Queue` with a dedicated consumer).

4. **Feedback ratings skewed positive**: Users tend to rate satisfactory responses 4-5 and only rate 1-2 when truly angry, creating a biased dataset. Fix: add structured feedback questions ("Was this answer factually correct? Yes/No") alongside the numeric rating.

5. **Approval UI blocks the main thread**: If using `input()` in the terminal UI without proper async wrapping, the entire event loop blocks and the agent cannot send intermediate steps. Fix: always use `asyncio.get_event_loop().run_in_executor(None, input, ...)` as shown in the implementation.

6. **PII leaks into audit log despite redaction**: Regex-based redaction misses PII in unusual formats (e.g., "My social is one two three four five six seven eight nine"). Fix: use Presidio or an LLM-based PII detector for the audit redaction processor rather than regex alone.

7. **Role permissions not enforced at the tool level**: The agent's LLM may still try to call tools that the user doesn't have access to, because filtering only removes tools from the tool list, not from the LLM's knowledge. Fix: also add an execution rail that checks permissions before any tool call executes.

## Extension Ideas

1. **Approval delegation and SLA tracking**: Implement a system where approval requests are assigned to specific operators based on expertise, with SLA tracking (e.g., high-risk approvals must be handled within 5 minutes). If the SLA is breached, escalate to a supervisor.

2. **Feedback-driven prompt tuning**: Automatically analyze low-rated responses, identify patterns (e.g., "the agent always gets X wrong"), and update the system prompt or few-shot examples to address the most common failure modes. Run the Lab 8 evaluation pipeline after each update to verify improvement.

3. **Audit trail analytics dashboard**: Build a dashboard over the audit log that shows: approvals per day, approval rate by tool type, average approval latency, feedback score trends, and escalation frequency. Use this data to identify which tools need better safety rails and which agents need retraining.
