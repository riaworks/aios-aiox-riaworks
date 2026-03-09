# Fix: Skill/Agent Activation Logging

**Data:** 2026-03-08
**Componente:** `.claude/hooks/code-intel-pretool.cjs`, `.claude/settings.local.json`
**Severidade:** Enhancement (logging extension)

---

## Problema

Quando um agente AIOX e ativado via `/AIOX:agents:pm` (ou qualquer `/AIOX:agents:{name}`), o prompt do agente (YAML completo com persona, comandos, dependencias) e carregado pelo Skill tool do Claude Code. Esse conteudo:

- NAO passa por nenhum hook existente
- NAO e armazenado no JSONL do Claude
- E completamente invisivel nos logs

O watcher (`watch-context.js`) mostra synapse-rules, code-intel, metricas — mas NAO mostra o prompt do agente carregado.

## Solucao

### 1. Adicionar `Skill` ao matcher do PreToolUse

**Arquivo:** `.claude/settings.local.json`

O hook `code-intel-pretool.cjs` so dispara em `Write|Edit`. Adicionando `Skill` ao matcher, o hook tambem dispara quando um Skill/agente e ativado.

```json
"PreToolUse": [{
  "hooks": [{
    "type": "command",
    "command": "RW_INTEL_CONTEXT_LOG=1 RW_CONTEXT_LOG_FULL=1 node .claude/hooks/code-intel-pretool.cjs"
  }],
  "matcher": "Write|Edit|Skill"
}]
```

**Antes:** `"matcher": "Write|Edit"`
**Depois:** `"matcher": "Write|Edit|Skill"`

### 2. Adicionar `rwSkillLog()` ao code-intel-pretool.cjs

**Arquivo:** `.claude/hooks/code-intel-pretool.cjs`

Nova funcao que detecta Skill tool, resolve o arquivo do agente, le o conteudo e loga.

```javascript
/**
 * Log Skill/agent activation to .logs/ (fire-and-forget).
 * Reads the agent prompt file and logs its full content.
 *
 * @param {string} cwd - Project root
 * @param {string} skillName - Skill identifier (e.g., "AIOX:agents:pm")
 * @param {string} [skillArgs] - Optional skill arguments
 */
function rwSkillLog(cwd, skillName, skillArgs) {
  const intelLog = process.env.RW_INTEL_CONTEXT_LOG === '1';
  const fullLog = process.env.RW_CONTEXT_LOG_FULL === '1';
  if (!intelLog && !fullLog) return;
  try {
    const logsDir = path.join(cwd, '.logs');
    ensureLogsDir(logsDir);

    // Resolve skill file path
    let content = '(skill content not available — built-in or file not found)';
    let resolvedPath = skillName;

    // AIOX agents: AIOX:agents:name -> .aiox-core/development/agents/name.md
    const agentMatch = skillName.match(/^AIOX:agents:(.+)$/);
    if (agentMatch) {
      const agentFile = path.join(cwd, '.aiox-core', 'development', 'agents', `${agentMatch[1]}.md`);
      if (fs.existsSync(agentFile)) {
        content = fs.readFileSync(agentFile, 'utf8');
        resolvedPath = `.aiox-core/development/agents/${agentMatch[1]}.md`;
      }
    }

    const ts = new Date().toISOString();
    const sep = '='.repeat(80);
    const argsLine = skillArgs ? `\nARGS: ${skillArgs}` : '';

    if (intelLog) {
      const entry = [
        sep,
        `[${ts}] SKILL ACTIVATION -- PreToolUse`,
        sep,
        `SKILL: ${skillName} -> ${resolvedPath}${argsLine}`,
        '',
        content,
        '',
        '',
      ].join('\n');
      fs.appendFileSync(path.join(logsDir, 'rw-intel-context-log.log'), entry);
    }

    if (fullLog) {
      const entry = [
        '-'.repeat(80),
        `[${ts}] PreToolUse -- Skill`,
        '-'.repeat(80),
        `[SKILL] ${skillName} -> ${resolvedPath}${argsLine}`,
        '',
        '[AGENT PROMPT] (loaded by Skill tool)',
        content,
        '',
        '',
      ].join('\n');
      fs.appendFileSync(path.join(logsDir, 'rw-context-log-full.log'), entry);
    }
  } catch (_) {
    // Fire-and-forget
  }
}
```

