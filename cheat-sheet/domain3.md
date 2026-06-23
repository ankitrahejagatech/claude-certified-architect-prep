# Domain 3 — Claude Code Configuration & Workflows (20%)

**Exam weight: 20%. Pass cut: 720/1000.**

---

## Three Global Rules That Resolve Most Questions

1. **Configuration > invocation.** When the team needs something to happen consistently, automatically, for everyone — the answer is a file in the right location, not a command someone has to remember.
2. **Path prefix is the whole answer.** `~/.claude/` is personal, never shared. `.claude/` (project) is shared via git. Same filename, opposite scope. Memorise the paths cold.
3. **Load only what's needed.** Always-loaded standards → CLAUDE.md. Pattern-matched standards → `.claude/rules/` with `paths:` frontmatter. On-demand workflows → skills/commands.

---

## CLAUDE.md Hierarchy

| Level | Path | Shared via git? | Scope |
|---|---|---|---|
| **User** | `~/.claude/CLAUDE.md` | **No** | Only you, on your machine |
| **Project** | `.claude/CLAUDE.md` or root `CLAUDE.md` | Yes | Everyone on the repo |
| **Directory** | `<subdir>/CLAUDE.md` | Yes | Only when working in that subdirectory |

**Directory CLAUDE.md scopes downward and local, never upward.**

**THE most-tested trap — the divergent-developers scenario:**  
Two developers on the same repo get inconsistent behaviour. Root cause 9/10 times: one developer wrote conventions to `~/.claude/CLAUDE.md` (user-level, never git-tracked). Git never carried it. The other developer's clone has nothing.

**Diagnostic:** *Divergent behaviour between teammates on same repo → suspect user-level vs project-level config first.*

**Modular organisation:**
- **`@import` syntax** inside CLAUDE.md — reference external files, import per-package standards
- **`.claude/rules/` directory** — topic-specific files as alternative to monolithic CLAUDE.md

**`/memory` command:** Lists which memory files are currently loaded. Use to **confirm the diagnosis**, not to fix it. If a file isn't there, fix it by putting the file in the right location.

---

## Custom Commands & Skills

| Type | Location | Shared? |
|---|---|---|
| Project slash commands | `.claude/commands/` | Yes — version-controlled |
| Personal slash commands | `~/.claude/commands/` | No |
| Project skills | `.claude/skills/<name>/SKILL.md` | Yes |
| Personal skills | `~/.claude/skills/<name>/SKILL.md` | No |

**Skill frontmatter — the three options:**

| Option | Effect | Exam trigger phrase |
|---|---|---|
| `context: fork` | Runs in isolated sub-agent; verbose output stays in fork; only summary returns to main | "produces verbose output", "clutters main context", "keep main context clean" |
| `allowed-tools` | Restricts which tools the skill can call; prevents destructive actions | "analyse only, must not modify", "restrict to read-only" |
| `argument-hint` | Prompts developer for required parameters when invoked without arguments | "prompt user for the parameter" |

**Skills vs CLAUDE.md:**
| | Skills | CLAUDE.md |
|---|---|---|
| When loaded | On-demand, when invoked | Always loaded |
| Purpose | Task-specific workflows | Universal standards |

**Personal customisation pattern:** Team has `/review` skill; one developer wants a stricter variant. **Wrong:** edit the project skill (affects teammates). **Right:** create `~/.claude/skills/review-strict/SKILL.md` — different name, personal location.

---

## Path-Specific Rules (`.claude/rules/`)

```yaml
---
# .claude/rules/testing.md
paths:
  - "**/*.test.tsx"
  - "**/*.test.ts"
---
# Testing Conventions
All test files must use describe/it blocks...
```

Loads **only** when Claude Code is about to work on a file matching the glob. Not loaded for non-matching files.

**Why path-specific rules beat directory CLAUDE.md for spread files:**

| | Directory CLAUDE.md | Path-specific rules |
|---|---|---|
| Scope | One directory only | Anywhere the glob matches |
| Test files spread across codebase | Need one CLAUDE.md per directory | One rule file, globs everywhere |
| Token efficiency | Loaded whenever in that directory | Loaded only on matching files |

**Signature phrase:** *"pattern of files spread across the codebase"* → path-specific rules.

---

## Plan Mode vs Direct Execution

| Use Plan Mode | Use Direct Execution |
|---|---|
| Large-scale changes (monolith → microservices) | Single-file bug fix, clear stack trace |
| Multiple valid approaches exist | Mechanical, known change (add validation conditional) |
| Architectural decisions required | Well-understood scope |
| Library migration with different API across 45+ files | Simple rename or trivial conditional |

**The dividing line:** *Are there decisions still to make?* → plan mode. *Is there just a known thing to do?* → direct execution.

**Hybrid pattern (commonly tested):** Enter plan mode → explore + design → exit plan mode → implement directly. Not either/or.

**Explore subagent:** Isolates verbose discovery output; returns summaries to main conversation. Same goal as `context: fork`, different mechanism.

---

## Iterative Refinement Techniques

When prose descriptions produce inconsistent results, switch modalities:

