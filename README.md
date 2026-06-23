# Claude Certified Architect: Foundations

Practice kit for Anthropic's Claude Certified Architect: Foundations exam. Content built from the official exam guide PDF.

Pass score: **720 / 1000.** Scenario-based. Tests architectural judgment under production constraints. Not API memorization.

**▶ [Launch the practice exam](https://ankitrahejagatech.github.io/claude-certified-architect-prep/practice-exam.html)** - runs in your browser, nothing to install.

---

## Why AI PMs should get this cert

Most AI PMs can describe what a model does. Few can specify how an agent system should behave when a tool call fails, when context runs out, or when a subagent goes off-track. This exam tests exactly that gap.

What you'll actually be able to do after passing:

1. **Write specs engineers don't have to translate.** You'll know the difference between a hooks-enforced constraint and a prompt-based one - and which one to specify for a given risk level.
2. **Debug production failures faster.** When an agent misbehaves, you'll know whether to look at stop_reason handling, context isolation, or tool description quality - instead of waiting for eng to explain it.
3. **Make better build-vs-configure calls.** MCP vs custom tool, `tool_choice: any` vs `auto`, project-level vs user-level config - these are PM decisions disguised as engineering ones.
4. **Evaluate AI vendors without getting played.** You'll recognize when a vendor is describing probabilistic behavior (prompts) as if it were guaranteed behavior (hooks).
5. **Hold your ground in technical design reviews.** Knowing the architecture means you can push back when a proposed design is fragile - not just defer to the team.

The cert takes about 2 weeks to prep if you use this kit. The signal it sends is disproportionate to the effort.

---

## What's in here

```
practice-exam.html        - interactive exam, all 5 domains, no setup needed
cheat-sheet/
  domain1.md              - Agentic Architecture & Orchestration (27%)
  domain2.md              - Tool Design & MCP Integration (18%)
  domain3.md              - Claude Code Config & Workflows (20%)
  domain4.md              - Prompt Engineering & Structured Output (20%)
  domain5.md              - Context Management & Reliability (15%)
  sample-questions.md     - All 12 official sample questions verbatim, with explanations
```

---

## How to use this

1. **Read the cheat sheet for a domain:** scan Key Concepts, commit the distractor-recognition cribsheet to memory, read the mnemonics.
2. **Run the domain's quiz:** open `practice-exam.html`, click the domain card, hit Start Quiz.
3. **Review every miss:** the explanation tells you why the wrong answers are wrong, not just what's right. That's where the pattern recognition happens.
4. **Prioritize Domain 1:** 27% weight, and its concepts (hooks, loop termination, context isolation) appear as distractors in D2-D5 questions. If D1 is weak, everything else suffers.
5. **Run the full exam:** when you're hitting 80%+ per domain, take the full exam cold.

---

## Exam format

| Property | Detail |
|---|---|
| Scoring | 100-1000 scaled (not raw %) |
| Pass | 720 / 1000 |
| Scenarios | 4 of 6 picked randomly per sitting |
| Questions | MCQ, one correct answer |
| Penalty | None. Guess freely. |

### The 6 official scenarios
1. Customer Support Resolution Agent
2. Code Generation with Claude Code
3. Multi-Agent Research System
4. Developer Productivity with Claude
5. Claude Code for CI/CD
6. Structured Data Extraction

---

## What each domain actually tests

**D1: Agentic Architecture (27%)**  
Stop_reason handling. Subagent context isolation. When to use hooks vs. prompts. PreToolUse vs PostToolUse. Idempotency keys. Failure tracing (missing topics = coordinator decomposition, not subagents). Task decomposition patterns.

**D2: Tool Design & MCP (18%)**  
Tool descriptions are the routing mechanism. Minimal descriptions break everything. `isError: false` for zero results vs `isError: true` for actual failures. `tool_choice: "any"` vs `"auto"`. Scoped tool access (4-5 per agent, not 18). `.mcp.json` project-level vs `~/.claude.json` personal.

**D3: Claude Code Config (20%)**  
Path prefix is the whole answer: `~/.claude/` = personal, never git-tracked; `.claude/` = shared via git. Divergent team behavior = user-level config never committed. `context: fork` for noisy skills. `-p` flag for CI. Path-specific rules for file patterns spread across the codebase.

**D4: Prompt Engineering (20%)**  
Explicit categorical criteria > "be conservative." Few-shot for borderline cases, not obvious ones. `tool_use` eliminates syntax errors, not semantic errors. Retry fails when the information isn't in the source document. Batch API: 50% savings, no latency SLA, no multi-turn tool calling.

**D5: Context Management (15%)**  
Case facts block to preserve specific values outside summarized history. Three escalation triggers: explicit human request, policy gap, can't make progress. Lost-in-the-middle: put key findings at the start. Conflicting sources: annotate both, flag contested. Don't pick one.

---

## The one frame that unlocks this exam

> **Prompts are probabilistic. Hooks are deterministic.**

When a single failure causes financial loss, a security breach, or a compliance violation: use hooks. Always. Doesn't matter how specific or well-worded the prompt is. The exam tests this distinction relentlessly across all 5 domains.

---

## What's NOT on the exam

Constitutional AI, RLHF, fine-tuning, embedding models, computer use, vision API, streaming/SSE, OAuth/API key rotation, rate limiting, quotas, pricing, prompt caching implementation details, specific cloud provider configs, token counting.

Don't waste time on any of it.

---

## Other community resources

| Resource | What it is | Link |
|---|---|---|
| Hamza Farooq: cheat sheets + 64Q exam | Community cheat sheets, sample questions, full practice exam | [github.com/hamzafarooq/claude-certified-architect](https://github.com/hamzafarooq/claude-certified-architect) · [live](https://practice-exam-deploy.vercel.app) |
| jaysevak: exam simulator | Timed exam + practice mode + domain drill. Single HTML file, works offline | [github.com/jaysevak/ccaf-practice-exam](https://github.com/jaysevak/ccaf-practice-exam) · [live](https://jaysevak.github.io/ccaf-practice-exam/) |
| aderegil: guided labs | 6 scenarios, 5 domains, 30 hands-on tasks in Python. Closest thing to the actual exam's production setup | [github.com/aderegil/claude-certified-architect](https://github.com/aderegil/claude-certified-architect) |
| mominurr: mock exam | 60 questions, 120-min timer, per-domain performance analytics | [github.com/mominurr/cca-f-mock-exam](https://github.com/mominurr/cca-f-mock-exam) |
| SpillwaveSolutions: Jupyter notebooks | 9 notebooks, 234 tests, architectural patterns by scenario | [github.com/SpillwaveSolutions/cca-exam-prep-customer-support](https://github.com/SpillwaveSolutions/cca-exam-prep-customer-support) |
| CCA Prep: study guide + flashcards | Full study guide across all 5 domains, plus cheat sheet, quiz, mock exam, and flashcards. Free | [claudecertprep.com](https://claudecertprep.com) |
| Anthropic official | Skilljar courses + exam registration | [anthropic.skilljar.com](https://anthropic.skilljar.com) |

---

## Credits

- Official exam guide: Anthropic (Claude Certified Architect: Foundations Exam Guide)
- Community cheat sheets and question patterns: Hamza Farooq - [github.com/hamzafarooq/claude-certified-architect](https://github.com/hamzafarooq/claude-certified-architect)