### 3. Adicionar handling de Skill no `main()`

**Arquivo:** `.claude/hooks/code-intel-pretool.cjs`

No inicio de `main()`, antes do check de `TARGET_TOOLS`, detectar Skill e logar:

```javascript
async function main() {
  const input = await readStdin();
  const toolName = input && input.tool_name;
  if (!toolName) return;

  const cwd = input.cwd || process.cwd();

  // ── Skill activation — log agent prompt (no injection) ──
  if (toolName === 'Skill') {
    const skillName = input.tool_input && input.tool_input.skill;
    const skillArgs = input.tool_input && input.tool_input.args;
    if (skillName) {
      rwSkillLog(cwd, skillName, skillArgs);
    }
    return; // No additionalContext for Skill — just log
  }

  // ── Write/Edit — inject code-intel context ──
  if (!TARGET_TOOLS.has(toolName)) return;

  // ... rest of existing code unchanged ...
}
```

**Importante:** Para Skill, o hook apenas loga — NAO injeta `additionalContext`.

### 4. Adicionar `rwSkillLog` ao module.exports

```javascript
module.exports = { readStdin, main, run, rwIntelContextLog, rwSkillLog, HOOK_TIMEOUT_MS, TARGET_TOOLS };
```

---

## Mapeamento de Skill para arquivo

| Skill Name | Arquivo |
|------------|---------|
| `AIOX:agents:pm` | `.aiox-core/development/agents/pm.md` |
| `AIOX:agents:dev` | `.aiox-core/development/agents/dev.md` |
| `AIOX:agents:qa` | `.aiox-core/development/agents/qa.md` |
| `AIOX:agents:{name}` | `.aiox-core/development/agents/{name}.md` |
| Skills built-in (commit, simplify) | Nao resolve arquivo — loga apenas o nome |

## Formato do log

### No rw-intel-context-log.log:

```
================================================================================
[2026-03-08T21:33:53.654Z] SKILL ACTIVATION -- PreToolUse
================================================================================
SKILL: AIOX:agents:pm -> .aiox-core/development/agents/pm.md

# pm

ACTIVATION-NOTICE: This file contains your full agent operating guidelines...
(conteudo completo do agente)
```

### No rw-context-log-full.log:

```
--------------------------------------------------------------------------------
[2026-03-08T21:33:53.654Z] PreToolUse -- Skill
--------------------------------------------------------------------------------
[SKILL] AIOX:agents:pm -> .aiox-core/development/agents/pm.md

[AGENT PROMPT] (loaded by Skill tool)
# pm

ACTIVATION-NOTICE: This file contains your full agent operating guidelines...
(conteudo completo do agente)
```

## Verificacao

```bash
# Simular ativacao de Skill
echo '{"tool_name":"Skill","tool_input":{"skill":"AIOX:agents:pm"},"cwd":"DIR-MEU-PROJETO"}' \
  | RW_INTEL_CONTEXT_LOG=1 RW_CONTEXT_LOG_FULL=1 node .claude/hooks/code-intel-pretool.cjs 2>/dev/null

# Verificar se foi logado
grep "SKILL ACTIVATION" .logs/rw-intel-context-log.log && echo "OK" || echo "FAIL"
grep "PreToolUse -- Skill" .logs/rw-context-log-full.log && echo "OK" || echo "FAIL"
```

## Arquivos modificados

| Arquivo | Mudanca |
|---------|---------|
| `.claude/settings.local.json` | matcher `Write\|Edit` → `Write\|Edit\|Skill` |
| `.claude/hooks/code-intel-pretool.cjs` | `rwSkillLog()`, Skill handling em `main()`, `module.exports` |

---

*Documentado em 2026-03-08 — RIAWORKS*
