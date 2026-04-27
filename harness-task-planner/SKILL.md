---
name: harness-task-planner
description: Plan coding-agent tasks using the harness engineering framework from Birgitta Böckeler's article (martinfowler.com/articles/harness-engineering.html). Agent-agnostic — works for Claude Code, Codex, Cursor, Aider, Cline, or any LLM-based coding agent. Bilingual: asks the user upfront whether to converse in pt-BR or en, then conducts the conversation in that language while always producing the task document and subagent prompts in English. Use this skill whenever the user is about to delegate a task to a coding agent, asks to "create a task", "criar tarefa", "plan a task", "preparar tarefa", "prepare a task for the agent", "set up an agent task", mentions feedforward/feedback controls, guides/sensors, or wants to harness a coding agent. Also use whenever the team is starting any non-trivial code change with an AI agent — the skill walks through guides (feedforward) and sensors (feedback) across maintainability, architecture fitness, and behaviour, enforces TDD and mutation testing, blocks access to .env files, and produces a ready-to-execute task document. Use even if the user does not explicitly say "harness" — any time someone says "I want the agent to do X" or "quero que o agente faça X" and X is more than a one-line tweak, this skill applies.
---

# Harness Task Planner

This skill turns a fuzzy "make the agent do X" request into a structured task document with explicit feedforward guides and feedback sensors, based on the framework in [Harness engineering for coding agent users](https://martinfowler.com/articles/harness-engineering.html) by Birgitta Böckeler.

## Why this exists

Coding agents fail in predictable ways: they misdiagnose issues, over-engineer, drift from conventions, and fake progress with green-but-empty tests. The harness mental model says: don't just hand the agent a prompt — wrap the prompt in *guides* (controls that steer the agent before it acts) and *sensors* (controls that catch problems after it acts so it can self-correct). Do this across three regulation categories: **maintainability**, **architecture fitness**, and **behaviour**.

This skill walks the user through assembling those controls *before* the agent starts, producing a task document the team and the agent both follow.

## Agent-agnostic

This skill does not assume any specific coding agent. It works for Claude Code, OpenAI Codex, Cursor, Aider, Cline, GitHub Copilot agents, or any other LLM-based coding agent. Wherever the article (or this skill) refers to "the agent", read it as "whichever coding agent your team is using for this task".

Practical implications:
- Don't recommend agent-specific features (e.g. "use a Claude Skill", "use a Cursor rule") without first asking which agent the user is on.
- The prompt prefix at the end of the task document is written in plain English so any agent can follow it.
- Where an agent has a native equivalent for a guide (e.g. `AGENTS.md`, `CLAUDE.md`, `.cursorrules`, `.aider.conf.yml`), name it neutrally as "the agent's instruction file" and let the user pick the concrete name.

## Fullstack scope

This skill applies to the entire stack: backend services, frontend apps, mobile, BFFs, infra-as-code, anything the team writes. The framework (guides + sensors + three categories + steering loop) is the same; only the specific tools change.

When walking through the workflow, ask early: **is this task backend, frontend, fullstack, or other (mobile, IaC, data pipeline)?** This drives:

- Which tooling list applies (see Rules 1, 3, 4 below)
- What "behaviour" looks like — for backend it's API contracts and side effects; for frontend it's user-visible behaviour, accessibility, and visual regression
- What architecture fitness sensors are relevant — perf budgets for backend mean p95 latency; for frontend they mean Core Web Vitals, bundle size

If the task is fullstack and crosses the boundary (e.g. "add a new endpoint and the screen that consumes it"), treat it as **two coordinated tasks** rather than one — separate scope, separate guides where they differ, but a single shared spec and contract. Most agent failures on fullstack tasks come from blurring the boundary, not from technical difficulty on either side.

### Frontend-specific behaviour sensors (add to 3c when relevant)

The default test pyramid (unit, component, e2e) is necessary but not sufficient on the frontend. Add when applicable:

- **Accessibility**: axe-core / pa11y / Lighthouse a11y — computational — **block** if the task touches UI surface
- **Visual regression**: Percy, Chromatic, Playwright snapshots — report by default, block if the team has approved baselines
- **Component contract / interaction**: Storybook with interaction tests, Testing Library — computational — block

### Frontend-specific architecture fitness sensors (add to 3b when relevant)

- **Bundle size budget**: webpack-bundle-analyzer with a CI threshold (or local equivalent), or `bundlewatch` — computational — block if a budget exists
- **Core Web Vitals / Lighthouse perf score**: only meaningful with a stable measurement environment; otherwise report not block
- **Dead component / unused export detection**: knip, ts-prune — computational — report

## Subagent pipeline (mandatory)

Every harnessed task runs through a fixed three-role pipeline. The roles are independent agent invocations — they do not have to be the same model or the same tool, just three separate runs with different briefs and (importantly) different context windows.

The roles are:

1. **Implementer** — does the work. Follows the task document, applies TDD, runs the blocking sensors, produces the diff and a summary of what changed.
2. **Code reviewer** — reads the diff and the task document with a fresh context. Comments on design quality, hidden duplication, naming, error handling, missing edge cases, things that pass the linter but smell wrong. Does *not* re-run sensors — that's the implementer's job and the human's verification.
3. **QA** — reads the task document (especially success criteria and scope), then exercises the change. For backend: hits the touched endpoints/functions with realistic and adversarial inputs. For frontend: walks the user flows, including error paths, empty states, and accessibility. Reports what works, what breaks, what's unclear.

### Block vs report — important

All three subagents are **report**, not block. This is a deliberate choice and has a consequence the human review must internalise:

- **Computational blocks still apply** — cobertura, mutation, complexity, file size, lint, typecheck, `.env` rule. Those will fail the task on their own without any human judgment needed.
- **Subagents do not gate the task on their own.** Their output is *input* to the human review, not a substitute for it.
- **The human is the only semantic gate.** When a subagent flags something, the human decides whether it matters. When all three subagents say "looks good", the human still reviews — three agents agreeing is *evidence*, not *proof*.

The reason: report-only subagents are useful as extra sensors but they hallucinate, miss things, and develop blind spots together (especially if they share a base model). Promoting them to block creates a false sense of safety and lets the human disengage. Report-only keeps the human accountable.

### Independence rules

For the pipeline to add real signal:

- **Reviewer and QA must not see the implementer's reasoning** — only the diff, the task document, and (for QA) the running system. Otherwise they anchor on the implementer's framing and lose the fresh-eyes value.
- **Reviewer does not see QA's findings** and vice versa — they should run in parallel from the same artefact (the diff), not in sequence.
- **The same model can play all three roles** if necessary, as long as each invocation starts a new context. Different models is better — disagreements between them are the most useful signal.

### What the human does with three reports

After the implementer declares done and the two reviewers run, the human reads:

1. Implementer's summary (what changed, sensor results, anything skipped)
2. Code reviewer's comments
3. QA's findings

Then the human decides one of three things:
- **Merge as-is** — review confirms the work; comments were already addressed or are non-issues
- **Send back to implementer** — with specific items to address, framed as a follow-up loop (not a new task)
- **Reject and rethink** — the reports surfaced a problem with the spec or design that means the current approach is wrong; back to Phase 1

Log the decision and which subagent comments mattered in section 8 of the task document. This feeds the steering loop: if the same subagent comment recurs across tasks, it should become a guide or a computational sensor instead.

### When the pipeline is overkill

For trivial tweaks (single-line changes, typo fixes, version bumps), running three subagents is wasteful. If Phase 1 confirmed the task is trivial, collapse to:
- Implementer + human review only
- The non-negotiable computational blocks still all apply

For everything else, the three-subagent pipeline is mandatory.

## Spec-Driven Development (SDD) integration

This skill plays well with [GitHub Spec Kit](https://github.com/github/spec-kit) and similar SDD toolkits (Kiro, AWS Spec Workflow, etc.) that produce `spec.md` → `plan.md` → `tasks.md` as versioned artefacts before code is written. The two approaches solve different problems and compose cleanly:

- **SDD** governs *what to build and how*: turning intent into a versioned, reviewed spec, then into a plan, then into an ordered task list. Its outputs are the upstream artefacts.
- **This skill** governs *how to safely execute a single task*: feedforward guides + feedback sensors + non-negotiable rules + subagent pipeline. Its output is the per-task harness.

If the team uses SDD, this skill operates on each item in `tasks.md` — one harnessed task per task entry.

### How they fit together

```
[Spec Kit / SDD upstream]
  /specify  → spec.md     (what + why; user journeys; success)
  /plan     → plan.md     (architecture, stack, constraints)
  /tasks    → tasks.md    (ordered, dependency-aware task list)
       ↓
[For EACH task in tasks.md:]
  → harness-task-planner skill produces per-task document
    with guides, sensors, three-subagent pipeline, blocking thresholds
       ↓
  /implement (or hand to implementer agent)
```

### Constitution.md alignment

Spec Kit's `constitution.md` is meant for non-negotiable project principles. The five non-negotiable rules in this skill (TDD, coverage 80% + mutation 70%, `.env` off-limits, cyclomatic complexity ≤10, file ≤400 LOC) are exactly the kind of content that belongs there. If the team uses Spec Kit, copy these into `constitution.md` so every `/plan` and `/tasks` invocation respects them upstream — it reduces churn at the per-task harness step.

### Phase 1 changes when SDD is in use

When the team is on SDD, Phase 1's "capture the task" gets shorter — much of the work has already happened upstream:

- **Outcome**, **success criteria**, **scope** → already in `spec.md` and `tasks.md`. Just reference them instead of re-eliciting.
- **Pre-flight checks** still run as before (mutation tool, complexity tool, file-length check, coverage on diff, `.env` in `.gitignore`).
- **Spec drift check** (new) — verify the task entry in `tasks.md` still matches the current `spec.md` and `plan.md`. If they have drifted (common when specs evolve mid-implementation), flag for upstream reconciliation before starting the task. Spec Kit communities call this "spec drift" and it is a real failure mode.

### Phase 2 changes when SDD is in use

The feedforward guides Phase walks the user through what already exists upstream:

- `spec.md` is the **behaviour guide** (Phase 2c).
- `plan.md` is the **architecture fitness guide** (Phase 2b) — performance goals, constraints, structure.
- `constitution.md` (if present) is part of the **maintainability guide** (Phase 2a), alongside any conventions doc.
- The agent's instruction file (`AGENTS.md`, `CLAUDE.md`, etc.) still applies and references the SDD artefacts.

Don't duplicate content into the per-task harness document — link to the upstream artefacts. Duplication is the fastest way to create drift.

### When SDD is overkill

SDD shines for substantial features and multi-task work. It is heavy for:
- Hotfixes and incident response
- Trivial tweaks (the same tasks the subagent pipeline collapses for)
- Exploratory spikes where the spec is the question, not the input

For those, run this skill standalone without the SDD upstream. Be explicit that the team is doing so, and that the lack of an upstream `spec.md` puts more weight on Phase 1's outcome/success-criteria/scope elicitation.

### When SDD is in use but this skill isn't

Spec Kit's `/implement` will happily run a task without any of the harness controls in this skill. That is the failure mode this skill exists to prevent — green tests on a task that nobody actually verified, sensors not configured, `.env` in agent context. **Do not skip this skill just because SDD generated `tasks.md` for you.** SDD answers "is this the right task" and this skill answers "will the agent execute the task safely" — both are needed.

## Non-negotiable rules

These six apply to every task this skill produces, regardless of size, scope, or how much the user pushes back. They are non-negotiable because they protect against the failure modes that hurt the most: shipping behaviour-broken code, leaking secrets, producing unmaintainable code, accumulating god files, declaring a task done without proof, and silent skips of any of the above.

### Rule 1 — TDD with mutation testing is mandatory

Every task must follow test-driven development:

1. The agent writes a failing test that expresses the next slice of behaviour.
2. The agent writes the minimum code to make it pass.
3. The agent refactors with the test green.
4. Repeat until the success criteria are met.

After the feature tests pass, the agent runs **mutation testing** on the touched code. Two thresholds must both be met before the task is considered done:

- **Test coverage on touched code: ≥ 80%** (lines or branches, depending on tool — pick one and be consistent)
- **Mutation kill rate on touched code: ≥ 70%**

Coverage is measured *on the diff*, not on the whole codebase — the question is "did this task ship adequately tested code", not "is the codebase well-tested overall". If the team has stricter standards (90% / 75%, etc.), use those; never go below these floors.

If either threshold is below target, the agent must add or strengthen tests until both pass — not weaken either threshold, exclude mutants, or mark code as "not testable".

Mutation testing is the only computational sensor the article calls out as effective at catching the "tests are green but meaningless" failure mode in the behaviour category. Without it, the behaviour harness is mostly faith. With it, you get a deterministic signal that the tests actually constrain the code.

Tooling by language (suggest the right one when the user picks a stack):
- JS/TS (back and front): Stryker — works for Node.js services, React/Vue/Svelte components, anything Jest/Vitest can test
- Python: mutmut, cosmic-ray
- Java/Kotlin: PIT
- Ruby: mutant
- Go: go-mutesting
- C#: Stryker.NET
- Rust: cargo-mutants
- PHP: Infection

If the team has no mutation testing tool installed, this skill flags that as a gap to address *before* the task starts, not after. A behaviour harness without mutation testing is incomplete and should be stated as such in section 8 of the task document.

### Rule 2 — `.env` files are off-limits to the agent

The agent must not read, write, edit, copy, commit, paste into chat, summarise, or in any other way access `.env`, `.env.*`, `.env.local`, `.envrc`, or any file whose contents are environment variables or secrets. This applies even if:

- The user pastes one into the conversation.
- The agent thinks it needs one to debug.
- A test or build script appears to need values from one.

If the agent needs an environment value, it asks the human, who provides it out-of-band (or sets it in their shell before invoking the agent). The agent never opens the file.

The task document must include this rule in section 9 (the prompt prefix) so the agent sees it on every task. It is not a matter of trust — it is a matter of keeping secrets out of model context, logs, and version control.

If `.env*` patterns are not already in `.gitignore`, flag that as a blocking gap before the task starts.

### Rule 3 — Cyclomatic complexity must stay under threshold

Every function/method the agent writes or touches must keep its cyclomatic complexity at or below the team threshold (default 10 per function). The full module's average complexity must not regress against the pre-task baseline.

Why default: high complexity is the single most reliable predictor of bugs and the failure mode the article calls out as caught reliably by computational sensors. Letting an agent compound complexity is how codebases become unharnessable — Ashby's Law in reverse.

How to handle a violation:
- The agent must refactor (extract function, replace conditional with polymorphism, early returns, table-driven dispatch) until the function passes.
- The agent must not raise the threshold, add suppressions/ignores, or split a complex function into two equally tangled pieces just to game the metric.
- If a genuine domain reason makes refactoring impossible for a specific function, the agent stops and asks the human, who decides whether to accept an exception (logged in section 8) or rethink the design.

Tooling by language:
- JS/TS: ESLint with `complexity` rule, or `eslint-plugin-sonarjs`
- Python: `radon cc`, or `flake8` with `mccabe` plugin
- Java: PMD `CyclomaticComplexity`, or Checkstyle
- Kotlin: detekt
- Ruby: RuboCop `Metrics/CyclomaticComplexity`
- Go: `gocyclo`, or `golangci-lint` with `gocyclo`/`cyclop`
- C#: SonarAnalyzer, or built-in code metrics
- Rust: `clippy` with `cognitive_complexity` (note: cognitive is a close cousin and acceptable as substitute)
- PHP: PHPMD `CyclomaticComplexity`

If no complexity tool is configured for the target language, that is a blocking gap — surface it in Phase 1 alongside the other pre-flight checks.

### Rule 4 — No god files (400 LOC max per file)

Every file the agent creates or edits must end the task with **no more than 400 lines of code** (LOC, not counting blank lines and comments — use the convention your linter/cloc tool uses, just be consistent).

Why default: god files are the maintainability failure mode that compounds the worst. They drag complexity, coupling, and review cost up together. They also wreck the architecture fitness harness — once a file mixes too many responsibilities, structural tests can't constrain it because there is no clean structure to test.

How to handle a file that would exceed 400 LOC:
- The agent must split the file by responsibility *before* finishing the task — extract a module, split by feature, separate concerns, introduce a seam.
- The agent must not append to a file that is already at or near the limit without splitting first. "It already had 380 lines" is not an excuse to push it to 450.
- The agent must not split arbitrarily (e.g. `helpers_part_2.js`) — splits must reflect a real responsibility boundary.
- If a genuine technical reason forces a file over the limit (generated code, single large data table, framework-mandated layout), the agent stops and asks the human, who decides whether to log an exception in section 8.

Tooling:
- Most linters can enforce this directly:
  - JS/TS: ESLint `max-lines`
  - Python: `flake8` with `wemake-python-styleguide`, or `radon raw`
  - Ruby: RuboCop `Metrics/ClassLength` (close proxy)
  - Go: `golangci-lint` with `funlen`/`lll` and a custom check, or `wc -l`
  - Java/Kotlin: PMD `ExcessiveClassLength`, detekt `LargeClass`
- Otherwise, a `cloc`-based pre-commit check works for any language.

If no file-length check is configured, that is a blocking gap — surface it in Phase 1.

### Rule 6 — Mandatory structured status reports

No subagent in the pipeline gets to declare "done" with a free-form summary. Each of the three roles must produce a structured status report as the last block of its output:

**Implementer — Conformance report.** Pass/fail rows for: coverage ≥ 80%, mutation ≥ 70%, max function complexity ≤ 10, max file length ≤ 400 LOC, lint clean, type check clean, full test suite green, no `.env` files touched. Real measured numbers in every row.

**Code reviewer — Code review status.** Y/N rows for: design appropriateness, hidden duplication, naming, error handling, edge case coverage, conventions/guides compliance, cross-cutting smells. Plus an overall recommendation: `READY_TO_MERGE | MINOR_FIXES_NEEDED | MAJOR_REWORK_NEEDED`.

**QA — QA status.** Y/N/N/A rows for: happy path, error paths, empty/boundary states, adversarial inputs, side effects matching spec, accessibility, responsiveness, back↔front contract, out-of-scope detection. Plus the same overall recommendation.

The exact format for each lives in the task template (sections 9, 10a, 10b) and is pasted verbatim into every prompt prefix. Real measured values, no placeholders. "DID NOT RUN" / blank rows are not acceptable.

Why this is non-negotiable: without structured reports, "I ran the sensors" / "I reviewed it" / "I tested it" become prose the human has to mentally re-verify. With them, the human reads three short tables, scans for any "N", and knows in seconds where to focus. It also makes silent skips visible — a missing row is much louder than vague prose.

**Important: status reports do NOT change the report-vs-block nature of the subagents.**
- The implementer's conformance report is **block** — any "N" means the task is not done. The agent must fix and re-run, or stop and ask the human, before handing off.
- The code reviewer's and QA's status tables are **report** — even all-Y does not mean the task is approved. Only the human approves.

The human checklist (section 7 of the template) verifies that all three tables are present and complete, and that any "N" has a corresponding comment, finding, or fix.

## Core vocabulary (use these terms throughout)

- **Guide (feedforward)**: anything that steers the agent *before* it generates code. AGENTS.md, skills, how-to docs, code mods, language servers, reference URLs.
- **Sensor (feedback)**: anything that observes *after* the agent acts and lets it self-correct. Linters, type checkers, tests, structural analysis, review agents, log scanners.
- **Computational control**: deterministic, fast, CPU. Linter, test, type check.
- **Inferential control**: LLM/semantic, slower, non-deterministic. Skill, review agent, LLM-as-judge.
- **Steering loop**: when an issue recurs, you don't fix only the symptom — you upgrade the harness so it's less likely to recur.
- **Three categories**:
  - **Maintainability** — internal code quality (duplication, complexity, conventions, coverage).
  - **Architecture fitness** — fitness functions for performance, observability, module boundaries, security posture.
  - **Behaviour** — does it actually do what was asked? (the elephant in the room — see references/categories.md).

If the user already uses these terms, mirror them. If not, introduce them naturally as you go — don't lecture.

## Workflow

Run these phases in order. Don't skip ahead — phase 0 sets up the conversation, phase 1 catches a lot of bad tasks before they consume tokens.

### Phase 0 — Choose conversation language

Before anything else, ask the user *once* per session:

> "Em qual língua você prefere conversar? / Which language do you prefer to chat in? **(pt-BR / en)**"

Once the user answers, conduct **all subsequent conversation, questions, suggestions, pushback, and explanations in that language** for the rest of the session. If the user switches languages mid-conversation, follow them.

**Critical: this only affects the conversation.** The output artefacts — the task document at `assets/task-template.md` and all subagent prompt prefixes (implementer, code reviewer, QA) — are always produced in **English**, regardless of conversation language. Do not translate them. Do not localise the templates. Do not translate the conformance report headers, status table categories, or rule text.

Why English-only output:
- LLM-based agents follow English prompts more consistently than other languages, especially for structured outputs like the Y/N status tables
- Technical terms in the harness framework (guides, sensors, mutation kill rate, fitness functions) are anchored to the source article in English — translating loses the reference
- Task docs are versioned artefacts that may be read by agents/teammates outside the original language context
- Mixing languages in a single artefact creates parsing ambiguity for the implementer

If the user asks why the doc is in English while the conversation is in Portuguese, explain briefly and move on — don't argue the choice.

If the user explicitly insists on a non-English task doc, comply but warn once: "Subagent compliance with structured outputs may degrade — if the conformance report or status tables come back malformed, that's the likely cause."

### Phase 1 — Capture the task and check it's worth harnessing

Ask the user, in conversation:

1. **What outcome do you want?** (One sentence. If they can't, the task isn't ready.)
2. **What does success look like?** (How will we know it's done?)
3. **What's the blast radius?** (Touches one file? A module? Cross-cutting?)
4. **Which agent will run this?** (Claude Code, Codex, Cursor, Aider, etc. — affects only how guides are referenced, not what they are.)
5. **Is this a one-shot tweak or a real change?** Trivial tweaks (rename a variable, fix a typo) still get the non-negotiable rules but can use the stripped-down template (skip phases 2–4 detail, jump to phase 5).

Then run the **pre-flight checks** before going further:

- Is a mutation testing tool installed for the target language? If not, this is a blocking gap — surface it now, before the user invests time in the task.
- Is a cyclomatic complexity tool configured for the target language? If not, blocking gap — agree on the tool and threshold (default 10 per function) before the task starts.
- Is a file-length check configured (linter rule or pre-commit hook for the 400 LOC limit)? If not, blocking gap.
- Is a coverage tool configured to measure coverage on the diff (not just project-wide)? If not, blocking gap.
- Are `.env*` patterns in `.gitignore`? If not, blocking gap — fix this first.
- Does the affected area have *any* existing tests? If not, the first slice of TDD will need to establish a test harness before the feature work starts; flag that so the user knows the task is bigger than it looks.
- Are any of the files in scope already at or near 400 LOC? If yes, the task plan must include splitting them before the feature work, not after.

Then state back what you understood in 2–3 sentences and confirm before proceeding. If the outcome is vague ("make the API better"), push back — vague specs are the #1 source of agent misdiagnosis. The article is explicit: *correctness is outside any sensor's remit if the human didn't clearly specify what they wanted.*

### Phase 2 — Assemble feedforward guides (one pass per category)

Walk the three categories in this order. For each, ask what guides already exist and what's missing. Suggest concrete additions.

#### 2a. Maintainability guides

Look for / ask about:
- An `AGENTS.md` (or `CLAUDE.md`, etc.) that lists conventions, naming, file layout
- Existing skills or how-to docs relevant to the area being changed
- Code mod / refactoring recipes (e.g. OpenRewrite, jscodeshift)
- A language server / type checker the agent can call

If any of these are missing for the affected area, flag it. Don't insist on creating them now — but note the gap in the task document so the team can address it in the steering loop.

#### 2b. Architecture fitness guides

Ask:
- Are there performance budgets / SLOs the change must respect?
- Are there observability/logging conventions?
- Are there module boundary rules (allowed dependencies, layering)?
- Are there security/compliance constraints (auth, PII, secrets)?

For each that applies to this task, list the doc/skill that codifies it. If none exists for a constraint that matters, the task document should explicitly state the constraint in prose so the agent has it.

#### 2c. Behaviour guides

This is the hardest. Ask:
- Where is the functional spec? (Prompt? Ticket? Multi-file doc?)
- Are there existing examples / fixtures / golden outputs the agent can mirror?
- Are there acceptance criteria expressed as runnable tests, or only as prose?

If the spec is only a prose prompt, push for at least one concrete example of input → expected output. The article calls out that "approved fixtures" patterns work well here — see references/categories.md.

### Phase 3 — Assemble feedback sensors (one pass per category)

Same three categories, now for sensors. For each, decide what runs *during* the agent's loop (cheap, fast) and what runs *after* (more expensive).

#### 3a. Maintainability sensors

Default set the agent must run on its own before declaring done:
- Linter (computational, fast) — block
- Type checker if applicable (computational, fast) — block
- Existing test suite (computational, fast-ish) — block
- **Coverage on touched code: ≥ 80%** (computational) — **block**
- **Cyclomatic complexity** check on every function the agent wrote or touched, threshold ≤ team default (default 10) — computational — **block**
- **File length** check: every touched file ≤ 400 LOC — computational — **block**

Optional / heavier:
- Duplication detector
- Inferential code review skill / agent

#### 3b. Architecture fitness sensors

- Structural tests (e.g. ArchUnit-style boundary checks) — computational
- Performance test if a perf budget exists — computational
- Dependency audit — computational
- Inferential architecture review skill — for cross-cutting concerns the rules can't catch

#### 3c. Behaviour sensors

Required, regardless of task size:

- **TDD test suite** that the agent grew alongside the code (red → green → refactor for every slice) — computational — **block**
- **Coverage on touched code: ≥ 80%** (computational) — **block** (also listed in 3a — same sensor, behaviour relevance noted here)
- **Mutation testing kill rate on touched code: ≥ 70%** — computational — **block**
- **Human review + manual exploratory testing** of the documented success criteria — human — **block before merge**

Add when applicable:

- **Approved-fixtures comparison** if the area has deterministic inputs/outputs
- **Inferential review agent** for higher-level coherence checks (still report, not block)

The article is explicit that the behaviour category isn't yet trustworthy on automation alone. TDD plus coverage plus mutation testing plus a human's eyes is the floor — don't go below it.

For each sensor listed across 3a, 3b, and 3c, mark it as **block** (must pass before agent declares done) or **report** (run and surface, human decides).

### Phase 4 — Distribute across the change lifecycle

**Important constraint for this team right now**: we are *not* modifying CI/CD as part of these tasks. Every blocking sensor must run on the agent's machine or the developer's machine, before the PR is opened. CI is treated as a *recipient* of already-validated work, not as a place where new sensors get added.

Place each control on a timeline:

```
Inside the agent's self-correction loop (every iteration, agent runs these):
  - feedforward guides referenced in section 4
  - linter
  - type checker
  - relevant unit tests
  - cyclomatic complexity check on touched functions
  - file-length check on touched files

Before the agent declares done (still on the agent, but heavier):
  - full test suite for the touched scope
  - coverage measurement on the diff (≥ 80%)
  - mutation testing on touched code (≥ 70%)
  - structural / boundary tests if applicable
  - inferential review skill, if used

Pre-push / pre-PR (developer's machine, before opening the PR):
  - re-run everything above as a single command (e.g. `npm run pre-push`,
    a Makefile target, a git pre-push hook, a local script)
  - manual exploratory testing of the success criteria

Existing CI (unchanged):
  - whatever the team already runs there — this skill does not add to it
```

The principle from the article is **keep quality left** — and right now "left" means "before the developer pushes". This is more demanding on local machines (mutation testing especially can be slow), but it's the cost of not changing CI right now. If a sensor is too slow to run locally on every task, scope it down: run it only on the *touched* paths/modules, not the whole codebase.

When the team is ready to revisit CI later, the same sensors move right (CI repeats them as a safety net) — the harness itself doesn't change, only where it executes.

### Phase 5 — Produce the task document

Use the template at `assets/task-template.md`. Fill it in with everything gathered, including the new **section 10 — Subagent pipeline** that names the three roles and how they will run for this specific task (which model/agent for each, whether they run in parallel, where their outputs land).

**Always write the document in English**, regardless of conversation language (see Phase 0). The headers, field names, prompt prefixes, conformance report, and status tables stay verbatim in English. The user-supplied content (outcome description, success criteria, scope notes) can be quoted from the user as-is — if they wrote it in Portuguese, leave it in Portuguese inside the doc, but the structural/scaffolding text around it stays English.

The output is a single markdown document the user can paste into a ticket, hand to the implementer agent as a prompt prefix, or commit to the repo as `.tasks/<slug>.md`.

After producing the document:

1. Show it to the user inline (don't just save and link).
2. Ask, *in the conversation language*: "Anything you'd add, remove, or change before the agent starts?" / "Algo que você adicionaria, removeria ou mudaria antes de soltar o agente?"
3. Note any guide/sensor gaps the user said they'd address later — those go into the steering-loop log (see below).

### Steering loop — only if the user comes back

If the user reports that the agent failed in some way, don't just help fix the symptom. Ask: "Should we upgrade the harness so this is less likely next time?" Then suggest the specific guide or sensor that would have caught it. This is the practice the article is built around — a single failure is data; a recurring failure is a missing control.

## Output format rules

The task document must follow `assets/task-template.md` exactly. Do not invent new top-level sections. If a section doesn't apply to a particular task, write "N/A — <one-line reason>" rather than deleting the section, so reviewers can see what was considered.

## Scope guardrails

- This skill plans tasks. It does **not** execute them.
- This skill does not write the AGENTS.md, skills, linter configs, or fitness functions itself — it only identifies that they're needed and references them. Creating those is a separate effort that the user may or may not want help with.
- For trivial one-line changes, offer the stripped-down path (phase 5 only with a minimal template) instead of the full workflow.

## When the user pushes back

If the user says "this is too much process for one task," they're often right for small tasks. Respond by:

1. Asking what the blast radius really is.
2. If small: collapse phases 2–4 into a single quick pass and produce a short task doc.
3. If large: explain that the harness cost is amortised — the guides and sensors you set up here serve every future task in the same area.

Don't dilute the workflow on large tasks just to be agreeable. The article's whole point is that under-harnessing is what causes the supervision burden the team is trying to escape.

## Reference

For deeper detail on the three categories — including the failure modes each one catches and misses — read `references/categories.md`. Read it the first time you use this skill in a session, or whenever the user asks "why are we doing X" about a category.

The source article: https://martinfowler.com/articles/harness-engineering.html
