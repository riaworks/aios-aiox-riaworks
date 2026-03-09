# rwIntelContextLog — Log de Injecao Code-Intel

**Funcao:** `rwIntelContextLog(cwd, { toolName, filePath, xml })`
**Env var:** `RW_INTEL_CONTEXT_LOG=1`
**Arquivo de log:** `.logs/rw-intel-context-log.log`
**Definida em:** `.claude/hooks/code-intel-pretool.cjs`

---

## O que e

Log que registra o **XML `<code-intel-context>` injetado** pelo hook de code-intel a cada operacao Write ou Edit, e tambem **prompts de agentes** carregados via Skill activation. Responde a pergunta: "que contexto de codigo foi injetado quando o agente editou este arquivo?" e "que prompt de agente foi carregado?"

Nao registra regras do Synapse (para isso use `rwSynapseTrace`). Nao registra o log operacional (para isso use `rwHooksLog`).

## Por que existe

O hook `code-intel-pretool.cjs` injeta contexto invisivel sobre entidades, referencias e dependencias do arquivo sendo editado. Sem log, e impossivel saber:

- Se o code-intel encontrou dados para o arquivo
- Quais entidades, referencias e dependencias foram injetadas
- Se o entity-registry tem cobertura para aquele arquivo
- Qual ferramenta (Write/Edit) disparou a injecao

## Quando dispara

Em operacoes **Write**, **Edit** e **Skill** do Claude Code. Nao dispara em Read, Bash, Grep, etc.

- **Write/Edit:** Loga o XML `<code-intel-context>` injetado via `rwIntelContextLog()`
- **Skill:** Loga o prompt completo do agente carregado via `rwSkillLog()` (ex: `/AIOX:agents:pm` loga o conteudo de `.aiox-core/development/agents/pm.md`)

## Como ativar

Adicione `RW_INTEL_CONTEXT_LOG=1` no comando do hook em `.claude/settings.local.json`:

```json
"PreToolUse": [{
  "hooks": [{
    "type": "command",
    "command": "RW_INTEL_CONTEXT_LOG=1 node .claude/hooks/code-intel-pretool.cjs"
  }],
  "matcher": "Write|Edit|Skill"
}]
```

## Como desativar

Remova `RW_INTEL_CONTEXT_LOG=1` do comando:

```json
"command": "node .claude/hooks/code-intel-pretool.cjs"
```

## Como visualizar

```bash
# Log completo
cat .logs/rw-intel-context-log.log

# Tempo real
tail -f .logs/rw-intel-context-log.log

# Apenas arquivos editados
grep "TOOL:" .logs/rw-intel-context-log.log

# Apenas entidades encontradas
grep -A 3 "existing-entity" .logs/rw-intel-context-log.log
```

## Formato do log

```
================================================================================
[2026-03-06T17:10:45.123Z] CODE-INTEL INJECTION -- PreToolUse
================================================================================
TOOL: Edit -> .claude/hooks/synapse-engine.cjs

<code-intel-context>
  <target-file>.claude/hooks/synapse-engine.cjs</target-file>
  <existing-entity>
    <path>.claude/hooks/synapse-engine.cjs</path>
    <purpose>SYNAPSE Hook Entry Point</purpose>
  </existing-entity>
  <referenced-by count="2">
    <ref file=".claude/settings.local.json" context="Registered as UserPromptSubmit hook" />
    <ref file="aios-aiox-riaworks/rw-synapse-trace.md" context="Documentation" />
  </referenced-by>
</code-intel-context>
```

## Comportamento

- **Opt-in:** so grava quando `RW_INTEL_CONTEXT_LOG=1`
- **Fire-and-forget:** nunca bloqueia a execucao do hook
- **Auto-create:** cria `.logs/` com `.gitignore` se nao existir
- **Append-only:** nunca sobrescreve, sempre adiciona
- **Condicional:** so gera entry quando ha XML (entity encontrada)

## Relacao com outros logs

| Log | O que captura | Evento |
|-----|---------------|--------|
| `rw-hooks-log` | Status operacional dos hooks | UserPromptSubmit |
| `rw-synapse-trace` | XML do Synapse injetado | UserPromptSubmit |
| **`rw-intel-context-log`** | **XML do code-intel + prompts de agentes** | **PreToolUse (Write/Edit/Skill)** |
| `rw-context-log-full` | Tudo unificado | Ambos |

---

*RIAWORKS — 2026-03-06*
