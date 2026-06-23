# Domain 1 — Agentic Architecture & Orchestration (27%)

**Exam weight: 27% (heaviest single domain). Pass cut: 720/1000.**

---

## Three Global Rules That Resolve Most Questions

1. **Deterministic > probabilistic when stakes are real.** Financial / security / compliance / safety → hooks or programmatic gates. Style / tone / formatting → prompts.
2. **Trace failures to their origin.** Scope problems = upstream (decomposition / partitioning / context-passing). Quality problems = subagent prompts / tools. Don't blame the agent that faithfully executed a bad brief.
3. **Proportionate fixes.** Single agent beats multi-agent when single suffices. Prompts beat hooks when stakes are low. Don't over-engineer.

---

## Key Concepts

| Concept | Definition | Exam Signal |
|---|---|---|
| **`stop_reason: "tool_use"`** | Model wants to call a tool — loop continues: execute, append results, call again | "continue or stop?" |
| **`stop_reason: "end_turn"`** | Model has finished — present result to user, loop terminates | "loop termination signal" |
| **Task Tool** | SDK mechanism for spawning subagents; `allowedTools` must include `"Task"` | "won't delegate" bug |
| **Subagent Context Isolation** | Subagents do NOT inherit coordinator history — every byte they need must be in their prompt | "what do subagents inherit?" |
| **Parallel Subagents** | Multiple `Task` tool calls in ONE coordinator response — SDK executes concurrently | "how to parallelize?" |
| **Hub-and-Spoke** | ALL inter-subagent communication routes through coordinator for observability | "direct agent-to-agent comms?" → No |
| **Programmatic Enforcement** | Code-level prerequisites block downstream calls until prerequisite completes — 100% compliance | "vs prompt instructions" |
| **Prompt-Based Guidance** | Non-zero failure rate (95–99%) — insufficient when single failure = financial/safety harm | "sufficient for critical flow?" → No |
| **`fork_session`** | Independent branches from a shared expensive baseline; explore N alternatives | "parallel exploration" |
| **`--resume <session-name>`** | Continues a specific prior named session | "resume prior work" |
| **PostToolUse Hook** | Fires after tool, before model sees result — normalise formats, redact PII, truncate | "heterogeneous formats" |
| **PreToolUse Hook** | Fires before tool executes — gates, authorization, prerequisite checks, **prior state hash** | "prior state", "audit", "block if" |
| **Idempotency Key** | UUID generated on FIRST attempt, passed on every retry — server-side dedup prevents double-charge | "transient retry risk" |

---

## The Agentic Loop (Know Cold)

```python
messages = [{"role": "user", "content": user_input}]
while True:
    response = client.messages.create(model=MODEL, tools=tools, messages=messages)
    
    if response.stop_reason == "end_turn":
        print(response.content[0].text)
        break                          # ← only legitimate exit
    
    if response.stop_reason == "tool_use":
        tool_results = execute_tools(response.content)
        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user",      "content": tool_results})
        # loop continues ↑
```

**Three anti-patterns (always wrong answers):**
- Parsing natural language for "I'm done" phrases to determine termination
- Using iteration caps as the *primary* stopping mechanism (caps are safety backstops)
- Checking `content[0].type == "text"` — tool_use responses can begin with a text block

---

## Enforcement Spectrum

```
PROBABILISTIC ───────────────────────────► DETERMINISTIC
prompt rules → few-shot → routing → hooks / prerequisite gates
  ~95%           ~98%       ~98%         100%
```

**Decision rule (memorise):**
> Single failure causes financial loss, security breach, or safety harm → **hooks / programmatic prerequisite**.  
> Single failure causes bad-but-recoverable UX → **prompts**.

**Canonical high-stakes scenarios → answer is always hooks:**
refund verification, trade limits, compliance checks, identity gates, age gates, audit logging.

**The four-distractor pattern:** hook ← correct | stronger prompt | few-shot examples | specialised subagent.

---

## Task Decomposition Patterns

