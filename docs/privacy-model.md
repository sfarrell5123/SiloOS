# SiloOS Privacy Model

## The Core Principle

**Agents never see real personally identifiable information (PII).**

Not customer names. Not email addresses. Not phone numbers. Not physical addresses. The agent works in a tokenized world where real data is replaced with opaque references.

## Why Tokenization?

AI agents have unrestricted access to LLM inference—they need it to think. But that means:

- Every piece of data they process might flow through an external LLM API
- LLM providers log inputs (for safety, training, debugging)
- The agent's "thoughts" could be logged, cached, or leaked
- A compromised agent could exfiltrate data through its LLM calls

By tokenizing PII, even if the agent's entire context were leaked, the attacker gets:

```
"Customer cust_tk_789xyz at address addr_tk_456def
requested a refund for order ord_tk_123abc"
```

Useless without the token-to-real mapping, which lives outside the silo.

## How Tokenization Works

### The Tokenization Service

Outside the SiloOS boundary, a tokenization service manages the mapping:

```
┌─────────────────────────────────────────────────────────┐
│                    OUTSIDE THE SILO                      │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │           Tokenization Service                    │   │
│  │                                                   │   │
│  │   Real Data              Token                    │   │
│  │   ─────────              ─────                    │   │
│  │   john@email.com    ←→   email_tk_a1b2c3         │   │
│  │   555-123-4567      ←→   phone_tk_d4e5f6         │   │
│  │   123 Main St       ←→   addr_tk_g7h8i9          │   │
│  │   John Smith        ←→   name_tk_j0k1l2          │   │
│  └──────────────────────────────────────────────────┘   │
│                                                          │
└─────────────────────────────────────────────────────────┘
                            │
                    ┌───────▼───────┐
                    │ Security Proxy │
                    └───────┬───────┘
                            │
┌───────────────────────────▼─────────────────────────────┐
│                    INSIDE THE SILO                       │
│                                                          │
│  Agent sees:                                             │
│  {                                                       │
│    "customer": "cust_tk_789xyz",                        │
│    "email": "email_tk_a1b2c3",                          │
│    "phone": "phone_tk_d4e5f6",                          │
│    "order_total": 149.99,  // Non-PII passes through    │
│    "product": "Widget Pro"  // Non-PII passes through   │
│  }                                                       │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### What Gets Tokenized

| Data Type | Tokenized? | Example Token |
|-----------|------------|---------------|
| Customer name | Yes | `name_tk_j0k1l2` |
| Email address | Yes | `email_tk_a1b2c3` |
| Phone number | Yes | `phone_tk_d4e5f6` |
| Physical address | Yes | `addr_tk_g7h8i9` |
| Credit card | Yes | `card_tk_m3n4o5` |
| Order ID | Maybe | Depends on sensitivity |
| Product name | No | Plain text |
| Order total | No | Plain number |
| Chat history | Partial | Names/emails tokenized |

### Token Properties

Tokens are:

- **Opaque**: No information about the real value can be derived
- **Consistent**: Same real value → same token (within a context)
- **Scoped**: Tokens are tied to task keys (can't use one customer's token to access another's data)
- **Reversible**: Only by the tokenization service, outside the silo

## Hydration: When Real Data is Needed

Sometimes the real data needs to be used—like sending an actual email. But the agent doesn't do this directly.

### The Hydration Pattern

```
1. Agent composes email:
   {
     "to": "email_tk_a1b2c3",
     "template": "refund-confirmation",
     "variables": {
       "name": "name_tk_j0k1l2",
       "amount": 49.99
     }
   }

2. Agent submits to email proxy with both keys

3. Email proxy (OUTSIDE the silo):
   - Validates base key (is this agent allowed to send emails?)
   - Validates task key (is this agent working on this customer?)
   - Calls tokenization service to hydrate tokens
   - Renders template with real values
   - Sends actual email

