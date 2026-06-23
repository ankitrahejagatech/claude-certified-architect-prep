# Domain 4: Prompt Engineering & Structured Output (20%)

> Focus: explicit criteria over vague instructions, few-shot for ambiguous cases, `tool_use` for guaranteed structured output, validation-retry loops, and the Message Batches API tradeoffs.

---

## Key Concepts Table

| Concept | Definition | Exam Signal |
|---|---|---|
| **Explicit Criteria** | Define exactly which issue types to flag vs. skip — beats "be conservative" or "only high-confidence findings" | "false positives", "inconsistent severity" |
| **Categorical Criteria** | Specify issue categories to flag (bugs, security) vs. skip (style, local patterns) | "style issues eroding trust" |
| **Few-Shot Examples** | 2-4 targeted examples for ambiguous scenarios — more effective than instructions alone for format and edge cases | "inconsistent output format" |
| **Ambiguous Case Demonstration** | Few-shot example showing a borderline pattern + reasoning for the decision | "generalize to novel inputs" |
| **`tool_use` for Structured Output** | Most reliable approach for guaranteed schema-compliant output; eliminates JSON syntax errors | "reliable structured extraction" |
| **`tool_choice: "any"`** | Guarantees a tool is called; use when multiple extraction schemas exist | "occasional plain-text response" |
| **Nullable Fields** | Optional schema fields that return null for absent information — prevents hallucination | "information may not exist" |
| **Validation-Retry Loop** | Send failed extraction + specific errors back as a follow-up request for self-correction | "semantic errors after extraction" |
| **Retry Ineffective When** | Required information is absent from source document — retrying cannot create information that doesn't exist | "when does retry fail?" |
| **`detected_pattern` field** | Schema field tracking which code construct triggered a finding — enables false positive pattern analysis | "analyze dismissal patterns" |
| **Message Batches API** | 50% cost savings, up to 24-hour processing window, no guaranteed latency SLA, `custom_id` for correlation | "cost savings", "overnight reports" |
| **Batch API Limitation** | Does NOT support multi-turn tool calling within a single request | "can't use for agentic workflows" |
| **Multi-Pass Review** | Per-file local analysis passes + separate cross-file integration pass — avoids attention dilution | "14-file PR inconsistent review" |
| **Independent Review Instance** | A separate Claude session without the generator's reasoning context catches more subtle issues | "self-review bias" |
| **Self-Review Limitation** | Generator retains reasoning context, less likely to question its own decisions | "same session reviews own code" |
| **Attention Dilution** | Processing many files simultaneously leads to inconsistent depth and missed issues | "superficial comments on some files" |

---

## Explicit Criteria vs Vague Instructions

**VAGUE (fails):**
> "Be more conservative and only report high-confidence findings."
> "Only flag issues you're sure about."

**Why it fails:** These don't tell the model what categories to prioritize — it applies the instruction inconsistently.

**EXPLICIT (works):**
> **Flag:** Security vulnerabilities (SQL injection, XSS, auth bypass), logic bugs (incorrect conditional, data corruption), data integrity errors (null pointer, out-of-bounds).
> **Skip:** Style preferences, local naming patterns that match existing codebase conventions, variable names differing from docs but consistent with local code.

**Why it works:** Categorical inclusion/exclusion criteria are applied consistently across all findings.

---

## Few-Shot Examples: Targeting Ambiguous Cases

Don't write examples for obvious cases. Target the borderline ones:

```
BAD example: [OBVIOUS BUG] → Flag as HIGH  (the model already knows this)
GOOD example: [BORDERLINE PATTERN] → Skip (acceptable local convention)
               Reasoning: This differs from the API docs naming convention but 
               matches all other usages in this file — flagging it would be a false positive.
```

**2-4 targeted examples** for ambiguous scenarios > lengthy prose instructions.

Few-shot enables **generalization to novel patterns** not explicitly specified — the model learns the judgment, not just the rules.

---

## Structured Output: `tool_use` Pattern

```python
# Define extraction as a tool
tools = [{
    "name": "extract_invoice_data",
    "description": "Extract structured invoice data from document text",
    "input_schema": {
        "type": "object",
        "properties": {
            "invoice_number": {"type": "string"},
            "total_amount": {"type": "number"},
            "line_items": {
                "type": "array",
                "items": {"type": "object", "properties": {
                    "description": {"type": "string"},
                    "amount": {"type": "number"}
                }}
            },
            "vendor_name": {"type": "string"},
            "due_date": {"type": ["string", "null"]}  # nullable — may not exist
        },
        "required": ["invoice_number", "total_amount", "line_items", "vendor_name"]
    }
}]

# Force tool use
response = client.messages.create(
    model="claude-opus-4-8",
    tools=tools,
    tool_choice={"type": "any"},  # Guarantees tool is called
    messages=[{"role": "user", "content": document_text}]
)
```

