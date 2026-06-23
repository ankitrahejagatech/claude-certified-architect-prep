# Domain 2: Tool Design & MCP Integration (18%)

> Focus: tool description quality, structured error responses, scoped tool access, `tool_choice` configuration, and MCP scoping.

---

## Key Concepts Table

| Concept | Definition | Exam Signal |
|---|---|---|
| **Tool Description** | Primary mechanism LLMs use to select tools; minimal descriptions → unreliable selection | "tool misrouting" |
| **Description Boundaries** | Explain when to use this tool vs. similar alternatives — critical for overlapping tools | `analyze_content` vs `analyze_document` |
| **`isError` flag** | MCP pattern to signal tool failure vs. empty results back to the agent | "access failure vs no results" |
| **`errorCategory`** | `transient` (retry) / `validation` (bad input) / `business` (policy violation) / `permission` (access denied) | structured error response |
| **`isRetryable`** | Boolean indicating whether retrying the same call would succeed | "retry or escalate?" |
| **`tool_choice: "auto"`** | Model may call a tool OR return conversational text | "not guaranteed to use tool" |
| **`tool_choice: "any"`** | Model MUST call at least one tool (can choose which) | "guarantee tool is called" |
| **Forced tool selection** | `{"type": "tool", "name": "extract_metadata"}` — forces specific tool first | "specific sequence required" |
| **Scoped tool access** | Each agent gets only tools relevant to its role; too many tools degrades selection reliability | "18 tools vs 4-5" |
| **`.mcp.json`** | Project-level MCP server config — version-controlled, shared with team | "shared team tooling" |
| **`~/.claude.json`** | User-level MCP server config — personal/experimental, not version-controlled | "personal MCP servers" |
| **MCP Resources** | Read-only content catalogs (issue summaries, DB schemas) that give agents upfront data visibility | "reduce exploratory tool calls" |
| **Empty result vs error** | Empty result = valid query, no matches; error = failure (invalid input, service down) | "no records found" |

---

## Tool Description Quality Spectrum

```
BAD  → "get_customer - Retrieves customer information"
BAD  → "lookup_order - Retrieves order details"
                ↓ (result: agent can't distinguish these)

GOOD → "get_customer(identifier) - Look up a customer account by email, phone, 
        or customer_id. Use THIS tool first before any order operations to verify 
        customer identity. Returns: customer_id, name, email, account_status.
        Use lookup_order (not this tool) when the user provides an order number."

GOOD → "lookup_order(order_id_or_customer_id) - Retrieve order details using 
        an order number OR a verified customer_id from get_customer. Returns: 
        order_id, items, status, amount, delivery_date.
        Only use after customer identity is verified via get_customer."
```

**Rule:** Descriptions must answer — WHAT it does, WHAT parameters, WHAT it returns, WHEN to use it, WHEN NOT to use it (vs. similar tools).

---

## Structured Error Response Pattern

```json
{
  "isError": true,
  "errorCategory": "transient",
  "isRetryable": true,
  "message": "Database connection timeout after 30s",
  "attemptedOperation": "lookup_order(order_id='ORD-12345')",
  "partialResults": null,
  "suggestedAction": "retry after 5s"
}
```

```json
{
  "isError": true,
  "errorCategory": "business",
  "isRetryable": false,
  "message": "Refund amount $750 exceeds automatic approval limit of $500",
  "suggestedAction": "escalate_to_human with refund_id for manual review"
}
```

**Key distinctions:**
- `transient` → retry (timeout, service unavailable)
- `validation` → fix the input (invalid format, missing required field)
- `business` → policy violation, `isRetryable: false`, explain to user or escalate
- `permission` → access denied, `isRetryable: false`

---

## `tool_choice` Options

| Option | Behavior | Use When |
|---|---|---|
| `"auto"` | Model may call a tool OR return text | Default; flexible |
| `"any"` | Model MUST call one tool (any tool) | Structured output needed; multiple schemas possible |
| `{"type":"tool","name":"extract_metadata"}` | Forces specific named tool | First step must be specific tool before subsequent steps |

**Exam trap:** `tool_choice: "auto"` does NOT guarantee a tool is called — the model can return plain text. If you need structured output, use `"any"` or forced selection.

---

## Scoped Tool Access

```
PROBLEM: Give synthesis agent access to 18 tools
→ Tool selection reliability degrades
→ Agent misuses tools outside its specialization (web search during synthesis phase)

SOLUTION: Scope tools to role
→ Synthesis agent: [verify_fact, format_output, cite_source]  (3 tools)
→ Web search agent: [search_web, fetch_url, filter_results]   (3 tools)
→ Coordinator: [Task, evaluate_coverage, generate_subtasks]   (3 tools)
→ Each agent has "Task" only if it needs to spawn subagents
```

**Cross-role tools:** Provide a scoped `verify_fact` tool for the synthesis agent's 85% simple verification cases rather than routing everything through the coordinator.

---

## MCP Server Scoping

| Scope | File | Use For |
|---|---|---|
| Project (shared team) | `.mcp.json` in project root | Standard integrations, version-controlled, with `${GITHUB_TOKEN}` expansion |
| User (personal) | `~/.claude.json` | Experimental servers, personal tools, not shared |

Both are available simultaneously when Claude Code runs. Tools from all configured MCP servers are discovered at connection time.

---

## Common Exam Traps

1. **Empty results ≠ error** — `{"records": [], "total": 0}` with `isError: false` is the correct response for zero-result queries
2. **Generic errors hide context** — "Operation failed" prevents the agent from knowing whether to retry, fix input, or escalate
3. **`isRetryable: true` on business errors** — Don't mark business rule violations as retryable; the same call will fail again
4. **Too many tools** — 18 tools for one agent degrades selection; scope to 4-5 relevant tools per role
5. **`tool_choice: "auto"` guarantees tool use** — FALSE. Use `"any"` or forced selection for guaranteed structured output
6. **Community vs custom MCP servers** — Prefer existing community servers for standard integrations (Jira, GitHub); custom servers for team-specific workflows

---

## Mnemonics

**DESCRIBE** — tool design checklist:
- **D**ifferentiate from similar tools (explicit boundary conditions)
- **E**xamples (input examples in description)
- **S**tructure errors (isError, errorCategory, isRetryable)
- **C**ategory errors correctly (transient/validation/business/permission)
- **R**etryable flag (business errors: false; transient: true)
- **I**solate tools per role (scoped access, 4-5 per agent)
- **B**oundary explanations (when to use vs when NOT to use)
- **E**mpty ≠ error (zero results with isError: false)

---

## High-Yield Scenarios

| Scenario | Correct Answer |
|---|---|
| Agent calls `get_customer` for order lookups (wrong tool) | Expand tool descriptions with clear boundaries |
| Synthesis agent calls web search tools | Scope tool access — remove web search from synthesis agent |
| Refund exceeds policy limit | isError:true, errorCategory:business, isRetryable:false |
| Database timeout | isError:true, errorCategory:transient, isRetryable:true |
| Zero results from valid customer query | isError:false, records:[], total:0 |
| Need to guarantee first tool called is `extract_metadata` | Forced tool selection: {"type":"tool","name":"extract_metadata"} |
| Shared team GitHub integration | .mcp.json with ${GITHUB_TOKEN} environment variable |
| Personal experimental MCP server | ~/.claude.json |
