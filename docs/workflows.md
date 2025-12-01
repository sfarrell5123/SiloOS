# SiloOS Workflows

## This Replaces n8n

n8n, Zapier, Make, and friends have the same problem: **LLM in the control plane**.

You draw a visual graph, hope the AI in the middle makes good decisions, and pray it's auditable. It's not. It's:

- **Not inspectable**: Visual graphs hide complexity
- **Not testable**: How do you unit test a flowchart?
- **Not secure**: AI making routing decisions outside any sandbox
- **Not auditable**: Good luck explaining that to compliance
- **Hard to version**: Try diffing a JSON blob of node positions

MCP has the same disease: bloated, opaque, AI-in-the-wiring.

**SiloOS workflows are the opposite:**

- YAML text files (diff-able, reviewable)
- Router is dumb code (no AI, fully testable)
- AI only runs inside padded cells (agents)
- Everything logged, everything explicit

No low-code spaghetti. No hidden AI in the control plane.

---

## The Router Stays Dumb

The router is outside the jail cells. It's outside the security boundary. So we don't run AI in the router.

**The router is old-school code:**
- Fully deterministic
- Fully testable
- Fully inspectable
- Fully auditable

No LLM. No "creativity." No surprises.

```
Router's job:
├── Receive events
├── Look up the workflow
├── Assign the next agent
├── Mint task keys
├── Pass along workflow state
├── Log everything
└── Handle escalations
```

Think Kafka Streams meets a rules engine. Pure logic + config.

## Workflows Are Declarative Pipelines

A workflow isn't code. It's a YAML definition that says "run these agents in order":

```yaml
# workflows/publish-ebook.yaml

name: publish-ebook
description: Research, create, and publish an ebook across platforms

stages:
  - agent: prethink-agent
  - agent: research-agent
  - agent: dtp-agent
  - agent: ebook-agent
  - agent: wordpress-publisher
  - agent: twitter-publisher
  - agent: linkedin-publisher

on_error:
  400: human-agent    # Human-in-the-loop request
  500: abort          # Kill the workflow
```

The router loads this definition and marches through it like a pipeline.

**No agent needs to know it's part of a workflow.**

They're just individual workers that get handed a task with context. They do their job and return a result. The router handles the sequencing.

## Workflow State

Agents are stateless. But workflows need continuity. The solution: **workflow state** that travels with the task key.

### What Gets Passed

Each agent receives:

```
/workflow/
├── task_key.jwt           # Binds to this workflow run
├── base_keys.jwt          # Agent's permanent authority
├── state/
│   ├── interim/           # Documents from previous agents
│   │   ├── research.md
│   │   ├── outline.json
│   │   └── draft.html
│   ├── summary.json       # What the last agent did
│   └── log.txt            # Workflow history
```

Each agent produces:

```python
return {
    "status": 200,              # 200, 400, or 500
    "summary": {
        "what_i_did": "Completed research on topic X",
        "what_i_produced": ["research.md", "sources.json"],
        "what_still_needs_doing": "Needs editorial review",
        "warnings": []
    },
    "interim_assets": [         # Files to pass forward
        "research.md",
        "sources.json"
    ]
}
```

The router reads this and decides the next step.

### Interim Assets Live Under the Task Key

Workflow assets are stored in a proxy-backed store, scoped to the task key:

- **Isolated**: No agent can peek at other workflows
- **Temporary**: Garbage-collected when workflow ends
- **Controlled**: Access through proxy, validated by keys

This fits the SiloOS security model perfectly. The workflow state is just another form of task-scoped data.

## Agents Don't Know About Each Other

Agents don't "talk" to each other. They don't even know who came before or who comes after.

**Wrong approach:**
```python
# Agent tries to orchestrate
next_agent = decide_next_step()
call_agent(next_agent, my_data)  # NO!
```

**Right approach:**
```python
# Agent does its job and returns
return {
    "status": 200,
    "summary": {"what_i_did": "...", "what_still_needs_doing": "..."},
    "interim_assets": ["output.md"]
}
# Router decides what happens next
```

Each agent produces a **neutral summary**:
- What it did
- What it produced
- What still needs doing
- Any warnings or flags

The next agent sees that summary and interprets it using its own markdown playbooks. No coupling. No emergent chaos.

## Status Codes

Agents return simple status codes. The router acts deterministically.

