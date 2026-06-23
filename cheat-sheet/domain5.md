# Domain 5: Context Management & Reliability (15%)

> ⚠️ This domain is NOT about safety/ethics. It's about managing tokens, errors, escalation, and reliability across multi-turn conversations and multi-agent systems.

---

## Key Concepts Table

| Concept | Definition | Exam Signal |
|---|---|---|
| **Lost-in-the-Middle Effect** | Models reliably process info at beginning and end of long inputs but may omit findings from middle sections | "position matters in long context" |
| **Tool Result Accumulation** | Tool outputs grow in context disproportionately to relevance (e.g., 40+ fields from a lookup when only 5 are needed) | "trim verbose outputs" |
| **Structured Error Context** | Error responses that include failure type, attempted query, partial results, and alternative approaches | "intelligent coordinator recovery" |
| **Escalation Triggers** | Explicit customer request for human, policy gaps/ambiguity, inability to make meaningful progress | "when to escalate" |
| **Progressive Summarization Risk** | Condensing exact numbers, dates, and customer-stated expectations into vague prose loses critical precision | "case facts must stay precise" |
| **Case Facts Block** | Persistent structured data (amounts, dates, order IDs) maintained separately from conversation summary | "preserve numeric precision" |
| **Scratchpad Files** | Files agents write to persist key findings across context boundaries when context fills with verbose output | "context degradation solution" |
| **Context Degradation** | Extended sessions start giving inconsistent answers, referencing "typical patterns" instead of specific findings | "extended session signals" |
| **Claim-Source Mappings** | Structured outputs linking each claim to its source URL, document name, publication date | "provenance preservation" |
| **Conflict Annotation** | Flagging conflicting statistics with source attribution rather than arbitrarily selecting one value | "two sources disagree" |
| **Self-Review Limitation** | A model that generated code retains reasoning context, making it less likely to catch its own errors | "independent review instance" |
| **Structured State Manifests** | Agent state exported to a known file location for crash recovery — coordinator loads on resume | "crash recovery" |
| **/compact** | Claude Code command to reduce context usage during extended exploration sessions | "context fills up" |

---

## Escalation Decision Framework

```
Customer explicitly requests human?
└── YES → Escalate IMMEDIATELY (do not attempt to resolve first)
└── NO → Is policy silent or ambiguous on this specific situation?
          └── YES → Escalate (policy gaps = escalation trigger)
          └── NO → Can the agent make meaningful progress?
                    └── NO → Escalate
                    └── YES → Resolve autonomously
```

**Trap:** "Be conservative" and self-reported confidence scores are NOT reliable escalation signals. Explicit criteria + few-shot examples are.

**Trap:** Self-reported confidence fails because "the agent is already incorrectly confident on hard cases."

---

## Error Propagation in Multi-Agent Systems

**BAD — hides context:**
```json
{"status": "search unavailable"}
```

**GOOD — enables intelligent recovery:**
```json
{
  "isError": true,
  "errorCategory": "transient",
  "failureType": "timeout",
  "attemptedQuery": "AI impact on music production 2024",
  "partialResults": ["3 articles found before timeout"],
  "alternativeApproaches": ["try narrower query", "use academic database instead"]
}
```

**Two anti-patterns:**
1. Suppressing errors (returning empty results as success) — prevents recovery
2. Terminating entire workflow on single subagent failure — unnecessary when partial results could succeed

---

## Context Optimization Patterns

| Problem | Solution |
|---|---|
| Tool results accumulate (40+ fields, only 5 relevant) | Trim to relevant fields BEFORE they enter context |
| Key findings buried in middle of long input | Place summaries at beginning + use explicit section headers |
| Extended session context degradation | Scratchpad files + spawn subagents for isolation |
| Verbose exploration output polluting main context | Use subagents with `context: fork` to isolate |
| Session crash mid-pipeline | Structured state manifests + coordinator loads on resume |
| Context fills with verbose discovery output | Use `/compact` command |

---

## Multi-Source Synthesis Reliability

**Source attribution loss pattern:**
1. Web search agent returns 10 sources with citations
2. Document analysis agent summarizes → citations lost
3. Synthesis agent receives prose → no way to attribute claims

**Fix:** Require subagents to output **structured claim-source mappings**:
```json
{
  "claim": "AI adoption in creative industries grew 47% in 2024",
  "evidence_excerpt": "...direct quote from source...",
  "source_url": "https://...",
  "document_name": "McKinsey AI Report 2024",
  "publication_date": "2024-03"
}
```

**Conflicting data:** Annotate both values with source attribution. Don't pick one. Don't average. Flag as contested.

---

## Common Exam Traps

1. **Generic errors hide recovery context** — "search unavailable" prevents the coordinator from knowing whether to retry, use alternatives, or proceed with partial results
2. **Suppressing errors as success** — returning empty results instead of an error prevents any recovery
3. **Whole-workflow termination on single failure** — when recovery strategies could succeed
4. **Explicit human requests** — must be honored immediately, NOT after attempting resolution
5. **Policy silence = best guess** — WRONG. Policy gaps are escalation triggers
6. **Progressive summarization is always safe** — WRONG. Condensing "customer stated $23.50 refund for order #45892 received 3 days late" into "customer had delivery issue" loses precision
7. **Self-review catches errors** — WRONG. The model that generated code is biased toward its own decisions
8. **Lost-in-middle is fixed by larger context windows** — WRONG. Position-based attention issues don't go away with bigger windows

---

## Mnemonics

**SPACE** — Context Management pillars:
- **S**tructure errors (failure type + partial results + alternatives)
- **P**osition matters (key findings at beginning, not middle)
- **A**ccumulation trimmed (only relevant fields in context)
- **C**laims mapped to sources (provenance through pipeline)
- **E**scalate on gaps (policy silence = escalate, not guess)

**"Independent instances catch what self-review misses"**

**"Suppress errors = suppress recovery"**

---

## High-Yield Scenarios

| Scenario | Correct Answer |
|---|---|
| Web search subagent times out | Return structured error context (failure type, query, partial results, alternatives) |
| Customer says "I want a human" | Escalate immediately — no autonomous attempt first |
| Policy doesn't cover competitor price matching | Escalate (policy gap = escalation trigger) |
| Two sources conflict: 34% vs 41% | Annotate both with sources, flag as contested |
| Long context session gives inconsistent answers | Scratchpad files + subagent delegation + /compact |
| 40+ fields in tool output, only 5 relevant | Trim before they enter context |
| Session crashed mid-research | Load structured manifest from agent state exports |
| Same session reviews code it generated | Use independent review instance instead |