4. Agent receives: { "status": "sent", "message_id": "msg_123" }
```

The agent never sees:
- The real email address
- The real customer name
- The rendered email content

It just knows the email was sent.

### Other Hydration Points

| Action | Hydration Point | Agent Sees |
|--------|-----------------|------------|
| Send email | Email proxy | Status only |
| Send SMS | SMS proxy | Status only |
| Process payment | Payment proxy | Transaction ID |
| Ship order | Shipping proxy | Tracking number |
| Verify phone | Verification proxy | Valid/invalid |

## Privacy-Preserving Operations

Some operations need to work with real data without exposing it.

### Validation

```python
# Agent wants to verify customer's phone number
# But can't see the actual phone number

request = {
    "operation": "validate_phone",
    "token": "phone_tk_d4e5f6",
    "provided_value_hash": sha256("555-123-4567")  # Customer provided this
}

response = proxy.validate(request)
# Returns: { "valid": true } or { "valid": false }
```

The agent:
- Receives a hash of what the customer typed
- Asks the proxy to compare against the real value
- Gets a boolean result
- Never sees the actual phone number

### Geographic Queries

```python
# Agent needs to know customer's state for tax purposes
# But shouldn't see their full address

request = {
    "operation": "get_jurisdiction",
    "address_token": "addr_tk_g7h8i9"
}

response = proxy.query(request)
# Returns: { "state": "CA", "country": "US" }
```

The agent gets enough information to make decisions without seeing the full address.

### Comparison Operations

```python
# Agent needs to check if two customers are the same person
# (Fraud detection, duplicate prevention)

request = {
    "operation": "compare_customers",
    "token_a": "cust_tk_789xyz",
    "token_b": "cust_tk_111222"
}

response = proxy.compare(request)
# Returns: { "same_email": false, "same_phone": true, "same_address": false }
```

## Special Considerations

### Chat History

Customer conversations need special handling:

```
Original chat:
"Hi, my name is John and my email is john@email.com"

Tokenized for agent:
"Hi, my name is name_tk_j0k1l2 and my email is email_tk_a1b2c3"
```

The tokenization service scans chat content and replaces PII before it reaches the agent.

### Corner Cases

Some agents may need partial PII access for specific functions:

```json
{
  "agent": "jurisdiction-agent",
  "capabilities": {
    "view_state": true,      // Can see state/province
    "view_country": true,    // Can see country
    "view_address": false    // Cannot see street address
  }
}
```

This is handled through specialized base keys that unlock specific proxy endpoints.

### Audit Requirements

Even though agents can't see real PII, all access is logged:

```json
{
  "timestamp": "2024-03-15T14:22:33Z",
  "agent": "refund-agent",
  "task_key": "task_tk_xxx",
  "operation": "send_email",
  "tokens_used": ["email_tk_a1b2c3", "name_tk_j0k1l2"],
  "result": "sent"
}
```

Compliance teams can reconstruct what data was accessed by resolving tokens after the fact.

## The Privacy Boundary

```
┌─────────────────────────────────────────────────────────┐
│            REAL WORLD (Outside Silo)                     │
│                                                          │
│  - Real PII exists here                                  │
│  - Tokenization service lives here                       │
│  - Hydration happens here                                │
│  - Emails/SMS sent from here                             │
│                                                          │
└─────────────────────────────────────────────────────────┘
                         │
              ┌──────────▼──────────┐
              │   Privacy Boundary   │
              │   (Security Proxy)   │
              └──────────┬──────────┘
                         │
┌────────────────────────▼────────────────────────────────┐
│            TOKENIZED WORLD (Inside Silo)                 │
│                                                          │
│  - Agents live here                                      │
│  - Only tokens, never real PII                           │
│  - LLM calls happen here (with tokenized data)           │
│  - Full reasoning capability, zero PII exposure          │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

## Summary

| Principle | Implementation |
|-----------|----------------|
| Agents never see PII | All PII replaced with tokens |
| Tokens are opaque | No information derivable from token |
| Hydration happens outside | Real data only used by trusted services |
| Operations are privacy-preserving | Validation, comparison without exposure |
| Everything is audited | Token usage logged for compliance |

The result: Agents can reason about customer data, make decisions, compose communications—all without ever touching real personal information. If the agent, its logs, or its LLM calls are compromised, no PII is exposed.