| Code | Meaning | Router Action |
|------|---------|---------------|
| `200` | Success | Pass to next stage |
| `302` | Redirect | Jump to named agent/workflow (must be in allowed list) |
| `400` | Human needed | Route to human-agent, then continue |
| `500` | Failed | Abort workflow |

No ambiguity. No negotiation.

```yaml
# Workflow handles errors declaratively
on_error:
  400: human-agent
  500: abort
```

### 400: Human-in-the-Loop

Agent hits something it can't handle:

```python
return {
    "status": 400,
    "summary": {
        "what_i_did": "Attempted to generate cover image",
        "problem": "Content policy violation on DALL-E",
        "needs_human": "Please provide alternative image or approve bypass"
    }
}
```

Router routes to `human-agent`. Human resolves. Workflow continues.

### 500: Abort

Something is fundamentally broken:

```python
return {
    "status": 500,
    "summary": {
        "what_i_did": "Attempted to fetch research sources",
        "error": "All API endpoints returned 503",
        "recommendation": "Retry workflow later"
    }
}
```

Router aborts. Logs everything. Alerts operator.

### 302: Redirect

An agent can request a redirect to a different agent or workflow—but only if it's in the allowed list:

```python
return {
    "status": 302,
    "redirect": "calendar-agent",  # Must be in allowed_redirects
    "summary": {
        "what_i_did": "Classified intent as calendar request",
        "confidence": 0.94
    }
}
```

The workflow must explicitly allow this:

```yaml
stages:
  - agent: intent-agent
    allowed_redirects:
      - calendar-agent       # Jump to a single agent (most common)
      - research-agent
      - support-agent
      - escalate-workflow    # Or jump to an entire workflow
```

Redirects can target:
- **An agent** (typical): Run that agent, then continue this workflow
- **A workflow** (less common): Exit this workflow, enter the other one

Either way, it must be in `allowed_redirects`. The agent suggests. The router validates against the whitelist. No agent can redirect somewhere unexpected.

## Decision Agents

The popular n8n pattern: "Message arrives → LLM decides what to do → routes to calendar or info lookup or whatever."

In SiloOS, that becomes a **decision agent**:

```yaml
# workflows/incoming-message.yaml

name: incoming-message
stages:
  - agent: intent-agent
    allowed_redirects:
      - calendar-agent
      - research-agent
      - order-agent
      - support-agent

  # After redirect, continue with response
  - agent: response-agent

on_error:
  400: human-agent
  500: abort
```

### How It Works

1. **intent-agent** runs in its padded cell:
   - Reads the incoming message
   - Reads its markdown about available capabilities
   - Outputs: `{ status: 302, redirect: "calendar-agent", confidence: 0.94 }`

2. **Router** (dumb code):
   - Checks: Is "calendar-agent" in `allowed_redirects`? Yes.
   - Runs `calendar-agent`

3. **calendar-agent** does its thing, returns 200

4. **Workflow continues** to `response-agent`

The decision is AI-generated. The branching is old-school code reading a string field and checking a whitelist.

### Low Confidence? Human.

```python
# intent-agent returns
return {
    "status": 302 if confidence > 0.8 else 400,
    "redirect": "calendar-agent" if confidence > 0.8 else None,
    "summary": {
        "intent": "calendar",
        "confidence": 0.62,  # Too low
        "needs_human": "Unclear if this is a calendar request or general question"
    }
}
```

Router sees 400 → routes to human. Human clarifies. Workflow continues.

## Atomic Workflows, Not Giant Graphs

Instead of one massive n8n flowchart, you get small focused workflows:

```
workflows/
├── incoming-message.yaml      # Intent classification → redirect
├── calendar-workflow.yaml     # Handle calendar requests
├── research-workflow.yaml     # Research and respond
├── order-workflow.yaml        # Order lookup and support
├── support-workflow.yaml      # General support
└── publish-workflow.yaml      # Content publishing pipeline
```

Each is just a list of agents + simple error handling.

The only "graphy" bit: `incoming-message` can jump into one of the others based on intent-agent's output.

**Branching stays:**
- Explicit (listed in YAML)
- Testable (unit test "intent X → workflow Y")
- Diff-able in Git
- No hidden flows inside a visual editor

## Example: Publishing Pipeline

