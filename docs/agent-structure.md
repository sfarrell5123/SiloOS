# SiloOS Agent Structure

## An Agent is a Folder

In SiloOS, an agent is simply a folder containing everything it needs to operate:

```
agents/refund-agent/
├── main.py                    # Entry point
├── tools.py                   # Python tools the agent can use
├── requirements.txt           # Dependencies
├── .keys/
│   └── base.jwt               # Base capabilities (chmod 600)
├── policies/
│   ├── README.md              # Agent's purpose and scope
│   ├── how-to-refund.md       # Step-by-step refund procedure
│   ├── escalation.md          # When and how to escalate
│   ├── limits.md              # Authority limits ($500 max, etc.)
│   └── edge-cases.md          # Unusual situations and handling
├── templates/
│   ├── email-confirmation.md  # Email template for confirmations
│   └── apology.md             # Template for service recovery
└── examples/
    ├── good-refund.md         # Example of well-handled refund
    └── escalation-example.md  # Example of proper escalation
```

This folder is:
- **Self-contained**: Everything the agent needs is here
- **Versionable**: Entire folder goes in git
- **Deployable**: Copy folder → agent is live
- **Inspectable**: A human can read and understand it

## The Markdown Operating System

The key insight: you don't program agents, you *instruct* them. Like training a new employee, you give them documentation.

### Policy Files

Markdown files in `policies/` tell the agent how to do its job:

```markdown
# how-to-refund.md

## When to Issue a Refund

A refund should be issued when:
1. Product was damaged on arrival
2. Product was significantly different from description
3. Product never arrived (after 14 days)
4. Customer changed mind within 30-day window

## Refund Process

1. Verify the customer's identity (use `authenticate_customer` tool)
2. Look up the order (use `get_order` tool)
3. Confirm the order is eligible for refund
4. Check refund amount is within your authority ($500 limit)
5. Process the refund (use `process_refund` tool)
6. Send confirmation email (use `send_email` tool with `confirmation` template)

## If Amount Exceeds Your Limit

If the refund amount exceeds $500:
- Explain to the customer you need manager approval
- Escalate to manager with reason: "refund_over_limit"
- Include your recommendation (approve/deny) and reasoning

## Never Refund If

- Order is older than 90 days
- Product shows signs of misuse
- This is the customer's 4th+ refund this year (escalate to fraud review)
```

The agent reads this like a human would—understanding context, making judgments, following the spirit of the instructions.

### Why Markdown?

- **Human-readable**: Managers can review and approve policies
- **LLM-friendly**: Models understand natural language better than code
- **Flexible**: Handles edge cases through explanation, not if/else trees
- **Auditable**: Changes tracked in git, diff-able, reviewable
- **No compilation**: Update markdown → agent behavior changes

## The Entry Point: main.py

`main.py` is the agent's brain—a thin wrapper that:

1. Loads the task (from environment/arguments)
2. Reads its policies
3. Makes LLM calls to reason about the task
4. Uses tools to take actions
5. Returns a result

```python
# Simplified main.py structure

import os
from silo_sdk import Agent, TaskResult

def main():
    # Initialize with base keys
    agent = Agent(
        keys_path=".keys/base.jwt",
        policies_path="policies/",
        tools_path="tools.py"
    )

    # Get task from environment (injected by router)
    task = agent.get_task()

    # Run the agent loop
    result = agent.run(task)

    # Return result to router
    return result

if __name__ == "__main__":
    result = main()
    TaskResult.output(result)
```

## Tools: Python Functions

If markdown is the agent's brain, tools are its hands.

- **Markdown** = reasoning, judgment, decisions, context
- **Tools** = execution, actions, real-world effects

The agent reads markdown to decide *what* to do. It calls tools to *do* it.

### The Tools File

