# SiloOS Workflow Patterns

## The Router as Kernel

The router is the central nervous system of SiloOS. It's the only component that:

- Receives external tasks
- Knows about all available agents
- Mints and manages task keys
- Handles escalation and handoffs
- Routes to humans when needed

Think of it as the kernel of the operating system—all communication flows through it.

```
                    External World
                          │
          ┌───────────────▼───────────────┐
          │                               │
   Web ───┤                               ├─── Email
          │         ROUTER                │
   API ───┤        (Kernel)               ├─── SMS
          │                               │
  Cron ───┤                               ├─── Webhooks
          │                               │
          └───────────────┬───────────────┘
                          │
          ┌───────────────▼───────────────┐
          │         Agent Pool            │
          │                               │
          │  ┌─────┐ ┌─────┐ ┌─────┐     │
          │  │ A1  │ │ A2  │ │ A3  │     │
          │  └─────┘ └─────┘ └─────┘     │
          │  ┌─────┐ ┌─────┐ ┌─────┐     │
          │  │ A4  │ │ A5  │ │Human│     │
          │  └─────┘ └─────┘ └─────┘     │
          │                               │
          └───────────────────────────────┘
```

## Task Lifecycle

### 1. Task Arrival

```python
# Customer starts a web chat
incoming_task = {
    "source": "web_chat",
    "customer_token": "cust_tk_789xyz",  # Already tokenized by ingress
    "session_id": "sess_abc123",
    "message": "I want a refund for my order",
    "metadata": {
        "channel": "website",
        "page": "/support",
        "timestamp": "2024-03-15T14:22:33Z"
    }
}
```

### 2. Initial Classification

The router makes a quick classification:

```python
# Router's internal process
classification = router.classify(incoming_task)
# Returns: {"type": "refund_request", "confidence": 0.92}
```

### 3. Key Minting

```python
# Router creates task keys
task_keys = router.mint_keys(
    customer=incoming_task["customer_token"],
    task_type=classification["type"],
    ttl=timedelta(minutes=30)
)

# task_keys now contains:
# - Customer data access token
# - Case ID token
# - Expiration timestamp
# - Router signature
```

### 4. Agent Selection

```python
# Find the right agent
agent = router.select_agent(
    task_type=classification["type"],
    required_capabilities=["refund", "email"]
)
# Returns: "refund-agent"
```

### 5. Task Dispatch

```python
# Dispatch to agent
result = router.dispatch(
    agent="refund-agent",
    task=incoming_task,
    task_keys=task_keys
)
```

### 6. Result Handling

```python
# Agent returns one of:

# SUCCESS - Task completed
{"status": "complete", "response": "Refund processed", "actions_taken": [...]}

# ESCALATE - Needs another agent
{"status": "escalate", "target": "manager", "reason": "over_limit", "context": {...}}

# ERROR - Something went wrong
{"status": "error", "error": "customer_not_found", "recoverable": False}

# CONTINUE - Multi-turn conversation
{"status": "continue", "response": "What's your order number?", "awaiting": "user_input"}
```

## Escalation Patterns

### Vertical Escalation (To Manager)

When an agent hits its authority limit:

```
Customer: "I want a refund for $750"

refund-agent: [Checks limit: $500 max]
            → Escalates to manager-agent

manager-agent: [Has $2000 limit]
             → Processes refund
             → Returns success
```

The flow:

```
1. refund-agent returns:
   { "escalate": "manager", "reason": "amount_exceeds_limit",
     "recommendation": "approve", "notes": "Valid return, good customer history" }

2. Router receives escalation

3. Router mints NEW task keys for manager-agent
   (Old keys are invalidated)

4. Router dispatches to manager-agent with:
   - Original task context
   - Escalation reason
   - Previous agent's recommendation
   - Fresh task keys

5. manager-agent handles it
```

### Horizontal Escalation (To Another Department)

When the task belongs to a different team:

```
Customer: "My payment failed and now my account is locked"

support-agent: [This is a payments issue]
             → Escalates to payments-agent

payments-agent: [Checks payment status]
              → Unlocks account
              → Returns success
```

### Human Escalation

When AI can't or shouldn't handle it:

```
Customer: "I'm going to sue you"

support-agent: [Detects legal threat]
             → Escalates to human
```

## "Plug In a Human"

One of the most powerful patterns in SiloOS: humans are just another agent type.

