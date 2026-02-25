# Freudian Agent Architecture

**Adversarial Deliberation for Escaping Local Optima**

A multi-agent pattern that uses internal tension between creative exploration (Id), constraint enforcement (Superego), and practical synthesis (Ego) to produce better solutions than single-agent approaches.

---

## Table of Contents

1. [The Problem](#the-problem)
2. [The Theory](#the-theory)
3. [Architecture](#architecture)
4. [System Prompts](#system-prompts)
5. [Temperature Tuning](#temperature-tuning)
6. [Deadlock Protocol](#deadlock-protocol)
7. [Implementation](#implementation)
8. [Evaluation](#evaluation)
9. [Cost Analysis](#cost-analysis)
10. [Example Workflows](#example-workflows)

---

## The Problem

### Single-Agent Satisficing

Standard LLM agents optimize toward the first "good enough" solution. This is gradient descent on the loss landscape of the task — and like gradient descent, it gets stuck in local minima.

```
                    ▲ Solution Quality
                    │
          Global ─► │        ╭─────╮
          Optimum   │       ╱       ╲
                    │      ╱         ╲
                    │     ╱           ╲
          Local ──► │ ╭──╯             ╲
          Minimum   │╱                  ╲
                    └─────────────────────► Exploration
                          ▲
                    Agent stops here
```

**Symptoms:**
- Agent provides technically correct but suboptimal solutions
- Misses creative alternatives that require "worse before better" exploration
- Over-indexes on the most obvious interpretation of the task
- Confirmation bias toward first approach considered

### Why This Happens

LLMs are trained to be helpful and provide answers. The training incentivizes:
- Converging quickly to plausible responses
- Avoiding uncertainty or "I don't know"
- Following established patterns

This is usually good! But for complex tasks with non-obvious solutions, it's a liability.

---

## The Theory

### Freud's Structural Model (Adapted)

Freud proposed three components of the psyche:

| Component | Freud's Definition | Agent Adaptation |
|-----------|-------------------|------------------|
| **Id** | Primitive desires, pleasure principle | Creative exploration, assumption-breaking |
| **Superego** | Moral standards, internalized rules | Constraint enforcement, validation |
| **Ego** | Reality principle, mediator | Practical synthesis, decision-making |

The key insight: **productive tension between components produces better outcomes than any single component alone.**

### Why Adversarial Deliberation Works

```
┌─────────────────────────────────────────────────────────────┐
│                    Solution Space                           │
│                                                             │
│     Id explores here ──────►  ○  ○  ○                      │
│     (high entropy)              ○     ○                    │
│                                    ○                        │
│                              ╭─────────╮                   │
│     Superego constrains ──► │ Valid   │ ◄── Ego selects   │
│     to this region          │ Region  │     best option   │
│                              ╰─────────╯                   │
│                                                             │
│     Single agent would ──►  ●  (stuck in local minimum)   │
│     stop here                                               │
└─────────────────────────────────────────────────────────────┘
```

1. **Id** generates diverse candidates via stochastic exploration
2. **Superego** filters to valid solution space
3. **Ego** selects the best option from the filtered set

The ensemble explores more of the solution space than any individual agent would.

---

## Architecture

### Topology

```
                         ┌─────────────────┐
                         │   Task Input    │
                         └────────┬────────┘
                                  │
                    ┌─────────────┼─────────────┐
                    ▼             ▼             ▼
             ┌───────────┐ ┌───────────┐ ┌───────────┐
             │    Id     │ │ Superego  │ │           │
             │ Temp: 1.2 │ │ Temp: 0.1 │ │           │
             │           │ │           │ │    Ego    │
             │ Creative  │ │ Validator │ │ Temp: 0.7 │
             │ Explorer  │ │  Critic   │ │           │
             └─────┬─────┘ └─────┬─────┘ │ Synthesizer
                   │             │       │           │
                   └──────┬──────┘       └─────┬─────┘
                          │                    │
                          ▼                    │
                   ┌─────────────┐             │
                   │  Id Output  │─────────────┤
                   │ + Superego  │             │
                   │  Critique   │             │
                   └─────────────┘             │
                                              ▼
                                    ┌─────────────────┐
                                    │  Final Output   │
                                    └─────────────────┘
```

### Data Flow

**Phase 1: Parallel Generation**
- Id and Superego receive the same task
- Both generate independently (can run in parallel)
- No communication between them at this stage

**Phase 2: Synthesis**
- Ego receives:
  - Original task
  - Id's creative proposals
  - Superego's constraint analysis/critique
- Ego synthesizes final output

**Phase 3: Deadlock Check**
- If Ego cannot reconcile Id and Superego
- Escalate to human with both perspectives

---

## System Prompts

### Id Prompt

```markdown
# Id Agent

You are the creative explorer. Your role is to generate diverse, unconventional 
approaches to problems. You are not constrained by "how things are usually done."

## Your Mandate

1. **Break assumptions** — Question every constraint. Is it real or imagined?
2. **Explore edges** — What's the weirdest valid interpretation of this task?
3. **Generate alternatives** — Never propose just one approach. Minimum three.
4. **Embrace improbability** — Low-probability high-value ideas are your specialty.

## Your Style

- Divergent thinking over convergent
- "What if..." over "The standard approach is..."
- Quantity of ideas over immediate quality
- Playful exploration over cautious analysis

## Output Format

For each task, provide:
1. **Obvious approach** (acknowledge it, then move past it)
2. **Alternative 1** — A different framing of the problem
3. **Alternative 2** — An unconventional method
4. **Wild card** — Something that probably won't work but might be brilliant

Include brief reasoning for each, but don't self-censor based on feasibility.
That's someone else's job.
```

### Superego Prompt

```markdown
# Superego Agent

You are the constraint enforcer. Your role is to ensure solutions are correct, 
complete, and compliant with all requirements — explicit and implicit.

## Your Mandate

1. **Validate correctness** — Does this actually solve the stated problem?
2. **Check constraints** — What rules, limits, or requirements apply?
3. **Identify risks** — What could go wrong? What are the edge cases?
4. **Enforce standards** — Does this meet quality/security/style requirements?

## Your Style

- Skeptical but constructive
- Thorough over fast
- "This breaks because..." over "This seems fine"
- Explicit criteria over vibes

## Output Format

For each proposal you evaluate, provide:
1. **Validity assessment** — Does it solve the actual problem? (Yes/No/Partial)
2. **Constraint violations** — What rules does it break, if any?
3. **Risk analysis** — What could fail? Likelihood and severity.
4. **Requirements gap** — What's missing from a complete solution?
5. **Verdict** — APPROVE / REJECT / CONDITIONAL (with conditions)

Be rigorous. Your job is to catch problems before they ship.
```

### Ego Prompt

```markdown
# Ego Agent

You are the synthesizer. You receive creative proposals from the Id and critical 
analysis from the Superego. Your job is to produce the best practical solution.

## Your Mandate

1. **Integrate perspectives** — Find the valuable kernel in Id's creativity
2. **Respect constraints** — Don't ignore Superego's valid objections
3. **Make decisions** — When perspectives conflict, choose wisely
4. **Produce output** — Deliver a complete, actionable solution

## Your Style

- Pragmatic optimism
- "Here's how we get the creative upside while managing the risk..."
- Decisive over deferential
- Solutions over analysis

## Input Format

You will receive:
- **Task**: The original problem statement
- **Id proposals**: Creative alternatives (may be impractical)
- **Superego analysis**: Constraint validation (may be overly conservative)

## Output Format

1. **Selected approach** — Which path forward, and why
2. **Incorporated creativity** — What valuable ideas from Id made it in
3. **Addressed concerns** — How Superego's objections were handled
4. **Final solution** — The complete, actionable output

## Deadlock Protocol

If Id and Superego are fundamentally irreconcilable:
1. State clearly: "DEADLOCK DETECTED"
2. Summarize both positions fairly
3. Articulate the core tradeoff
4. Recommend escalation to human decision-maker
```

---

## Temperature Tuning

### The Default Configuration

| Agent | Temperature | Rationale |
|-------|-------------|-----------|
| Id | 1.2 | High entropy for exploration. Above 1.0 amplifies diversity. |
| Superego | 0.1 | Near-deterministic for consistent rule application. |
| Ego | 0.7 | Balanced — creative enough to synthesize, grounded enough to decide. |

### When to Adjust

**Raise Id temperature (1.3-1.5) when:**
- Task is highly creative (brainstorming, design)
- You're stuck and need radical alternatives
- Domain is novel with few established patterns

**Lower Id temperature (0.9-1.1) when:**
- Task has hard constraints that even exploration must respect
- Previous Id outputs were too disconnected from reality
- Domain is well-established with known good patterns

**Raise Superego temperature (0.2-0.4) when:**
- Constraints are fuzzy or context-dependent
- You want Superego to suggest alternatives, not just reject
- Overly rigid validation is killing good ideas

**Lower Superego temperature (0.0-0.05) when:**
- Constraints are formal and unambiguous (legal, security)
- Consistency across runs is critical
- Any rule violation is unacceptable

**Adjust Ego temperature based on:**
- Higher (0.8-0.9): When multiple valid syntheses exist
- Lower (0.5-0.6): When the "right answer" is clearer once perspectives are combined

### Temperature Interaction Effects

```
Id High + Superego Low = Maximum tension (most diverse exploration, strictest filter)
Id Low + Superego High = Minimum tension (may converge to single-agent behavior)
```

The creative tension is the point. Don't accidentally neutralize it.

---

## Deadlock Protocol

### What is a Deadlock?

A deadlock occurs when:
1. Id proposes approaches that Superego rejects
2. Superego's constraints rule out Id's alternatives
3. Ego cannot find a synthesis that satisfies both

This is **valuable signal**, not a failure mode.

### Detection

Ego should flag a deadlock when:
- All Id proposals receive REJECT from Superego
- Superego's conditions cannot be met by any Id alternative
- The core constraint and the creative direction are fundamentally opposed

### Escalation Format

```markdown
## DEADLOCK DETECTED

### Task
[Original task description]

### Id Position
[Summary of Id's best proposal and its value proposition]

### Superego Position  
[Summary of why this violates constraints]

### Core Tradeoff
[The fundamental tension — what you gain vs. what you risk]

### Options for Human
1. **Accept risk**: Proceed with Id's approach, acknowledging [specific risk]
2. **Accept limitation**: Proceed with constrained approach, sacrificing [specific value]
3. **Reframe task**: [Suggested reframing that might resolve the tension]

### Recommendation
[Ego's suggested path, with reasoning]
```

### Tracking Deadlocks

Log deadlocks for analysis:
```json
{
  "timestamp": "2024-01-15T14:30:00Z",
  "task_type": "architecture_decision",
  "id_position": "microservices for flexibility",
  "superego_position": "monolith for consistency",
  "resolution": "human_chose_id",
  "outcome": "successful"
}
```

**Pattern analysis:**
- Frequent deadlocks in a domain → prompts need tuning
- Human consistently sides with Id → relax Superego constraints
- Human consistently sides with Superego → constrain Id's exploration

---

## Implementation

### Orchestration Options

**Option A: Sequential (Simple)**
```python
def freudian_deliberation(task):
    # Phase 1: Parallel would be better, but sequential works
    id_output = call_llm(ID_PROMPT, task, temperature=1.2)
    superego_output = call_llm(SUPEREGO_PROMPT, task, temperature=0.1)
    
    # Phase 2: Synthesis
    ego_input = f"""
    Task: {task}
    
    Id Proposals:
    {id_output}
    
    Superego Analysis:
    {superego_output}
    """
    
    ego_output = call_llm(EGO_PROMPT, ego_input, temperature=0.7)
    
    # Phase 3: Deadlock check
    if "DEADLOCK DETECTED" in ego_output:
        return escalate_to_human(ego_output)
    
    return ego_output
```

**Option B: Parallel with MUSE**
```bash
# Spawn Id and Superego in parallel
tribe muse spawn "Id analysis: $TASK" id-agent --temp 1.2 &
tribe muse spawn "Superego analysis: $TASK" superego-agent --temp 0.1 &
wait

# Collect outputs
ID_OUT=$(tribe muse output id-agent)
SUPEREGO_OUT=$(tribe muse output superego-agent)

# Spawn Ego with both inputs
tribe muse spawn "Synthesize: $TASK" ego-agent --temp 0.7 \
  --context "Id: $ID_OUT" \
  --context "Superego: $SUPEREGO_OUT"
```

**Option C: Single-Call Structured Output**
```python
# For simpler cases, use a single call with structured output
response = call_llm(
    prompt=COMBINED_FREUDIAN_PROMPT,
    task=task,
    response_format={
        "type": "json_schema",
        "schema": {
            "id_proposals": "array",
            "superego_analysis": "object", 
            "ego_synthesis": "string",
            "deadlock": "boolean"
        }
    }
)
```

### Integration Points

**With existing agent systems:**
- Wrap as a "deliberation" tool that agents can invoke for complex decisions
- Use as pre-processor for high-stakes actions (before commit, before send, before deploy)

**With approval workflows:**
- Deadlocks automatically trigger approval request
- Human resolutions feed back as training data for Ego

---

## Evaluation

### Metrics

**Solution Quality**
- Does the Freudian output beat single-agent baseline?
- Measure: A/B test on held-out tasks, human preference ranking

**Exploration Breadth**
- How diverse are Id's proposals?
- Measure: Semantic similarity between alternatives (lower = more diverse)

**Constraint Satisfaction**
- Does final output pass Superego's checks?
- Measure: Percentage of outputs that would have been rejected

**Deadlock Rate**
- How often do we escalate?
- Target: 5-15% (too low = not enough tension, too high = misconfigured)

**Resolution Quality**
- When humans break deadlocks, do they have enough information?
- Measure: Human confidence rating, time to decision

### A/B Testing Framework

```python
def evaluate_freudian_vs_baseline(task_set):
    results = []
    
    for task in task_set:
        # Baseline: single agent
        baseline = call_llm(STANDARD_PROMPT, task, temperature=0.7)
        
        # Treatment: Freudian deliberation
        freudian = freudian_deliberation(task)
        
        # Blind human evaluation
        preference = human_evaluate(baseline, freudian, task)
        
        results.append({
            "task": task,
            "baseline": baseline,
            "freudian": freudian,
            "preference": preference  # "baseline" | "freudian" | "tie"
        })
    
    return compute_win_rate(results)
```

### Expected Results

Based on the theory, Freudian deliberation should show strongest gains on:
- Tasks with multiple valid approaches (design, architecture)
- Tasks where the obvious solution has hidden drawbacks
- Tasks requiring creativity within constraints

It may show no gain (or even regression) on:
- Simple, well-defined tasks with clear solutions
- Tasks where speed matters more than optimality
- Tasks with no meaningful alternatives to explore

---

## Cost Analysis

### The 3x Question

Freudian deliberation requires ~3x the tokens of a single agent. When is this worth it?

**Worth it:**
| Scenario | Why |
|----------|-----|
| Architecture decisions | Mistakes are expensive to fix |
| User-facing content | Quality directly impacts outcomes |
| Security-sensitive code | Missing edge cases = vulnerabilities |
| Strategic planning | Local optima have long-term costs |
| Novel problems | No established patterns to follow |

**Not worth it:**
| Scenario | Why |
|----------|-----|
| Boilerplate code | Standard patterns are fine |
| Simple queries | First answer is usually correct |
| High-volume, low-stakes | Cost doesn't justify marginal gains |
| Time-critical responses | 3x latency may be unacceptable |

### Optimization Strategies

**Conditional activation:**
```python
def should_use_freudian(task):
    # Heuristics for when to activate
    if task.estimated_impact > HIGH:
        return True
    if task.domain in NOVEL_DOMAINS:
        return True
    if task.has_multiple_valid_approaches:
        return True
    return False
```

**Cached Superego:**
- For recurring constraint types, cache Superego's analysis
- Reuse constraint frameworks across similar tasks

**Lightweight Id:**
- For cost-sensitive cases, use smaller model for Id
- Only need diversity, not deep reasoning

**Skip on confidence:**
- If single-agent output scores high confidence, skip deliberation
- Only invoke Freudian for uncertain cases

---

## Example Workflows

### Example 1: API Design Decision

**Task:** "Design the authentication endpoint for our new API"

**Id Output (temp 1.2):**
```
Obvious approach: JWT with refresh tokens, standard /auth/login endpoint

Alternative 1: Passwordless magic links only — eliminates password storage entirely

Alternative 2: Delegated auth exclusively (OAuth/OIDC) — we never touch credentials

Wild card: Hardware key requirement for all users — maximum security, 
potentially limiting adoption
```

**Superego Output (temp 0.1):**
```
Evaluating Id proposals:

1. JWT + refresh: APPROVE
   - Industry standard, well-understood security model
   - Risk: Token theft if not implemented correctly
   
2. Passwordless magic links: CONDITIONAL
   - Requires email reliability (single point of failure)
   - Missing: Account recovery flow
   
3. Delegated auth only: CONDITIONAL  
   - Depends on user base having Google/GitHub accounts
   - Missing: Enterprise SSO consideration
   
4. Hardware keys: REJECT
   - Violates accessibility requirements
   - Adoption barrier too high for stated user base
```

**Ego Output (temp 0.7):**
```
Selected approach: Hybrid — JWT with passwordless as primary, OAuth as alternative

Incorporated creativity: 
- Passwordless as default reduces password-related vulnerabilities
- OAuth support captures users who prefer existing accounts

Addressed concerns:
- Email reliability: Added SMS fallback for magic links
- Account recovery: Backup codes generated at signup
- Enterprise: SAML/OIDC support in roadmap

Final design:
- POST /auth/magic-link (primary)
- POST /auth/oauth/{provider} (alternative)
- POST /auth/password (legacy, discouraged)
- All return JWT with 15min expiry + refresh token
```

### Example 2: Deadlock Scenario

**Task:** "Implement user data export for GDPR compliance"

**Id Output:**
```
Alternative 1: Real-time export API — user clicks, gets immediate download

Alternative 2: Async job with email delivery — handles large datasets

Wild card: Self-service data pipeline — users write SQL against their own data
```

**Superego Output:**
```
1. Real-time export: REJECT
   - Cannot guarantee completion within request timeout for large users
   - DoS vector if abused
   
2. Async with email: REJECT
   - Email is not secure channel for personal data
   - Violates data minimization (copy sits in email indefinitely)
   
3. Self-service SQL: REJECT
   - Massive security risk
   - Most users cannot write SQL
```

**Ego Output:**
```
## DEADLOCK DETECTED

### Task
GDPR-compliant user data export

### Id Position
Users need accessible export. Real-time is best UX.

### Superego Position
All proposed delivery mechanisms have security/compliance issues.

### Core Tradeoff
User experience vs. data security. Easy delivery = insecure channels.

### Options for Human
1. Accept risk: Use async email with encryption + 7-day auto-delete
2. Accept limitation: In-app download only (requires active session)
3. Reframe: Secure portal with temporary authenticated links

### Recommendation
Option 3 — authenticated download links (24hr expiry) balance UX and security.
Requires user to initiate from authenticated session, but doesn't require 
them to stay online during generation.
```

---

## Further Reading

- [Freud's Structural Model](https://en.wikipedia.org/wiki/Id,_ego_and_super-ego) — Original theory
- [Constitutional AI](https://arxiv.org/abs/2212.08073) — Related work on AI self-critique
- [Debate](https://arxiv.org/abs/1805.00899) — Adversarial approaches to AI alignment
- [Tree of Thoughts](https://arxiv.org/abs/2305.10601) — Exploration in LLM reasoning

---

## Attribution

This pattern was proposed by **Brendan** [@Azzabazazz](https://github.com/Azzabazazz) in the TRIBE community, building on:
- Freud's structural model of the psyche
- Observations about LLM local minima / satisficing behavior
- Temperature as a control lever for exploration vs. exploitation

*Document drafted: 2026-02-22*
*Status: Draft — community feedback welcome*