```python
# tools.py

from silo_sdk import tool, require_keys

@tool
@require_keys("refund")
def process_refund(order_token: str, amount: float, reason: str) -> dict:
    """
    Process a refund for an order.

    Args:
        order_token: Tokenized order ID
        amount: Refund amount (must be within agent's limit)
        reason: Reason for refund

    Returns:
        {"success": True, "refund_id": "..."} or {"success": False, "error": "..."}
    """
    # This goes through the security proxy automatically
    return proxy.refund(
        order=order_token,
        amount=amount,
        reason=reason
    )

@tool
@require_keys("email")
def send_email(customer_token: str, template: str, variables: dict) -> dict:
    """
    Send an email to a customer using a template.

    Args:
        customer_token: Tokenized customer ID
        template: Template name from templates/ folder
        variables: Template variables

    Returns:
        {"sent": True, "message_id": "..."} or {"sent": False, "error": "..."}
    """
    return proxy.send_email(
        to=customer_token,
        template=template,
        variables=variables
    )

@tool
def get_order(order_token: str) -> dict:
    """
    Retrieve order details.

    Returns tokenized customer data and plain order data.
    """
    return proxy.get_order(order_token)

@tool
def authenticate_customer(method: str, value_hash: str) -> dict:
    """
    Verify customer identity.

    Args:
        method: "phone" or "email"
        value_hash: SHA256 hash of the value customer provided

    Returns:
        {"authenticated": True/False}
    """
    return proxy.authenticate(method=method, value_hash=value_hash)
```

### Tool Design Principles

**Keep it simple, stupid.**

| Principle | Why |
|-----------|-----|
| **Atomic** | One tool does one thing. `process_refund`, not `handle_customer_request`. |
| **Easy to deploy** | Just Python functions in a file. No build step. |
| **Easy to test** | Unit test each tool independently. Mock the proxy. |
| **Easy to recover** | Something wrong? Replace `tools.py`, redeploy. Done. |
| **Observable** | Every tool call is logged with inputs, outputs, timing. |
| **Composable** | Agent strings tools together as needed. Tools don't call tools. |

```python
# Good: Simple, atomic tools
@tool
def get_order(order_token): ...

@tool
def check_eligibility(order): ...

@tool
def process_refund(order_token, amount): ...

# Bad: Monolithic tool that does everything
@tool
def handle_refund_request(customer, order, reason, send_email=True, ...): ...
```

### Tools Don't Call Other Agents

This is critical. Tools are for *doing things*, not *coordinating*.

```python
# WRONG - tool tries to orchestrate
@tool
def complex_refund(order_token):
    result = call_agent("fraud-check", order_token)  # NO!
    if result.ok:
        call_agent("refund-processor", order_token)  # NO!
    return result

# RIGHT - tool does one thing, agent orchestrates via router
@tool
def process_refund(order_token, amount):
    return proxy.refund(order_token, amount)

# If fraud check is needed, the agent escalates through the router
# The markdown policy tells it when to escalate, not the tool
```

No inter-agent communication through tools. Everything goes through the router. Keep it simple.

### Why Python (Not MCP)?

MCP is bloated. It junks up context, requires the LLM to read and rewrite all data through verbose schemas, and adds overhead to every operation.

**The MCP problem:**

```
1. LLM reads massive tool schema (bloats context)
2. LLM generates MCP request (slow, token-heavy)
3. Request goes over network to MCP server
4. Server parses, executes, formats response
5. Full response comes back through context
6. LLM reads and interprets (more context bloat)
```

**Python is the muscle:**

```
1. LLM calls function
2. Python executes—fast, efficient, surgical
3. Only writes to context what it needs
4. Can write intermediate data to temp disk
5. Access to endless libraries
```

