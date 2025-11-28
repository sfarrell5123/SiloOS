# SiloOS Architecture

## The Padded Cell

The core insight of SiloOS is that AI agents are like dangerous prisoners with extraordinary abilities. You want them working for you—but you don't trust them. The architecture reflects this reality.

```
┌─────────────────────────────────────────────────────────────────┐
│                        THE OUTSIDE WORLD                         │
│   (Real databases, real emails, real customer PII)              │
└─────────────────────────────────────────────────────────────────┘
                              │
                    ┌─────────▼─────────┐
                    │   SECURITY PROXY   │
                    │   (Key Validation) │
                    └─────────┬─────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│                         SILO OS                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    ROUTER (Kernel)                       │    │
│  │         - Receives tasks                                 │    │
│  │         - Mints task keys                                │    │
│  │         - Routes to agents                               │    │
│  │         - Handles escalation                             │    │
│  └────────────────────┬────────────────────────────────────┘    │
│                       │                                          │
│    ┌──────────────────┼──────────────────┐                      │
│    │                  │                  │                       │
│    ▼                  ▼                  ▼                       │
│  ┌────────┐      ┌────────┐       ┌────────┐                    │
│  │ Agent  │      │ Agent  │       │ Agent  │                    │
│  │ (jail) │      │ (jail) │       │ (jail) │                    │
│  └────────┘      └────────┘       └────────┘                    │
│                                                                  │
│              THE PADDED CELL                                     │
└──────────────────────────────────────────────────────────────────┘
```

## Agent Isolation

Each agent runs in complete isolation:

### Operating System Level

- **Dedicated Linux user**: Each agent runs under its own username
- **File permissions**: `chmod 600` on sensitive files—agents can't read each other's data
- **Linux jails/containers**: AppArmor, seccomp, or container isolation
- **Network restrictions**: Firewall rules limit what ports agents can access

### Filesystem Level

```
/agents/
├── refund-agent/           # Owned by user 'refund-agent'
│   ├── main.py             # Entry point
│   ├── tools.py            # Agent's toolkit
│   ├── policies/
│   │   ├── how-to-refund.md
│   │   ├── escalation.md
│   │   └── limits.md
│   ├── templates/
│   │   └── email-confirmation.md
│   └── .keys/              # chmod 600
│       └── base.jwt
│
├── support-agent/          # Owned by user 'support-agent'
│   └── ...
│
└── collections-agent/      # Owned by user 'collections-agent'
    └── ...
```

### Network Level

Agents have extremely limited network access:

```
Agent → LLM Router (allowed)
Agent → Security Proxy (allowed)
Agent → Everything Else (BLOCKED)
```

No direct database access. No direct email sending. No arbitrary HTTP requests. Everything goes through controlled proxies that validate keys.

## The Router (Kernel)

The router is the brain of SiloOS. It's the only component that:

1. **Receives external tasks**: Web chat, incoming email, webhooks, scheduled jobs
2. **Mints task keys**: Creates time-limited JWT tokens for specific customer/case access
3. **Routes to agents**: Decides which agent should handle a task
4. **Handles returns**: When an agent finishes, escalates, or errors

### Routing Flow

```
1. Customer starts web chat
   └──▶ Router receives task

2. Router mints task keys
   └──▶ { customer_id: 12345, case_id: 67890, expires: +30min }

3. Router selects agent
   └──▶ support-agent (based on task type)

4. Agent executes
   └──▶ main.py runs with base keys + task keys

5. Agent completes/escalates
   └──▶ { status: "escalate", target: "manager", reason: "over limit" }

6. Router handles result
   └──▶ Routes to manager-agent with transferred keys
```

## Stateless Execution

Agents are completely stateless. Each execution:

1. **Starts fresh**: `main.py` is invoked
2. **Receives keys**: Base keys (what it can do) + task keys (whose data it accesses)
3. **Does work**: Makes LLM calls, runs tools, accesses proxied data
4. **Returns result**: Success, error, or escalation
5. **Exits cleanly**: All state is discarded

### Temp Space

Agents get a temporary scratch folder:

```python
# Temp folder is:
# - In RAM (tmpfs) for speed
# - Isolated per execution
# - Automatically cleaned up when main.py exits
# - The ONLY place the agent can write

temp_folder = os.environ['SILO_TEMP']  # /tmp/agent-run-abc123/
```

No persistent state between runs. If the agent needs to remember something, it goes through the proxy and gets persisted to the case record.

## The LLM Router

Agents have unrestricted access to LLM inference—that's where their intelligence comes from. But:

- **All requests are logged**: Every prompt, every response
- **Token usage is tracked**: Tied to agent + task for billing/auditing
- **No direct API access**: Goes through SiloOS LLM router

```
Agent ──▶ SiloOS LLM Router ──▶ OpenAI/Anthropic/etc.
              │
              └──▶ Logs everything
```

This gives us complete observability into what the agent is "thinking" without restricting its ability to reason.

## Execution Models

SiloOS supports multiple execution patterns:

### 1. Request/Response (Default)

```
Task ──▶ main.py ──▶ Response
```

Simple, stateless, synchronous. Good for quick interactions.

### 2. Long-Running (uvicorn)

```python
# main.py runs as a FastAPI service
# Handles multiple requests on its port
# Still isolated, still key-validated
```

Good for agents that need to maintain a conversation within a single session.

### 3. Claude Code Style

For complex tasks that require deep reasoning and file manipulation:

```python
# Heavier execution model
# Full markdown operating system
# Extended context and tool use
```

## No Direct Agent-to-Agent Communication

This is a deliberate architectural choice. Agents don't call each other. Instead:

1. Agent A can't handle something
2. Agent A returns to the router: `{ escalate: "collections" }`
3. Router mints new keys, routes to Agent B
4. Agent B handles it (or escalates further)

**Why?**

- **Simpler security model**: All key transfers go through the router
- **Observable**: Every handoff is logged at the router
- **Matches reality**: Real support teams transfer calls, not have side conversations
- **Avoids complexity**: No need for agent discovery, authentication, or protocols

## Summary

SiloOS architecture is defined by isolation and control:

| Component | Purpose |
|-----------|---------|
| **Padded Cell** | Complete isolation from real systems |
| **Agent Jails** | OS-level isolation between agents |
| **Router** | Central coordinator, key minter, traffic controller |
| **Security Proxy** | Validates keys, mediates all external access |
| **LLM Router** | Unrestricted intelligence, complete logging |
| **Stateless Execution** | No persistent state, clean starts |

The result: AI agents with full reasoning capabilities, but zero trust and complete observability.