```yaml
# workflows/publish-article.yaml

name: publish-article
stages:
  - agent: prethink-agent      # Frame the topic, identify angles
  - agent: research-agent      # Gather sources, facts, quotes
  - agent: writer-agent        # Draft the article
  - agent: editor-agent        # Polish, fact-check
  - agent: dtp-agent           # Format as HTML/PDF
  - agent: wordpress-agent     # Publish to blog
  - agent: twitter-agent       # Create thread, post
  - agent: linkedin-agent      # Format and post

on_error:
  400: human-agent
  500: abort
```

### How It Flows

```
1. prethink-agent
   ├── Receives: Initial topic/prompt
   ├── Produces: angles.md, hooks.json
   └── Returns: 200

2. research-agent
   ├── Receives: angles.md, hooks.json
   ├── Produces: research.md, sources.json
   └── Returns: 200

3. writer-agent
   ├── Receives: angles.md, research.md, sources.json
   ├── Produces: draft.md
   └── Returns: 200

4. editor-agent
   ├── Receives: draft.md, sources.json
   ├── Produces: final.md, fact-check.json
   └── Returns: 200

5. dtp-agent
   ├── Receives: final.md
   ├── Produces: article.html, article.pdf
   └── Returns: 200

6. wordpress-agent
   ├── Receives: article.html
   ├── Produces: published_url.txt
   └── Returns: 200

7. twitter-agent
   ├── Receives: final.md, published_url.txt
   ├── Produces: thread.json, tweet_ids.txt
   └── Returns: 200

8. linkedin-agent
   ├── Receives: final.md, published_url.txt
   ├── Produces: post_id.txt
   └── Returns: 200

WORKFLOW COMPLETE
```

Each agent is atomic. Each agent sees only what it needs. Each agent leaves structured breadcrumbs for the next.

The router just marches forward.

## Workflow State Storage

Interim assets need to persist across agent runs but die when the workflow ends.

```
/workflows/
└── run_abc123/              # Workflow instance
    ├── manifest.json        # Current stage, status, timing
    ├── state/
    │   ├── interim/         # Accumulating assets
    │   │   ├── angles.md
    │   │   ├── research.md
    │   │   ├── draft.md
    │   │   └── final.md
    │   ├── summaries/       # Each agent's summary
    │   │   ├── 01-prethink.json
    │   │   ├── 02-research.json
    │   │   └── ...
    │   └── log.txt          # Full workflow log
    └── output/              # Final deliverables
        ├── article.html
        ├── article.pdf
        └── published_urls.json
```

When workflow completes:
- `output/` is preserved (or pushed to permanent storage)
- Everything else is garbage-collected

## The Three Tiers

SiloOS workflows create three clean architectural layers:

### 1. Router (Deterministic, No AI)

```
Events → Workflow lookup → Next agent → Key minting → Logging
```

Old-school code. Fully testable. Fully auditable.

### 2. Workflow State (Proxy-backed, Task-scoped)

```
Interim assets, summaries, logs, metadata
```

Travels with the task key. Garbage-collected on completion.

### 3. Agents (Stateless AI Workers)

```
Think freely → Act through proxies → Read/write only workflow state
```

No knowledge of the pipeline. Just do the job.

## What This Avoids

| Anti-pattern | How SiloOS Avoids It |
|--------------|---------------------|
| Graph spaghetti | Linear pipeline, declarative YAML |
| Agent-to-agent coupling | Agents don't know each other exist |
| LLM in control plane | Router is pure deterministic code |
| Stateful agents | State lives in workflow, not agent |
| Monolithic workflows | Each agent is atomic, replaceable |
| Hidden coupling | Everything explicit in YAML + summaries |
| Undebuggable behavior | Full logging, deterministic routing |

## Summary

| Component | Role |
|-----------|------|
| **Router** | Dumb orchestrator. No AI. Loads YAML, marches through stages. |
| **Workflow** | Declarative pipeline. List of agents + error handling + allowed redirects. |
| **Workflow State** | Interim assets + summaries. Scoped to task key. |
| **Agent** | Stateless worker. Does its job. Returns status + summary. |
| **Decision Agent** | Classifies intent, returns 302 + redirect target from allowed list. |
| **Status Codes** | 200 (next), 302 (redirect), 400 (human), 500 (abort). |

Agents stay in their cells. The router stays deterministic. Workflows chain them together without coupling.

**This is what n8n should have been:** text files, dumb routing, AI only in the sandboxes.

*Simple. Inspectable. Debuggable. Safe.*
