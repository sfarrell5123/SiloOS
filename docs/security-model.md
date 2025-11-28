# SiloOS Security Model

## Zero Trust by Default

SiloOS treats every AI agent as if it were:

- An external attacker
- A rogue employee
- Someone who will do whatever they want if given the chance

This isn't pessimism—it's realism. AI agents are non-deterministic, unpredictable, and fundamentally untrustable. The security model reflects this.

> *"You sort of take care of them—give them nice food and water—here please, do all my work for me. But I'm not going to let you out of here. I don't trust you."*

## The Two-Key System

Every agent operation requires two types of authorization:

### Base Keys (What You Can Do)

Base keys define the agent's *capabilities*—what operations it's allowed to perform regardless of which customer it's working with.

```json
{
  "agent": "refund-agent",
  "capabilities": {
    "refund": { "max_amount": 500 },
    "send_email": { "templates": ["confirmation", "apology"] },
    "escalate_to": ["manager", "human"]
  },
  "expires": "2024-12-31T00:00:00Z"
}
```

Base keys are:
- **Deployed with the agent**: Stored in the agent's folder
- **Blessed during deployment**: Part of the policy review process
- **Long-lived**: Changed only when the agent is updated
- **Function-specific**: A refund agent can refund; a support agent cannot

### Task Keys (Whose Data You Can Access)

Task keys define *scope*—which specific customer, case, or interaction the agent is working on.

```json
{
  "task_id": "chat-session-abc123",
  "customer_token": "cust_tk_789xyz",
  "case_token": "case_tk_456def",
  "granted_by": "router",
  "expires": "2024-03-15T14:30:00Z"
}
```

Task keys are:
- **Minted by the router**: Created when a task is assigned
- **Short-lived**: Expire after the interaction (often 30 minutes or less)
- **Scope-limited**: Only access this customer's data, this case's history
- **Revocable**: Can be invalidated if the agent misbehaves

### The Combination

An agent can only act when it has *both* keys:

```
Base Key: "I can issue refunds up to $500"
Task Key: "I'm working on customer #12345's case #67890"

Combined: "I can issue a refund up to $500 for customer #12345's case #67890"
```

Without the task key, the agent has capabilities but nothing to act on.
Without the base key, the agent has data access but no authority to do anything.

## The Security Proxy

Agents never talk directly to databases, APIs, or external services. Everything goes through the security proxy.

```
┌──────────────┐     ┌─────────────────┐     ┌─────────────────┐
│    Agent     │────▶│  Security Proxy │────▶│  Real Database  │
└──────────────┘     │                 │     └─────────────────┘
                     │  - Validates    │
                     │    both keys    │
                     │  - Logs access  │
                     │  - Enforces     │
                     │    limits       │
                     └─────────────────┘
```

### What the Proxy Does

1. **Validates keys**: Every request must include valid base + task keys
2. **Checks permissions**: Is this operation allowed by the base key?
3. **Checks scope**: Is this data covered by the task key?
4. **Enforces limits**: Is this refund under $500?
5. **Logs everything**: Complete audit trail
6. **Returns tokenized data**: Real PII is replaced with tokens (see [Privacy Model](privacy-model.md))

### What the Proxy Doesn't Do

The proxy doesn't implement business logic. It doesn't check "has this order been refunded before?" That's the agent's job—guided by its markdown policies.

The proxy enforces *capabilities* and *scope*, not *logic*.

## Agent Isolation

### OS-Level Isolation

Each agent runs as a dedicated Linux user:

```bash
# refund-agent runs as user 'silo_refund'
# support-agent runs as user 'silo_support'

# Each has its own home directory
/home/silo_refund/
/home/silo_support/

# File permissions prevent cross-reading
chmod 600 /home/silo_refund/.keys/base.jwt  # Only silo_refund can read
```

### Container/Jail Isolation

Beyond user separation, agents can run in:

- **Linux containers**: Docker/Podman with restricted capabilities
- **Linux jails**: chroot with limited system access
- **AppArmor/SELinux**: Mandatory access control profiles
- **seccomp**: System call filtering

### Network Isolation

Agents have extremely limited network access:

```
# Agent's allowed connections:
ALLOW: llm-router.silo.internal:443
ALLOW: security-proxy.silo.internal:443
DENY: *
```

No arbitrary HTTP. No direct database connections. No email servers. Everything goes through controlled proxies.

## Key Lifecycle

### Minting Task Keys

```
1. Router receives task (customer chat initiated)
2. Router generates task key:
   - customer_token: tokenized customer ID
   - case_token: tokenized case ID
   - expires: now + 30 minutes
   - signed by router's private key
3. Router passes task key to agent
4. Agent includes key with every proxy request
```

### Key Validation

```
1. Agent sends request to proxy with both keys
2. Proxy validates base key:
   - Valid signature?
   - Not expired?
   - Has required capability?
3. Proxy validates task key:
   - Valid signature?
   - Not expired?
   - Covers requested data?
4. If both valid: process request
5. If either invalid: reject with 401/403
```

### Key Revocation

Task keys can be invalidated:

```
1. Agent misbehaves (detected by monitoring)
2. Router adds task key to revocation list
3. Proxy checks revocation list on each request
4. Agent's remaining requests are rejected
5. Task is re-routed (to another agent or human)
```

## Escalation and Key Transfer

When an agent escalates, keys don't transfer directly. Instead:

```
1. Agent A returns: { escalate: "manager", reason: "over $500 limit" }
2. Agent A's task key expires/is revoked
3. Router mints NEW task key for Agent B (manager)
4. New key has same customer/case tokens but fresh timestamp
5. Agent B receives new key, continues the task
```

This ensures:
- Complete audit trail of who had access when
- No key sharing between agents
- Router maintains central control

## Audit and Observability

### Everything is Logged

| Event | Logged Data |
|-------|-------------|
| Task assignment | Router, agent, task keys (hashed), timestamp |
| Proxy request | Agent, keys (hashed), operation, data accessed |
| LLM call | Agent, prompt (full), response (full), tokens used |
| Escalation | From agent, to agent, reason, key transfer |
| Key rejection | Agent, attempted operation, failure reason |

### Real-Time Monitoring

```
- Active agents and their current tasks
- Key usage patterns (anomaly detection)
- Proxy request rates
- LLM token consumption
- Failed access attempts
```

## Security Boundaries

| Boundary | Protection |
|----------|------------|
| Agent ↔ Agent | OS user isolation, file permissions |
| Agent ↔ Database | Proxy-only access, key validation |
| Agent ↔ External APIs | Proxy-only access, template-based |
| Agent ↔ LLM | Unrestricted but fully logged |
| Agent ↔ Filesystem | Temp folder only, cleaned on exit |

## The Result

An agent in SiloOS:

- **Can** reason freely (unlimited LLM access)
- **Can** use its granted capabilities (base key)
- **Can** access scoped data (task key)
- **Cannot** exceed its authority
- **Cannot** access other customers' data
- **Cannot** bypass the proxy
- **Cannot** hide what it's doing

Zero trust. Complete observability. Full capability within strict bounds.
