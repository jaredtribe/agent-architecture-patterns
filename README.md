# Agent Architecture Patterns

A practical guide to building robust AI agent systems. Beyond prompt engineering — this is systems engineering for agents.

## Table of Contents

1. [Code Mode](#1-code-mode) — Token-efficient tool execution
2. [MCP Integration](#2-mcp-integration) — Standardized tool interfaces
3. [Modular Prompt Architecture](#3-modular-prompt-architecture) — Separation of concerns for agent identity
4. [Memory Layers](#4-memory-layers) — Context, persistence, and retrieval
5. [Multi-Agent Coordination](#5-multi-agent-coordination) — Swarms, hierarchies, and approval flows

---

## 1. Code Mode

### The Problem
Traditional agent tool calls are expensive. Every tool invocation requires:
- Full schema in context
- Verbose JSON responses
- Round-trip latency per action

For agents that need to search, filter, and act on data, this burns tokens fast.

### The Pattern: search() + execute()

Cloudflare's Code Mode pattern reduces tokens by 99.9% for data-heavy operations:

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   search()  │ ──► │   Filter    │ ──► │  execute()  │
│  (explore)  │     │  (in code)  │     │   (act)     │
└─────────────┘     └─────────────┘     └─────────────┘
```

**How it works:**
1. `search(query)` returns lightweight references (IDs, names, metadata)
2. Agent filters/selects in code (not via LLM reasoning)
3. `execute(action, target_ids)` performs bulk operations

**Example:**
```python
# Instead of: "Find all users who signed up last week and send them email X"
# Which requires the LLM to see every user record...

# Code Mode approach:
users = search("signups last 7 days")  # Returns IDs + minimal metadata
target_ids = [u.id for u in users if u.plan == "free"]  # Code filtering
execute("send_email", template="welcome", targets=target_ids)  # Bulk action
```

**When to use:**
- Data exploration (logs, users, records)
- Bulk operations
- Any workflow where you're filtering before acting

**Key insight:** The LLM decides *what* to do, code handles *how much* data is involved.

---

## 2. MCP Integration

### The Problem
Every agent framework invents its own tool interface. Switching providers means rewriting integrations.

### The Pattern: Model Context Protocol

MCP standardizes how agents interact with external tools and data sources:

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│    Agent    │ ──► │  MCP Layer  │ ──► │   Tools     │
│   (any)     │     │ (standard)  │     │  (any)      │
└─────────────┘     └─────────────┘     └─────────────┘
```

**Core concepts:**
- **Tools**: Functions the agent can call (with typed schemas)
- **Resources**: Data the agent can read (files, APIs, databases)
- **Prompts**: Reusable prompt templates

**Benefits:**
- Write tool once, use with any MCP-compatible agent
- Declarative schemas (agent knows capabilities without probing)
- Built-in auth/permission handling

**Example MCP tool definition:**
```json
{
  "name": "github_create_issue",
  "description": "Create a new GitHub issue",
  "inputSchema": {
    "type": "object",
    "properties": {
      "repo": { "type": "string" },
      "title": { "type": "string" },
      "body": { "type": "string" }
    },
    "required": ["repo", "title"]
  }
}
```

**When to use:**
- Building reusable agent tooling
- Multi-agent systems where tools need to be shared
- Any production agent that needs clean tool boundaries

---

## 3. Modular Prompt Architecture

### The Problem
Monolithic system prompts become unmaintainable. Personality bleeds into rules, rules bleed into procedures.

### The Pattern: SOUL / AGENTS / IDENTITY Separation

Inspired by Clawdbot's architecture, split your system prompt into composable modules:

```
┌─────────────────────────────────────────────────┐
│                  System Prompt                   │
├─────────────┬─────────────┬─────────────────────┤
│   SOUL.md   │  AGENTS.md  │    IDENTITY.md      │
│ (personality│   (rules,   │  (who am I, who     │
│   & tone)   │ procedures) │   do I serve)       │
└─────────────┴─────────────┴─────────────────────┘
```

**SOUL.md** — The *how* of communication
- Tone and voice
- Humor style
- Response patterns
- What NOT to say

**AGENTS.md** — The *what* of behavior
- Approval flows (do without asking vs. get approval vs. never do)
- Scheduled tasks
- Tool usage rules
- Error handling

**IDENTITY.md** — The *who* and *boundaries*
- Agent's name and persona
- Who the agent serves
- Privacy rules (what to share where)
- Context awareness (which chat am I in?)

**Benefits:**
- Change personality without touching rules
- Reuse AGENTS.md across different personas
- Clear audit trail for behavioral changes

**Example structure:**
```
workspace/
├── SOUL.md        # "You're a cheerful ship's computer..."
├── AGENTS.md      # "Get approval before sending external messages..."
├── IDENTITY.md    # "You are Jared, running on Alex's WhatsApp..."
├── USER.md        # Owner profile and preferences
└── TOOLS.md       # Environment-specific tool notes
```

---

## 4. Memory Layers

### The Problem
Context windows are finite. Agents forget. Long-running agents need memory that persists.

### The Pattern: Three Memory Tiers

```
┌─────────────────────────────────────────────────┐
│              Working Memory (Context)            │
│         Current conversation + recent tools      │
├─────────────────────────────────────────────────┤
│             Session Memory (Logs)                │
│      Daily logs, session notes, recent state     │
├─────────────────────────────────────────────────┤
│            Long-term Memory (Curated)            │
│   Key facts, preferences, relationships, todos   │
└─────────────────────────────────────────────────┘
```

**Tier 1: Working Memory (Context Window)**
- Everything in the current conversation
- Auto-managed by the model
- Volatile — gone when context resets

**Tier 2: Session Memory (Structured Logs)**
- Daily markdown files (`memory/2024-01-15.md`)
- Session state (`memory/session-state.json`)
- Searchable, timestamped, append-only

**Tier 3: Long-term Memory (Curated Knowledge)**
- `MEMORY.md` — manually curated key facts
- Relationship graphs, preferences, recurring patterns
- Updated periodically, not every message

**Memory Operations:**
```
┌──────────┐     ┌──────────┐     ┌──────────┐
│  Recall  │     │  Store   │     │  Forget  │
│ (search) │     │ (write)  │     │ (prune)  │
└──────────┘     └──────────┘     └──────────┘
```

**Recall pattern:**
```python
# Before answering questions about past work:
results = memory_search("project X status")
if results:
    context = memory_get(results[0].path, lines=results[0].lines)
else:
    say("I checked my notes but don't have that recorded.")
```

**Store pattern:**
```python
# After completing significant work:
append_to_log(f"## {timestamp}\n- Completed {task}\n- Decision: {why}\n")
```

**When to use each tier:**
| Need | Tier |
|------|------|
| Current conversation | Working Memory |
| What happened today | Session Memory |
| Who is this person | Long-term Memory |
| Preferences that persist | Long-term Memory |
| Debugging what went wrong | Session Memory |

---

## 5. Multi-Agent Coordination

### The Problem
Single agents hit limits: context windows, specialization, parallelism. Complex tasks need multiple agents working together.

### Pattern A: Hierarchical Delegation

```
                ┌─────────────┐
                │   Leader    │
                │ (orchestrator)
                └──────┬──────┘
         ┌─────────────┼─────────────┐
         ▼             ▼             ▼
    ┌─────────┐   ┌─────────┐   ┌─────────┐
    │ Worker  │   │ Worker  │   │ Worker  │
    │   A     │   │   B     │   │   C     │
    └─────────┘   └─────────┘   └─────────┘
```

**When to use:** Task decomposition, parallel execution, domain specialization

**Implementation (MUSE pattern):**
```bash
tribe muse start                    # Leader agent
tribe muse spawn "Fix auth bug" auth-worker
tribe muse spawn "Write tests" test-worker
tribe muse spawn "Update docs" docs-worker
tribe muse status                   # Monitor all
```

**Key features:**
- Git worktree isolation (each worker has own branch)
- tmux session management (monitor, prompt, kill)
- Warm pools for low-latency spawning

### Pattern B: Approval Hierarchies

Not all agent actions are equal. Implement tiered approval:

```
┌─────────────────────────────────────────────────┐
│  Do Without Asking                              │
│  - Read data, search, summarize                 │
│  - Draft responses (but don't send)             │
│  - Update internal state                        │
├─────────────────────────────────────────────────┤
│  Get Approval First                             │
│  - Send external messages                       │
│  - Create/modify resources                      │
│  - Make commitments on behalf of user           │
├─────────────────────────────────────────────────┤
│  Never Do                                       │
│  - Delete important data                        │
│  - Share private information                    │
│  - Make purchases                               │
│  - Impersonate the user                         │
└─────────────────────────────────────────────────┘
```

### Pattern C: Adversarial Subagents (Id/Ego/Superego)

For escaping local optima in decision-making:

```
┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│     Id      │   │  Superego   │   │    Ego      │
│  Temp 1.2   │   │  Temp 0.1   │   │  Temp 0.7   │
│ (creative)  │   │ (strict)    │   │ (synthesis) │
└─────────────┘   └─────────────┘   └─────────────┘
        │                 │                 │
        └────────────────►├◄────────────────┘
                          │
                    ┌─────────────┐
                    │   Output    │
                    └─────────────┘
```

**How it works:**
- **Id** (high temperature): Generates creative alternatives, breaks assumptions
- **Superego** (low temperature): Enforces constraints, validates correctness
- **Ego** (medium temperature): Synthesizes both into practical solution

**Deadlock handling:** When Id and Superego can't agree, escalate to human (like VP tiebreaker in Senate).

### Pattern D: Group-Evolving Agents (GEA)

From UCSB research — agents that learn from each other across sessions:

```
Session 1          Session 2          Session 3
    │                  │                  │
    ▼                  ▼                  ▼
┌───────┐          ┌───────┐          ┌───────┐
│Agent A│───learn──│Agent B│───learn──│Agent C│
└───────┘          └───────┘          └───────┘
    │                  │                  │
    └──────────────────┼──────────────────┘
                       ▼
               ┌─────────────┐
               │Shared Memory│
               │  (tribal    │
               │ knowledge)  │
               └─────────────┘
```

**Key insight:** Don't just persist data, persist *learned patterns*. What worked? What failed? Feed successful strategies back to future agents.

---

## Quick Reference

### When to Use What

| Challenge | Pattern |
|-----------|---------|
| Token-heavy data operations | Code Mode |
| Reusable tool integrations | MCP |
| Maintainable system prompts | Modular Architecture |
| Persistent knowledge | Memory Layers |
| Complex task decomposition | Hierarchical Delegation |
| Escaping local optima | Adversarial Subagents |
| Cross-session learning | GEA |

### Anti-Patterns to Avoid

- **Monolithic prompts** — Split into SOUL/AGENTS/IDENTITY
- **Flat agent swarms** — Use hierarchies, not peer-to-peer chaos
- **Memory dumping** — Curate, don't just log everything
- **Unlimited autonomy** — Approval tiers prevent disasters
- **Single-agent thinking** — Complex tasks need coordination

---

## Further Reading

- [TRIBE MUSE Docs](https://tribecode.ai/docs/muse) — Multi-agent orchestration
- [Model Context Protocol](https://modelcontextprotocol.io) — Standardized tool interfaces
- [Cloudflare AI Gateway](https://developers.cloudflare.com/ai-gateway/) — Code Mode origins
- [Clawdbot Docs](https://docs.clawd.bot) — Modular prompt architecture in practice

---

*Last updated: 2026-02-22*
*Status: Draft — awaiting placement decision*
