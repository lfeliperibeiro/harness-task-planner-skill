# Harness categories — reference

Detail backing the three regulation categories in the main SKILL.md. Read this when you need to justify a choice, explain trade-offs to the user, or decide which sensors are worth the cost on a particular task.

Source: [Harness engineering for coding agent users](https://martinfowler.com/articles/harness-engineering.html), Birgitta Böckeler, April 2026.

## 1. Maintainability harness

**What it regulates**: internal code quality. Duplication, complexity, coverage, conventions, style, structural drift.

**Why it's the easiest of the three**: pre-existing tooling. Linters, type checkers, coverage tools, ArchUnit-style structural tests, code review skills — all of it works today.

**What computational sensors catch reliably**:
- Duplicate code (textual)
- Cyclomatic complexity over a threshold
- Missing test coverage on changed lines
- Architectural drift (forbidden dependencies)
- Style violations

**What inferential sensors can partially catch (expensively, probabilistically)**:
- Semantically duplicate code (same logic, different names)
- Redundant or low-value tests
- Brute-force fixes that paper over the real problem
- Over-engineered solutions

**What neither catches reliably** (and the user should know this):
- Misdiagnosis of the underlying issue
- Unnecessary features the agent invented
- Misunderstood instructions

When sensors miss the high-impact problems, the harness can't compensate for a vague spec. Push for spec clarity in phase 1.

## 2. Architecture fitness harness

**What it regulates**: the application's architectural characteristics — what Building Evolutionary Architectures calls *fitness functions*.

**Examples of guides**:
- A skill describing performance requirements (latencies, throughput targets)
- Logging/observability conventions as a how-to doc
- Module boundary rules expressed as constraints
- Security baseline (allowed crypto, secret-handling rules)

**Examples of sensors**:
- Performance tests that block if a budget regresses
- Structural tests for module boundaries (ArchUnit, dependency-cruiser, etc.)
- A debugging-quality skill that asks the agent to reflect on whether the logs it has would be enough to diagnose a production incident
- Dependency vulnerability scanners

**When this category matters most**: cross-cutting concerns. Anything the agent could implement *correctly in isolation* but *wrongly in context* — e.g. it adds a new database call in a hot path, or logs a secret, or breaks a layering rule.

## 3. Behaviour harness

**What it regulates**: does the application functionally do what the user actually needs?

**This is the elephant in the room.** The article is explicit that no one has fully solved this yet. The current default approach is:

- Feedforward: a functional spec, ranging from a one-line prompt to multi-file design docs
- Feedback: check the AI-generated test suite is green, has decent coverage, maybe mutation-test it; combine with manual testing

The problem: this puts a lot of faith in AI-generated tests, which can be green and meaningless. A test suite the agent wrote to make itself pass is not the same as a test suite that protects the user.

**Patterns that help (selectively)**:

- **Approved fixtures** ([reference](https://lexler.github.io/augmented-coding-patterns/patterns/approved-fixtures/)): the human approves a set of input → expected-output pairs once, and the agent's job is to make code that matches. Works well where outputs are deterministic and inspectable. Doesn't work for everything.
- **Concrete examples in the spec**: at least one input/output pair, even informal, beats prose-only specs.
- **Human review + manual exploratory testing**: still the floor. The article does not pretend otherwise.

**What to tell the user when planning a behaviour-heavy task**: the harness reduces toil but does not replace human review and manual testing for behaviour yet. Plan for both.

## Cross-cutting principles

### Computational vs inferential

| | Computational | Inferential |
|---|---|---|
| Speed | ms–s | s–min |
| Cost | low | high |
| Determinism | reliable | probabilistic |
| Best for | structural, syntactic, rule-based | semantic, judgment-based |
| Example guide | code mod, type stubs, LSP | AGENTS.md, skill, how-to |
| Example sensor | linter, test, structural check | review agent, LLM-as-judge |

Run computational controls early and often (cheap). Reserve inferential controls for places where they add semantic judgment that nothing else can — and accept they're non-deterministic.

### Keep quality left

Cost grows the further right a problem is found. Order of preference for catching an issue:

1. The agent's own self-correction loop (sensors fire, agent fixes silently)
2. Pre-commit / pre-push (still on the dev's machine)
3. CI pipeline post-integration
4. Code review by a human
5. QA / staging
6. Production

Anything that *can* run at level 1 *should* run at level 1. Don't put a fast linter in CI just because that's where linters traditionally live.

### Harnessability is not uniform

Some codebases are easier to harness than others. The article uses *ambient affordances* (Ned Letcher's term) for the structural properties that make an environment legible to an agent — strong typing, clear module boundaries, opinionated frameworks, etc.

Practical implication when planning a task: if the affected area lacks these affordances, accept that you'll need more inferential controls (more expensive, less reliable) to compensate, and the human review burden will be higher. Don't pretend a legacy area is as harnessable as a greenfield one.

### Ashby's Law of Requisite Variety

The regulator must have at least as much variety as the system it governs. An LLM agent can produce almost anything; harnessing all of it is impossible. The fix is not bigger harnesses — it's *narrower systems*. Service templates, opinionated stacks, and architectural constraints reduce variety so a comprehensive harness becomes feasible.

When a task crosses into an unconstrained area of the codebase, flag it. The right move may be to constrain the area first (via a template or refactor) before letting the agent loose.

## Common failure modes mapped to harness gaps

| Agent failure | Likely missing control |
|---|---|
| Wrong solution to wrong problem | Spec clarity (phase 1) — no harness compensates |
| Conventions violated | Maintainability guide (AGENTS.md, skill) |
| Hidden duplication | Maintainability sensor (semantic dup detector / review agent) |
| Module boundary violation | Architecture fitness sensor (structural test) |
| Performance regression | Architecture fitness sensor (perf test) |
| Logs unusable in prod | Architecture fitness guide + reflection sensor |
| Tests pass but feature broken | Behaviour — approved fixtures, human exploratory testing |
| Same mistake in next task | Steering loop not engaged — upgrade the harness |
