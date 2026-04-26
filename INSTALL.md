# Installation guide / Guia de instalação

🇧🇷 [Português](#-português) · 🇺🇸 [English](#-english)

---

## 🇧🇷 Português

### Claude.ai (web ou app)

1. Baixe o `.skill` da página de releases deste repo (ou empacote localmente — veja seção "Empacotar localmente" abaixo).
2. Abra Claude.ai → **Settings** → **Capabilities** → **Skills**.
3. Clique em **Upload skill** e selecione `harness-task-planner.skill`.
4. Pronto. A skill dispara automaticamente quando você pede para criar tarefa para um agente.

### Claude Code

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/lfeliperibeiro/harness-plan-task-skill.git /tmp/htp
cp -r /tmp/htp/harness-task-planner ~/.claude/skills/
```

Reinicie o Claude Code. Verifique com `/skills` no prompt.

### OpenAI Codex CLI

**Opção A — via skill-installer (recomendado para o time):**

```
$skill-installer install https://github.com/lfeliperibeiro/harness-plan-task-skill/tree/main/harness-task-planner
```

Reinicie o Codex.

**Opção B — manual:**

```bash
mkdir -p ~/.codex/skills
git clone https://github.com/lfeliperibeiro/harness-plan-task-skill.git /tmp/htp
cp -r /tmp/htp/harness-task-planner ~/.codex/skills/
```

Reinicie o Codex. Verifique com `/skills` ou `$` no prompt.

### Cursor

Cursor não tem suporte nativo a skills. Cole o conteúdo do `harness-task-planner/SKILL.md` no `.cursorrules` da raiz do projeto, ou crie um `.cursor/rules/harness-task-planner.mdc` com o mesmo conteúdo.

### Aider

Aider lê `CONVENTIONS.md` ou um arquivo passado via `--read`. Cole o conteúdo do `SKILL.md` em um desses, ou rode:

```bash
aider --read harness-task-planner/SKILL.md
```

### Outros agentes

Se o agente lê `AGENTS.md`, `CLAUDE.md`, ou outro arquivo de instruções persistente, basta concatenar o conteúdo do `SKILL.md` nele. A skill é texto plano e funciona em qualquer agente capaz de seguir instruções.

### Empacotar localmente como `.skill`

`.skill` é apenas um zip. Pra gerar:

```bash
cd harness-task-planner
zip -r ../harness-task-planner.skill .
cd ..
```

### Atualização

```bash
# Em qualquer um dos casos acima:
cd ~/.claude/skills/harness-task-planner   # ou ~/.codex/skills/harness-task-planner
git pull   # se você clonou
# ou copie a versão nova manualmente
```

Reinicie o agente após atualizar.

### Desinstalação

```bash
rm -rf ~/.claude/skills/harness-task-planner
# ou
rm -rf ~/.codex/skills/harness-task-planner
```

---

## 🇺🇸 English

### Claude.ai (web or app)

1. Download the `.skill` from this repo's releases page (or package it locally — see "Package locally" below).
2. Open Claude.ai → **Settings** → **Capabilities** → **Skills**.
3. Click **Upload skill** and select `harness-task-planner.skill`.
4. Done. The skill triggers automatically when you ask to create a task for an agent.

### Claude Code

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/lfeliperibeiro/harness-plan-task-skill.git /tmp/htp
cp -r /tmp/htp/harness-task-planner ~/.claude/skills/
```

Restart Claude Code. Verify with `/skills` in the prompt.

### OpenAI Codex CLI

**Option A — via skill-installer (recommended for teams):**

```
$skill-installer install https://github.com/lfeliperibeiro/harness-plan-task-skill/tree/main/harness-task-planner
```

Restart Codex.

**Option B — manual:**

```bash
mkdir -p ~/.codex/skills
git clone https://github.com/lfeliperibeiro/harness-plan-task-skill.git /tmp/htp
cp -r /tmp/htp/harness-task-planner ~/.codex/skills/
```

Restart Codex. Verify with `/skills` or `$` in the prompt.

### Cursor

Cursor has no native skill support. Paste the contents of `harness-task-planner/SKILL.md` into your project's `.cursorrules`, or create `.cursor/rules/harness-task-planner.mdc` with the same content.

### Aider

Aider reads `CONVENTIONS.md` or a file passed via `--read`. Paste the `SKILL.md` content into either, or run:

```bash
aider --read harness-task-planner/SKILL.md
```

### Other agents

If the agent reads `AGENTS.md`, `CLAUDE.md`, or any other persistent instruction file, just concatenate the `SKILL.md` content into it. The skill is plain text and works in any agent capable of following instructions.

### Package locally as `.skill`

`.skill` is just a zip. To produce one:

```bash
cd harness-task-planner
zip -r ../harness-task-planner.skill .
cd ..
```

### Updating

```bash
# In any of the cases above:
cd ~/.claude/skills/harness-task-planner   # or ~/.codex/skills/harness-task-planner
git pull   # if you cloned
# or copy the new version manually
```

Restart the agent after updating.

### Uninstalling

```bash
rm -rf ~/.claude/skills/harness-task-planner
# or
rm -rf ~/.codex/skills/harness-task-planner
```