**1. Concrete input/output examples (highest leverage)**
```
Input:  user_name → Output: userName
Input:  api_key   → Output: apiKey
```
Three examples eliminate ambiguity that prose admits. Model generalises from examples more reliably than descriptions.

**2. Test-driven iteration** — write tests first, share failures, Claude iterates against machine-checkable spec.

**3. Interview pattern** — have Claude ask questions before implementing. Surfaces considerations you'd miss in unfamiliar domains.

**Batch vs sequence feedback:**
- Fixes **interact** → single message (Claude sees all at once, makes consistent decisions)
- Issues are **independent** → sequential (fix one, verify, move on)

**THE headline rule:** when prose fails repeatedly, switch modalities — don't rewrite the prose.

---

## CI/CD Integration

**The 5 Failure Patterns — Memorise the symptom → fix mapping:**

| Symptom | Root Cause | Fix |
|---|---|---|
| CI job **hangs indefinitely** | Claude Code waiting for interactive input | Add **`-p` flag** |
| PR comments need automatic **inline posting** | Output is narrative prose, not structured data | `--output-format json --json-schema` |
| **Duplicate comments** on every new commit | No context about what was already reviewed | Include prior findings + "report only new or unaddressed" |
| CI generates **duplicate tests** | Doesn't know what's already covered | Provide existing test files in context |
| CI-generated tests are **low-quality boilerplate** | No testing standards in context | Document test-value criteria in `.claude/CLAUDE.md` |

**`-p` / `--print` flag:** The single most-tested CI detail. Without it, the CI job hangs. If you see "hangs" or "waiting for input" — `-p` is the answer, always.

**Session context isolation:** The same Claude session that generated code is LESS effective at reviewing it — it retains reasoning context, less likely to question its own decisions. Use an **independent review instance**.

**Trap distractors for CI hang:** "Increase the timeout", "redirect /dev/null to stdin", "run in detached shell" — treat symptoms, not cause.

---

## Configuration-Layer Decision Tree

1. **Needs to happen every interaction?** → Project-level CLAUDE.md
2. **Applies to files scattered by pattern?** → Path-specific rule in `.claude/rules/` with `paths:` glob
3. **Applies only to one subdirectory?** → Directory-level CLAUDE.md
4. **Workflow invoked on demand?** → Skill in `.claude/skills/` or command in `.claude/commands/`
5. **Personal-only?** → Same answers but in `~/.claude/` instead of `.claude/`
6. **For CI to consume?** → `.claude/CLAUDE.md` (CI reads same hierarchy)

---

## Distractor-Recognition Cribsheet

**Reject these:**
- "Run `/memory reload` to pick up project instructions" (the file doesn't exist on their machine — it's user-level)
- "Move conventions to `~/.claude/CLAUDE.md`" (that makes it *less* shared, not more)
- "Edit the project skill to add stricter behaviour" (personal customisation → personal skill)
- "Create a `/migrate` slash command everyone invokes" (standards must be ambient, not opt-in)
- "Put one CLAUDE.md in every directory with test files" (maintenance disaster — use path-specific rules)
- "Rewrite the prose more carefully" (when prose has failed 3+ times — switch to concrete examples)
- "Increase the CI timeout" (for hanging CI — `-p` fixes root cause)
- "Reuse the code-generating session for review" (motivated reasoning — fresh session)
- "Remove the review bot" (developers finding it noisy → incremental review context is the fix)

**Lean toward these:**
- "Conventions are in user-level `~/.claude/CLAUDE.md` and were never version-controlled"
- "Split CLAUDE.md into `.claude/rules/` files with `paths:` frontmatter"
- "Create personal skill in `~/.claude/skills/` with different name"
- "Add `context: fork` so verbose output stays out of main conversation"
- "Path-specific rule with `paths: [\"**/*.test.*\"]`"
- "Plan mode for investigation; direct execution for implementation"
- "Add the `-p` flag" (for CI hang — always)
- "Use fresh independent Claude Code session for PR review"
- "Pass prior review findings and instruct 'report only new or unaddressed issues'"

---

## One-Line Mnemonics

- **Path prefix is the answer.** `~/.claude/` = personal. `.claude/` = shared.
- **Two devs, same repo, divergent behaviour:** user-level config, never committed.
- **`/memory` is for diagnosis, not fix.**
- **"Pattern of files spread across the codebase"** → path-specific rules with globs.
- **Standards must be ambient.** Slash command for a convention = wrong.
- **Skill vs CLAUDE.md:** on-demand vs always-loaded.
- **Personal customisation of a team skill:** new name, `~/.claude/skills/`. Never edit the team skill.
- **`context: fork`** for noisy skills. **Explore subagent** for noisy investigations.
- **Plan mode = decisions still to make.** Direct execution = known thing to do.
- **Prose failed three times:** concrete input/output examples, not rewritten prose.
- **CI hangs:** `-p` flag. Always.
- **Automated PR inline comments:** `--output-format json --json-schema`.
- **Same session reviewing own code:** motivated reasoning. Use fresh session.
- **Developers ignoring review bot:** incremental review context, not removal.
