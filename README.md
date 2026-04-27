# harness-task-planner

> **Plan coding-agent tasks using Birgitta Böckeler's Harness Engineering framework.**
> A skill that turns a fuzzy "make the agent do X" request into a structured task document with explicit feedforward guides and feedback sensors — agent-agnostic, fullstack, with non-negotiable quality rules baked in.

Source framework: [Harness engineering for coding agent users](https://martinfowler.com/articles/harness-engineering.html) — Birgitta Böckeler, April 2026.

---

🇧🇷 [Português](#-português) · 🇺🇸 [English](#-english)

---

## 🇧🇷 Português

### O que é

Esta skill orienta um agente assistente (Claude, Codex, Cursor, Aider, Cline) a planejar tarefas de codificação seguindo o framework de **harness engineering**: combinar *guides* (controles que orientam o agente antes de agir) com *sensors* (controles que observam depois de agir e permitem auto-correção), distribuídos em três categorias — maintainability, architecture fitness e behaviour.

A skill produz um **documento de tarefa estruturado** que vira input pro agente implementador e referência pra revisão humana.

### Por que existe

Agentes de código falham de formas previsíveis: misdiagnose, over-engineer, drift de convenções, testes verdes-mas-vazios, vazamento de segredos. A resposta não é supervisionar mais — é **wrappar** a tarefa em controles explícitos antes de soltar o agente. Esta skill operacionaliza isso pro time todo.

### Compatibilidade

| Agente | Suporte | Como instalar |
|---|---|---|
| Claude.ai (web/app) | ✅ Nativo | Upload do `.skill` em Settings → Capabilities → Skills |
| Claude Code | ✅ Nativo | `~/.claude/skills/harness-task-planner/` |
| OpenAI Codex CLI | ✅ Nativo | `~/.codex/skills/harness-task-planner/` |
| Cursor / Aider / outros | ⚠️ Via instrução manual | Cole o conteúdo do `SKILL.md` no `AGENTS.md` ou equivalente |

Veja [INSTALL.md](INSTALL.md) pros comandos exatos.

### Como funciona — o que cada etapa faz

A skill executa 5 fases sequenciais. Pular etapa é o mecanismo principal de falha — cada uma resolve um problema diferente.

#### Phase 0 — Escolher a língua da conversa

**O que faz:** logo no início, pergunta ao humano em qual língua quer conversar (pt-BR ou en). Conduz toda a conversa subsequente naquela língua.

**Por que existe:** time PT prefere conversar em PT, mas o **doc de tarefa final e os prompts dos subagentes são sempre em inglês** — agentes seguem instruções estruturadas em inglês com mais consistência, e os termos técnicos do framework (guides, sensors, mutation kill rate, fitness functions) ficam ancorados ao artigo original. A separação resolve os dois objetivos sem comprometer nenhum.

**Output:** língua escolhida, marcada pra duração da sessão.

#### Phase 1 — Capturar a tarefa e checar pré-requisitos

**O que faz:** elicita do humano *outcome*, *critérios de sucesso*, *blast radius*, *qual agente* vai rodar, e se a tarefa é trivial ou substantiva.

**Por que existe:** o artigo é explícito — "correctness is outside any sensor's remit if the human didn't clearly specify what they wanted". Um spec vago é a primeira causa de misdiagnose. Esta fase também roda **pre-flight checks**: ferramenta de mutation testing instalada? linter de complexidade configurado? coverage mede no diff? `.env*` no `.gitignore`? Se algum não, a tarefa é bloqueada antes de começar — não vale a pena escrever código que não vai conseguir provar que está pronto.

**Output:** confirmação dos requisitos com o humano, ou lista de gaps bloqueantes a resolver primeiro.

#### Phase 2 — Montar guides (controles feedforward)

**O que faz:** identifica e referencia os documentos que orientam o agente *antes* dele escrever código, em três categorias:

- **2a. Maintainability** — `AGENTS.md`/`CLAUDE.md`/`.cursorrules`, skills, how-tos, code mods, language servers
- **2b. Architecture fitness** — perf budgets, SLOs, convenções de logging/observability, regras de boundary
- **2c. Behaviour** — spec funcional, exemplos input/output, fixtures aprovadas, critérios de aceitação testáveis

**Por que existe:** guides reduzem a probabilidade do agente errar na primeira tentativa. É mais barato evitar erro com um doc bem escrito do que detectá-lo depois com um sensor.

**Output:** lista de guides relevantes com links/paths, e gaps registrados pra steering loop.

#### Phase 3 — Montar sensors (controles feedback)

**O que faz:** define o que vai observar o output do agente *depois* dele agir, classificado em **block** (precisa passar pra tarefa estar done) ou **report** (roda e expõe, humano decide).

- **3a. Maintainability** — lint, typecheck, testes, **cobertura ≥80%**, **complexidade ciclomática ≤10**, **arquivo ≤400 LOC**
- **3b. Architecture fitness** — testes estruturais (boundaries), perf tests, dep audit, axe-core (a11y) e bundle size pra frontend
- **3c. Behaviour** — **TDD obrigatório**, **mutation kill rate ≥70%**, revisão humana, exploratory testing, contrato back↔front pra fullstack

**Por que existe:** sensors permitem que o próprio agente se auto-corrija antes do humano olhar. Um sensor que produz mensagem otimizada pra LLM ("a função X tem complexidade 14, refatore extraindo o bloco do switch") é prompt injection positivo — o agente lê e age.

**Output:** matriz de sensors com tooling escolhido por linguagem e classificação block/report.

#### Phase 4 — Distribuir no lifecycle

**O que faz:** decide *onde* cada controle roda — dentro do loop de auto-correção do agente, antes do agente declarar done, no pre-push do humano, ou no CI existente.

**Por que existe:** o princípio do artigo é "keep quality left" — quanto mais cedo um problema é detectado, mais barato é corrigir. **Esta skill assume hoje que CI/CD não está sendo modificado** — todos os blocks rodam localmente antes do PR. Quando o time decidir mexer em CI, os mesmos sensors migram pra direita como rede de segurança; o harness não muda, só onde executa.

**Output:** seção 6 do task document com a distribuição explícita.

#### Phase 5 — Produzir o documento de tarefa

**O que faz:** preenche o template em [`assets/task-template.md`](harness-task-planner/assets/task-template.md) com tudo coletado nas fases anteriores. O documento tem 10 seções: outcome, critérios, scope (com layer back/front/fullstack e contexto SDD), guides, sensors, distribuição lifecycle, checklist humano, gaps, prompt prefix do implementador, e configuração do pipeline de subagentes.

**Por que existe:** o documento é o artefato físico que vira input pro agente, ticket no board e arquivo versionado em `.tasks/<slug>.md`. Sem ele, o conhecimento do planejamento mora só na cabeça de quem planejou.

**Output:** markdown único pronto pra colar num ticket, mandar pro agente ou commitar.

### Pipeline de subagentes (obrigatório)

Toda tarefa não-trivial passa por **três invocações independentes**:

1. **Implementador** — faz o trabalho seguindo TDD, roda os blocks, produz diff + summary
2. **Code reviewer** — lê só o diff e o doc da tarefa (não vê o raciocínio do implementador), comenta design/duplicação/edge cases
3. **QA** — exercita o sistema rodando contra back/front/contrato, reporta Works/Breaks/Unclear

**Os três são *report*, não *block*.** Os blocks computacionais (cobertura, mutation, complexidade, file size, `.env`, lint) seguem bloqueando sozinhos. **O humano é o único gate semântico** — três agentes concordando é evidência, não prova.

Pra tarefas triviais (typo, version bump, rename), só implementador + revisão humana; os blocks computacionais continuam valendo.

### Regras inegociáveis

Seis regras que valem em toda tarefa, independente do tamanho:

| # | Regra | Threshold |
|---|---|---|
| 1 | TDD obrigatório (red → green → refactor) | 100% das mudanças |
| 2 | Cobertura + mutation testing no código tocado | ≥80% cobertura, ≥70% mutation kill |
| 3 | `.env*` off-limits ao agente | Nunca lê, edita, copia ou cita |
| 4 | Complexidade ciclomática | ≤10 por função |
| 5 | Sem god files | ≤400 LOC por arquivo |
| 6 | Status reports estruturados obrigatórios | Tabela pass/fail por check do implementador, status por categoria do code reviewer e QA |

Cada regra está pareada a tooling concreto por linguagem dentro do `SKILL.md` — funciona pra back (Node, Python, Java, Kotlin, Go, C#, Rust, PHP) e front (TS/React/Vue/Svelte).

A regra 6 garante que o agente nunca declara "pronto" sem prova, e nunca revisa/QA sem registro estruturado. Toda execução do implementador termina com tabela mostrando, linha por linha, se cada check passou ou falhou. Code reviewer e QA terminam com tabela própria — Y/N por categoria (design, duplicação, edge cases, etc. pro reviewer; happy path, error paths, a11y, contrato back/front, etc. pro QA), mais um veredito final (READY_TO_MERGE / MINOR_FIXES_NEEDED / MAJOR_REWORK_NEEDED). O humano lê três tabelas curtas e sabe em segundos onde focar — sem ter que reler prosa de três relatórios.

### Integração com SDD (Spec-Driven Development)

A skill compõe com [GitHub Spec Kit](https://github.com/github/spec-kit), Kiro e similares. SDD responde "estamos construindo a coisa certa?" produzindo `spec.md`/`plan.md`/`tasks.md` upstream. Esta skill responde "o agente vai executar a tarefa com segurança?" — uma harness por tarefa.

Mapeamento:
- `constitution.md` (SDD) ↔ regras inegociáveis (esta skill)
- `spec.md` ↔ behaviour guide (Phase 2c)
- `plan.md` ↔ architecture fitness guide (Phase 2b)
- `tasks.md` ↔ uma invocação desta skill por entrada

### Steering loop

A pergunta que fecha o ciclo: *quando o agente erra, isso vira um guide ou sensor pra próxima vez?* Erros recorrentes são sinal de controle ausente, não de agente ruim. A seção 8 do task document existe pra registrar esse aprendizado.

### Estrutura

```
harness-task-planner/
├── SKILL.md                       # 5 fases, 5 regras, pipeline de 3 subagentes
├── references/
│   └── categories.md              # Aprofundamento das 3 categorias (maintainability/fitness/behaviour)
└── assets/
    └── task-template.md           # Template de 10 seções pro doc de tarefa
```

### Contribuir

PRs bem-vindos. Mudanças na skill devem rodar contra pelo menos uma tarefa real antes de mergear — esta skill foi construída iterando, não no papel.

### Licença

[MIT](LICENSE)

---

## 🇺🇸 English

### What it is

This skill guides a coding agent (Claude, Codex, Cursor, Aider, Cline) to plan tasks following the **harness engineering** framework: combine *guides* (controls that steer the agent before it acts) with *sensors* (controls that observe after it acts and enable self-correction), distributed across three categories — maintainability, architecture fitness, and behaviour.

The skill produces a **structured task document** that becomes input for the implementer agent and reference for human review.

### Why it exists

Coding agents fail in predictable ways: misdiagnosis, over-engineering, convention drift, green-but-empty tests, secret leaks. The answer isn't more supervision — it's **wrapping** the task in explicit controls before letting the agent loose. This skill operationalises that for the whole team.

### Compatibility

| Agent | Support | How to install |
|---|---|---|
| Claude.ai (web/app) | ✅ Native | Upload `.skill` in Settings → Capabilities → Skills |
| Claude Code | ✅ Native | `~/.claude/skills/harness-task-planner/` |
| OpenAI Codex CLI | ✅ Native | `~/.codex/skills/harness-task-planner/` |
| Cursor / Aider / others | ⚠️ Via manual instruction | Paste `SKILL.md` content into `AGENTS.md` or equivalent |

See [INSTALL.md](INSTALL.md) for exact commands.

### How it works — what each phase does

The skill runs 5 sequential phases. Skipping a phase is the main failure mode — each one solves a different problem.

#### Phase 0 — Choose conversation language

**What it does:** right at the start, asks the human which language to chat in (pt-BR or en). Conducts all subsequent conversation in that language.

**Why it exists:** PT-speaking teams prefer to chat in PT, but the **final task document and subagent prompts are always in English** — agents follow structured instructions more consistently in English, and the framework's technical terms (guides, sensors, mutation kill rate, fitness functions) stay anchored to the source article. The separation solves both goals without compromising either.

**Output:** chosen language, sticky for the session.

#### Phase 1 — Capture the task and check pre-flight

**What it does:** elicits from the human the *outcome*, *success criteria*, *blast radius*, *which agent* will run it, and whether the task is trivial or substantial.

**Why it exists:** the article is explicit — "correctness is outside any sensor's remit if the human didn't clearly specify what they wanted". A vague spec is the #1 cause of misdiagnosis. This phase also runs **pre-flight checks**: is a mutation testing tool installed? complexity linter configured? coverage measured on the diff? `.env*` in `.gitignore`? If any answer is no, the task is blocked before it starts — there's no point writing code you can't prove is done.

**Output:** confirmed requirements with the human, or a list of blocking gaps to resolve first.

#### Phase 2 — Assemble guides (feedforward controls)

**What it does:** identifies and references the documents that steer the agent *before* it writes code, in three categories:

- **2a. Maintainability** — `AGENTS.md`/`CLAUDE.md`/`.cursorrules`, skills, how-tos, code mods, language servers
- **2b. Architecture fitness** — perf budgets, SLOs, logging/observability conventions, boundary rules
- **2c. Behaviour** — functional spec, input/output examples, approved fixtures, testable acceptance criteria

**Why it exists:** guides reduce the probability the agent gets it wrong on the first attempt. Preventing an error with a well-written doc is cheaper than detecting it afterwards with a sensor.

**Output:** list of relevant guides with links/paths, plus gaps logged for the steering loop.

#### Phase 3 — Assemble sensors (feedback controls)

**What it does:** defines what observes the agent's output *after* it acts, classified as **block** (must pass for the task to be done) or **report** (runs and surfaces; human decides).

- **3a. Maintainability** — lint, typecheck, tests, **coverage ≥80%**, **cyclomatic complexity ≤10**, **file ≤400 LOC**
- **3b. Architecture fitness** — structural tests (boundaries), perf tests, dep audit, axe-core (a11y) and bundle size for frontend
- **3c. Behaviour** — **mandatory TDD**, **mutation kill rate ≥70%**, human review, exploratory testing, back↔front contract for fullstack

**Why it exists:** sensors let the agent self-correct before the human looks. A sensor that produces an LLM-optimised message ("function X has complexity 14, refactor by extracting the switch block") is positive prompt injection — the agent reads and acts.

**Output:** sensor matrix with language-specific tooling and block/report classification.

#### Phase 4 — Distribute across the lifecycle

**What it does:** decides *where* each control runs — inside the agent's self-correction loop, before the agent declares done, in the developer's pre-push, or in existing CI.

**Why it exists:** the article's principle is "keep quality left" — the earlier a problem is found, the cheaper it is to fix. **This skill currently assumes CI/CD is not being modified** — all blocks run locally before the PR. When the team decides to touch CI, the same sensors move right as a safety net; the harness itself doesn't change, only where it executes.

**Output:** section 6 of the task document with explicit distribution.

#### Phase 5 — Produce the task document

**What it does:** fills in the template at [`assets/task-template.md`](harness-task-planner/assets/task-template.md) with everything gathered in earlier phases. The document has 10 sections: outcome, criteria, scope (with back/front/fullstack layer and SDD context), guides, sensors, lifecycle distribution, human checklist, gaps, implementer prompt prefix, and subagent pipeline configuration.

**Why it exists:** the document is the physical artefact that becomes agent input, ticket on the board, and a versioned file at `.tasks/<slug>.md`. Without it, planning knowledge lives only in the planner's head.

**Output:** a single markdown ready to paste into a ticket, hand to the agent, or commit.

### Subagent pipeline (mandatory)

Every non-trivial task runs through **three independent invocations**:

1. **Implementer** — does the work following TDD, runs the blocks, produces diff + summary
2. **Code reviewer** — reads only the diff and task doc (does not see implementer's reasoning), comments on design/duplication/edge cases
3. **QA** — exercises the system against back/front/contract, reports Works/Breaks/Unclear

**All three are *report*, not *block*.** The computational blocks (coverage, mutation, complexity, file size, `.env`, lint) keep blocking on their own. **The human is the only semantic gate** — three agents agreeing is evidence, not proof.

For trivial tasks (typo, version bump, rename), only implementer + human review; computational blocks still apply.

### Non-negotiable rules

Six rules that apply to every task, regardless of size:

| # | Rule | Threshold |
|---|---|---|
| 1 | Mandatory TDD (red → green → refactor) | 100% of changes |
| 2 | Coverage + mutation testing on touched code | ≥80% coverage, ≥70% mutation kill |
| 3 | `.env*` off-limits to agent | Never read, edited, copied, or cited |
| 4 | Cyclomatic complexity | ≤10 per function |
| 5 | No god files | ≤400 LOC per file |
| 6 | Mandatory structured status reports | Implementer pass/fail table; code reviewer and QA status tables by category |

Each rule is paired with concrete tooling per language inside `SKILL.md` — works for backend (Node, Python, Java, Kotlin, Go, C#, Rust, PHP) and frontend (TS/React/Vue/Svelte).

Rule 6 guarantees the agent never declares "done" without proof, and never reviews/QAs without a structured record. Every implementer run ends with a table showing, row by row, whether each check passed or failed. Code reviewer and QA each end with their own status table — Y/N by category (design, duplication, edge cases, etc. for the reviewer; happy path, error paths, a11y, back/front contract, etc. for QA), plus an overall verdict (READY_TO_MERGE / MINOR_FIXES_NEEDED / MAJOR_REWORK_NEEDED). The human reads three short tables and knows in seconds where to focus — no need to re-read three reports of prose.

### SDD (Spec-Driven Development) integration

The skill composes with [GitHub Spec Kit](https://github.com/github/spec-kit), Kiro, and similar. SDD answers "are we building the right thing?" by producing upstream `spec.md`/`plan.md`/`tasks.md`. This skill answers "will the agent execute the task safely?" — one harness per task.

Mapping:
- `constitution.md` (SDD) ↔ non-negotiable rules (this skill)
- `spec.md` ↔ behaviour guide (Phase 2c)
- `plan.md` ↔ architecture fitness guide (Phase 2b)
- `tasks.md` ↔ one invocation of this skill per entry

### Steering loop

The cycle-closing question: *when the agent fails, does that become a guide or sensor for next time?* Recurring failures signal a missing control, not a bad agent. Section 8 of the task document exists to log this learning.

### Structure

```
harness-task-planner/
├── SKILL.md                       # 5 phases, 5 rules, 3-subagent pipeline
├── references/
│   └── categories.md              # Deeper dive on the 3 categories (maintainability/fitness/behaviour)
└── assets/
    └── task-template.md           # 10-section template for the task doc
```

### Contributing

PRs welcome. Skill changes should run against at least one real task before merging — this skill was built by iterating, not on paper.

### License

[MIT](LICENSE)