**`tool_use` eliminates JSON syntax errors but does NOT prevent semantic errors:**
- Values in wrong fields
- Line items that don't sum to stated total
- Misattributed data

→ Use validation-retry loops for semantic errors.

---

## Validation-Retry Loop Pattern

```python
def extract_with_retry(document, max_retries=2):
    result = extract(document)
    
    for attempt in range(max_retries):
        errors = validate(result)
        if not errors:
            return result
        
        # Include document + failed extraction + specific errors in retry
        result = extract(document, prior_extraction=result, errors=errors)
    
    return result  # Return best attempt

# When retries are EFFECTIVE:
# - Format mismatches (date "Jan 15 2024" instead of "2024-01-15")
# - Structural output errors
# - Field placement errors

# When retries are INEFFECTIVE:
# - Required information simply absent from the source document
# - Don't waste API calls retrying for data that doesn't exist
```

---

## Message Batches API Decision Matrix

| Criteria | Synchronous API | Message Batches API |
|---|---|---|
| Cost | Standard | 50% savings |
| Latency | Seconds | Up to 24 hours |
| Latency SLA | Guaranteed | None |
| Multi-turn tool calling | Supported | NOT supported |
| Use for blocking workflows | ✓ | ✗ |
| Use for overnight reports | ✓ | ✓ (preferred for cost) |
| Use for agentic workflows | ✓ | ✗ |

**`custom_id` field:** Correlates batch request/response pairs when processing results.

**Batch failure handling:** Resubmit only failed documents (identified by `custom_id`) with appropriate modifications (e.g., chunking documents that exceeded context limits).

---

## Multi-Pass Review Architecture

```
SINGLE-PASS PROBLEM (14 files together):
→ Attention dilution: inconsistent depth across files
→ Contradictory findings: flags pattern in file A, approves same pattern in file B
→ Misses cross-file issues that need holistic view

MULTI-PASS SOLUTION:
Pass 1: Per-file local analysis (each file independently → local issues, consistent depth)
Pass 2: Integration pass (cross-file data flow, architectural consistency, contradictions)
Independent instance: New session without generator's reasoning context
```

---

## Common Exam Traps

1. **"Be conservative"** → Doesn't reduce false positives; explicit categorical criteria do
2. **`tool_choice: "auto"` for guaranteed structured output** → FALSE. Use `"any"` or forced selection
3. **Strict schema eliminates all errors** → Eliminates syntax errors only; semantic errors still possible
4. **Retry always works** → Retries fail when information is absent from the source document
5. **Batch API for blocking workflows** → Batch has no latency SLA; unusable for pre-merge checks
6. **Batch API supports tool calling** → FALSE. No multi-turn tool calling in batch
7. **Larger context window fixes attention dilution** → FALSE. Split into passes instead
8. **Self-review is as good as independent review** → FALSE. Generator retains reasoning context

---

## Mnemonics

**CLEAR** — prompt engineering for reliability:
- **C**ategorical criteria (explicit include/exclude, not "be conservative")
- **L**imit false positives with few-shot examples for ambiguous cases
- **E**nforce structure via `tool_use` + nullable fields (not just prose instructions)
- **A**dd retry-with-error-feedback for semantic validation
- **R**oute to batch for latency-tolerant workloads (overnight, weekly)

**"Few-shot demonstrates judgment; instructions only describe rules"**

---

## High-Yield Scenarios

| Scenario | Correct Approach |
|---|---|
| 35% of findings are style issues developers dismiss | Explicit categorical criteria: flag bugs/security, skip style |
| Occasional plain-text response instead of JSON | Change `tool_choice` from "auto" to "any" |
| `due_date` field sometimes absent from invoices | Make `due_date` nullable in schema |
| Invoice totals not matching line item sums | Validation-retry loop with specific semantic error feedback |
| Retry returns same wrong result | Retrying is ineffective — information absent from source |
| 500 documents, results needed by 8am, ready at midnight | Message Batches API (latency-tolerant overnight workload) |
| Pre-merge check must complete before developer merges | Synchronous API (cannot use batch — no latency SLA) |
| 14-file PR review: inconsistent depth and contradictions | Split: per-file local passes + separate integration pass |
| Same session reviewing code it generated | Use independent Claude Code instance (no prior reasoning context) |