> *"Hey, quickly—we've got to plug in a human!"*

### When Agents Go Offline

If an agent needs maintenance:

1. Mark it as offline in the router
2. All traffic routes to human fallback
3. Humans use the same interface
4. Fix and redeploy the agent
5. Traffic returns to AI

```python
# Router configuration
agents = {
    "refund-agent": {
        "status": "offline",  # Under maintenance
        "fallback": "human-pool"
    }
}
```

### The Human Interface

Humans get:

- Same task assignment as agents
- Same tokenized data (PII-protected)
- Same tools (click to run)
- Same markdown procedures (read if unsure)
- Same response format

```
┌────────────────────────────────────────────────────┐
│  TASK: Refund Request                              │
│  Customer: cust_tk_789xyz                          │
│  Case: case_tk_abc123                              │
├────────────────────────────────────────────────────┤
│                                                    │
│  Customer message:                                 │
│  "I want a refund for my broken widget"            │
│                                                    │
│  [View Order] [View History] [View Policy]         │
│                                                    │
├────────────────────────────────────────────────────┤
│  Actions:                                          │
│  [Process Refund] [Send Email] [Escalate]          │
│                                                    │
├────────────────────────────────────────────────────┤
│  Response:                                         │
│  ┌──────────────────────────────────────────────┐ │
│  │                                              │ │
│  │                                              │ │
│  └──────────────────────────────────────────────┘ │
│                           [Send] [Escalate]        │
└────────────────────────────────────────────────────┘
```

### Testing with Humans

Route 1-in-100 cases to humans for quality assurance:

```python
# Router sampling
if random.random() < 0.01:
    route_to = "human-qa"  # Human handles it
else:
    route_to = "refund-agent"  # AI handles it
```

Compare human vs AI decisions to improve agent policies.

## Multi-Turn Conversations

Some tasks require back-and-forth:

```
Turn 1:
  Customer: "I want a refund"
  Agent: "Sure! What's your order number?"
  → Returns: { "status": "continue", "awaiting": "order_number" }

Turn 2:
  Customer: "Order #12345"
  Agent: "I found it. Reason for return?"
  → Returns: { "status": "continue", "awaiting": "reason" }

Turn 3:
  Customer: "It arrived broken"
  Agent: "Refund processed! Confirmation sent."
  → Returns: { "status": "complete" }
```

### Conversation State

The router maintains conversation state:

```python
conversation = {
    "session_id": "sess_abc123",
    "customer_token": "cust_tk_789xyz",
    "agent": "refund-agent",
    "turns": [
        {"role": "customer", "content": "I want a refund"},
        {"role": "agent", "content": "Sure! What's your order number?"},
        {"role": "customer", "content": "Order #12345"},
        # ...
    ],
    "awaiting": "reason",
    "task_keys": {...}  # Still valid
}
```

Each turn, the full conversation is passed to the agent.

## Return to Sender

When an agent escalates, it can include a "return address":

```python
{
    "escalate": "manager",
    "reason": "needs_approval",
    "return_to": "refund-agent",  # Come back here after
    "context": {
        "approved_action": "refund",
        "amount": 750
    }
}
```

If the manager approves:

```python
# Manager returns
{
    "return": "refund-agent",
    "decision": "approved",
    "notes": "Customer has good history"
}

# Router sends back to refund-agent
# With manager's approval in context
```

## No Direct Agent-to-Agent Communication

This is crucial. Agents never talk directly to each other.

**Wrong:**
```
Agent A ───────────────▶ Agent B
        (direct call)
```

**Right:**
```
Agent A ───▶ Router ───▶ Agent B
```

**Why?**

1. **Security**: All key transfers go through router
2. **Observability**: Every handoff is logged
3. **Simplicity**: No agent discovery/authentication
4. **Control**: Router can intercept, redirect, or deny

It's like a support center: agents transfer calls through the switchboard, they don't yell across the room.

## Summary

| Pattern | Use Case |
|---------|----------|
| Vertical escalation | Authority limits, approval needed |
| Horizontal escalation | Different department/skill |
| Human escalation | AI can't/shouldn't handle |
| Plug in human | Agent offline, QA testing |
| Multi-turn | Conversations requiring dialogue |
| Return to sender | Need approval then continue |

The router orchestrates everything. Agents are stateless workers. Humans are just another agent type. Everything flows through the kernel.
