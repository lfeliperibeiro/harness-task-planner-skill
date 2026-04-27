# Installation guide / Guia de instalação

🇧🇷 [Português](#-português) · 🇺🇸 [English](#-english)

---

## 🇧🇷 Português

### Claude Code

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/lfeliperibeiro/harness-task-planner-skill.git /tmp/htp
cp -r /tmp/htp/harness-task-planner ~/.claude/skills/
```

Reinicie o Claude Code. Verifique com `/skills` no prompt.

### OpenAI Codex CLI

**Opção A — via skill-installer (recomendado para o time):**

```
$skill-installer install https://github.com/lfeliperibeiro/harness-task-planner-skill/tree/main/harness-task-planner
```

Reinicie o Codex.

**Opção B — manual:**

```bash
mkdir -p ~/.codex/skills
git clone https://github.com/lfeliperibeiro/harness-task-planner-skill.git /tmp/htp
cp -r /tmp/htp/harness-task-planner ~/.codex/skills/
```

Reinicie o Codex. Verifique com `/skills` ou `$` no prompt.

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

### Claude Code

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/lfeliperibeiro/harness-task-planner-skill.git /tmp/htp
cp -r /tmp/htp/harness-task-planner ~/.claude/skills/
```

Restart Claude Code. Verify with `/skills` in the prompt.

### OpenAI Codex CLI

**Option A — via skill-installer (recommended for teams):**

```
$skill-installer install https://github.com/lfeliperibeiro/harness-task-planner-skill/tree/main/harness-task-planner
```

Restart Codex.

**Option B — manual:**

```bash
mkdir -p ~/.codex/skills
git clone https://github.com/lfeliperibeiro/harness-task-planner-skill.git /tmp/htp
cp -r /tmp/htp/harness-task-planner ~/.codex/skills/
```

Restart Codex. Verify with `/skills` or `$` in the prompt.

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