| Pattern | When | Discriminator |
|---|---|---|
| **Fixed Sequential Pipeline** | Stable structure, each step consumes prior step's output | "every X follows the same flow" |
| **Dynamic Decomposition** | Varying requests; typed subagents (search/doc/synth) | "same roles, scope varies per query" |
| **Orchestrator–Worker** | Uniform workers differentiated only by prompt-supplied scope | "N workers, each a different sub-topic" |
| **Evaluator–Optimiser** | Explicit checkable quality criteria; first pass often inadequate | "iterate until coverage/format passes" |

**Evaluator–Optimiser fails when** criteria aren't objectively checkable — subjective criteria → loop spins indefinitely.

---

## Hooks: PreToolUse vs PostToolUse

| Hook | Fires | Use Cases |
|---|---|---|
| **PreToolUse** | Before tool executes | Gates, authorisation, prerequisite checks, financial limits, **capturing prior state hash** for audit |
| **PostToolUse** | After tool, before model sees result | Normalisation (timestamps, currencies, field names), redaction (PII), truncation, enrichment |

**Keyword triggers:**
- "Prior state" / "pre-condition" / "before X happens" → **PreToolUse**
- "Heterogeneous formats" / "normalise" / "redact" / "model confused by varying outputs" → **PostToolUse**

**Hooks don't rank tools.** Hooks gate, block, or transform. Use prompts + tool descriptions to prefer one tool over another.

---

## Error Classification

| Type | Examples | Response |
|---|---|---|
| **Transient** | 429, 503, network timeout | Retry with exponential backoff + **jitter**, capped retries |
| **Permanent** | 400, 401, 403, 404, malformed input | Don't retry — feed back to model or escalate |
| **Logical** | Tool returned empty / irrelevant results | Feed back to model with context; model changes strategy |

**Anti-pattern:** blanket retry on every error type. Always classify first.

**Idempotency key:** UUID generated on FIRST attempt, passed on every retry. Without it, a transient retry can double-charge. The **paired-fix pattern**: when both retry policy is broken AND duplication occurs → you need BOTH error classification AND idempotency keys.

---

## Failure Tracing Rules

| Symptom | Root Cause |
|---|---|
| Missing topics in output | Coordinator's **decomposition** too narrow |
| Duplicated work / redundant sources | Coordinator's **scope partitioning** sloppy |
| Shallow on every topic | Subagent **prompts** or tool budgets |
| Missing source attribution | **Context-passing** format (collapsed prose instead of claim-source mapping) |
| "Won't delegate" | Coordinator's `allowedTools` missing `"Task"` |

---

## Reliability Patterns

- **Circuit breaker** — N failures in window → stop calling during cooldown period
- **Fallback agent / tool** — degraded answer > no answer
- **Cost / token budgets** — enforced at harness, not by prompt
- **Per-tool timeouts** — no tool blocks the loop indefinitely

---

## Distractor-Recognition Cribsheet

**Reject these when stakes are high:**
- "Add a stronger system prompt instruction…"
- "Include few-shot examples demonstrating the correct sequence…"
- "Route to a specialised subagent whose prompt emphasises…"
- "Forward the full conversation transcript to the human reviewer"
- "Catch errors and return an empty string silently"
- "Retry on every error type with the same delay"
- "Have subagent A pass results directly to subagent B"

**Lean toward these:**
- "PreToolUse hook that blocks…" / "PostToolUse hook that normalises…"
- "Idempotency key generated on first attempt"
- "Pass structured data with claim → source mappings"
- "Classify errors and handle each category appropriately"
- "Coordinator's decomposition step is too narrow" (for missing-topic scenarios)
- "Add `Task` to coordinator's `allowedTools`" (for "won't delegate" bugs)
- "Bounded timeout with fallback message" (for latency hangs)

---

## One-Line Mnemonics

- **Loop termination:** `stop_reason` or it's wrong.
- **Subagent state:** nothing carries over. The prompt is the universe.
- **High stakes:** hooks, every time, no matter how strong the prompt wording.
- **Missing topics:** blame the coordinator's decomposition.
- **Duplicate work:** blame the coordinator's partitioning.
- **Missing citations:** blame the context-passing format.
- **Won't delegate:** check `allowedTools` for `"Task"`.
- **Prior state in audit:** PreToolUse only.
- **Heterogeneous tool outputs:** PostToolUse normaliser.
- **Double-charged customer:** idempotency key.
- **Evaluator–optimiser:** dies when criteria aren't objectively checkable.
