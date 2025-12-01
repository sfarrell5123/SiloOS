# SiloOS Workflow Patterns

This document describes the high-level patterns for how work flows through SiloOS. For technical implementation details (YAML syntax, status codes, etc.), see [workflows.md](workflows.md).

## The Router as Kernel

The router is the central nervous system of SiloOS. It's the only component that:

- Receives external tasks
- Looks up which workflow to run
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

**The router is dumb on purpose.** No AI, no LLM calls, no "creativity." Just deterministic code that loads workflow definitions and follows the rules. This keeps the control plane secure, testable, and auditable.

## Task Lifecycle

### How Work Flows Through the System

```
1. Task arrives (web chat, email, API call)
         │
         ▼
2. Router looks up workflow by source
   (e.g., web_chat → support-workflow)
         │
         ▼
3. Router mints task keys
   (customer access token, case token, expiry)
         │
         ▼
4. Router runs first agent in workflow
         │
         ▼
5. Agent does its job, returns a result
         │
         ▼
6. Router handles result:
   ├── Success → next agent in workflow
   ├── Redirect → run specialist, come back
   ├── Human needed → route to human
   └── Abort → log and stop
```

### Intent Classification

Sometimes the router doesn't know what kind of task it is. That's where an **intent agent** comes in.

```
Unknown task arrives
        │
        ▼
Router runs intent-workflow
        │
        ▼
intent-agent classifies:
"This is a calendar request"
        │
        ▼
Router redirects to calendar-workflow
```

The intent agent is just another agent—it runs inside a padded cell like everything else. The router doesn't do the classification itself. The router stays dumb.

## Escalation Patterns

Real support teams escalate all the time. So do SiloOS agents.

### Vertical Escalation (Up the Chain)

When an agent hits its authority limit:

```
Customer: "I want a refund for $750"

refund-agent: [Checks: my limit is $500]
            → "I need manager approval"
            → Escalates up

manager-agent: [Has $2000 limit]
             → Approves refund
             → Returns success
```

The refund-agent knows its limits (defined in its base keys). When it can't proceed, it escalates with a recommendation. The manager-agent gets the context and makes the call.

### Horizontal Escalation (Different Department)

When the task belongs to a different team:

```
Customer: "My payment failed and now my account is locked"

support-agent: [This is a payments issue, not general support]
             → Escalates to payments-agent

payments-agent: [Checks payment status]
              → Unlocks account
              → Returns success
```

The support-agent recognizes it's out of its depth and hands off. No shame in that—it's the right thing to do.

### Human Escalation

When AI can't or shouldn't handle it:

```
Customer: "I'm going to sue you"

support-agent: [Detects legal threat]
             → Escalates to human immediately
```

Some things require human judgment. Legal threats, emotional customers, unusual situations—the agent knows when to step back.

## "Plug In a Human"

One of the most powerful patterns in SiloOS: **humans are just another agent type**.

> *"Hey, quickly—we've got to plug in a human!"*

### When Agents Go Offline

If an agent needs maintenance:

1. Mark it as offline in the router
2. All traffic routes to human fallback
3. Humans use the same interface
4. Fix and redeploy the agent
5. Traffic returns to AI

No downtime. No special handling. Humans slot right in.

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

The human sees the same tokenized customer ID the agent would see. Same tools, same constraints, same audit trail.

### Testing with Humans

Route a percentage of cases to humans for quality assurance. Compare human decisions against AI decisions. Use the differences to improve agent policies.

## Multi-Turn Conversations

Some tasks require back-and-forth:

```
Turn 1:
  Customer: "I want a refund"
  Agent: "Sure! What's your order number?"

Turn 2:
  Customer: "Order #12345"
  Agent: "I found it. Reason for return?"

Turn 3:
  Customer: "It arrived broken"
  Agent: "Refund processed! Confirmation sent."
```

The router maintains conversation state across turns. Each time the customer responds, the agent gets the full conversation history plus the new message. The agent is still stateless—it just gets more context each turn.

## Asking for Help (and Getting Control Back)

Sometimes an agent needs a specialist but wants to continue afterward.

```
intent-agent: "This needs calendar info"
            → Asks calendar-agent for help

calendar-agent: [Checks availability]
              → Returns: "Tuesday 2pm is free"

intent-agent: [Gets the answer back]
            → "I can book you for Tuesday 2pm. Confirm?"
```

The agent that asked for help gets control back with the specialist's output. It can then decide what to do next: continue, ask another specialist, or escalate to human.

For technical details on how this works, see the 302 soft redirect pattern in [workflows.md](workflows.md).

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
3. **Simplicity**: No agent discovery or authentication
4. **Control**: Router can intercept, redirect, or deny

It's like a support center: agents transfer calls through the switchboard, they don't yell across the room.

## Summary

| Pattern | Use Case |
|---------|----------|
| Vertical escalation | Authority limits, approval needed |
| Horizontal escalation | Different department or skill |
| Human escalation | AI can't or shouldn't handle |
| Plug in human | Agent offline, QA testing |
| Multi-turn | Conversations requiring dialogue |
| Ask for help | Need specialist input, then continue |

The router orchestrates everything. Agents are stateless workers. Humans are just another agent type. Everything flows through the kernel.

For implementation details, see [workflows.md](workflows.md).