| | MCP | Python Tools |
|--|-----|--------------|
| Context usage | Heavy (full schemas, full responses) | Light (only what's needed) |
| Speed | Network + parsing overhead | Direct execution |
| Data handling | Everything through LLM context | Can use temp files, memory |
| Libraries | Whatever MCP exposes | All of PyPI |
| Infrastructure | MCP servers to manage | Just a `.py` file |

**Get AI to write code, not MCP.**

If you need a new capability, write a Python function. Or better—get the AI to write it during development. The "watch what I did" strategy works here too: show the AI what you need, let it write the tool.

Python code is fast, efficient, has access to thousands of libraries, and doesn't bloat the context with data the LLM doesn't need to see. Everything stays small, atomic, simple.

### Testing Tools

Tools are just Python functions. Test them like any other code:

```python
# test_tools.py
from unittest.mock import Mock
from tools import process_refund, get_order

def test_process_refund_success():
    mock_proxy = Mock()
    mock_proxy.refund.return_value = {"success": True, "refund_id": "ref_123"}

    result = process_refund("ord_tk_abc", 49.99, "defective")

    assert result["success"] == True
    mock_proxy.refund.assert_called_once_with(
        order="ord_tk_abc",
        amount=49.99,
        reason="defective"
    )

def test_process_refund_over_limit():
    # Test that proxy rejects amounts over agent's limit
    mock_proxy = Mock()
    mock_proxy.refund.return_value = {"success": False, "error": "over_limit"}

    result = process_refund("ord_tk_abc", 999.99, "defective")

    assert result["success"] == False
```

Run tests before deployment. If tests pass, deploy with confidence. If something breaks in production, rollback is just replacing the file.

## Templates

Templates are markdown files with placeholders:

```markdown
# templates/email-confirmation.md

Subject: Your refund has been processed - Order {{order_id}}

---

Hi {{customer_name}},

Great news! We've processed your refund of ${{amount}} for order {{order_id}}.

**Refund Details:**
- Amount: ${{amount}}
- Method: Original payment method
- Timeframe: 3-5 business days

If you have any questions, just reply to this email.

Thanks for your patience!

The Support Team
```

The agent composes the email using tokenized values. Hydration (replacing tokens with real values) happens outside the silo when the email is actually sent.

## Execution Modes

### Mode 1: One-Shot (Default)

```
Task arrives → main.py runs → Result returned → Process exits
```

Best for: Simple requests, quick operations, stateless interactions.

### Mode 2: Session (FastAPI/uvicorn)

```python
# main.py as a service
from fastapi import FastAPI
from silo_sdk import SessionAgent

app = FastAPI()
agent = SessionAgent(...)

@app.post("/message")
async def handle_message(message: str, session_id: str):
    return await agent.respond(message, session_id)
```

Best for: Multi-turn conversations within a single session.

### Mode 3: Deep Reasoning (Claude Code style)

For complex tasks requiring extended reasoning:

```python
# Uses full markdown operating system
# Extended context window
# Multiple tool calls per turn
# File creation/editing capabilities
```

Best for: Complex analysis, document generation, multi-step workflows.

## Temp/Scratch Space

Agents can write to a temporary folder that:

- Is unique per execution
- Lives in RAM (tmpfs)
- Is automatically cleaned up when main.py exits
- Is the *only* place the agent can write

```python
import os

temp = os.environ['SILO_TEMP']  # e.g., /tmp/silo-run-abc123/

# Agent can create files here
with open(f"{temp}/working.json", 'w') as f:
    json.dump(intermediate_data, f)

# But they're gone after execution
```

This prevents:
- Persistent state between runs
- Data leakage through filesystem
- Accumulation of garbage

## Agent Lifecycle

```
1. DEPLOY
   └──▶ Upload agent folder to agent registry
   └──▶ Validate policies and base keys
   └──▶ Create Linux user, set permissions
   └──▶ Agent is "live" and routable

2. RECEIVE TASK
   └──▶ Router selects this agent
   └──▶ Task keys minted and injected
   └──▶ main.py is invoked

3. EXECUTE
   └──▶ Load policies, tools, keys
   └──▶ Reason about task (LLM calls)
   └──▶ Use tools as needed
   └──▶ Compose response/actions

4. COMPLETE
   └──▶ Return result to router
   └──▶ Temp space cleaned
   └──▶ Task keys expire
   └──▶ Process exits

5. UPDATE
   └──▶ New version uploaded
   └──▶ Old version retired
   └──▶ Instant rollback if needed
```

## Summary

| Component | Purpose |
|-----------|---------|
| `main.py` | Entry point, thin orchestration |
| `tools.py` | Python functions for actions |
| `policies/` | Markdown instructions (the "brain") |
| `templates/` | Email/message templates |
| `.keys/` | Base capability tokens |
| `examples/` | Few-shot examples for the LLM |

An agent is just a folder. Version it, deploy it, inspect it, roll it back. No monolithic codebases, no complex dependencies, no deployment ceremonies.

*Small. Atomic. Inspectable. Shippable.*
