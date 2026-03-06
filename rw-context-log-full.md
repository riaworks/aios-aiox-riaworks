# rwContextLogFull — Log Unificado de Todo Contexto Injetado

**Funcoes:** `rwContextLogFull()` (synapse-engine.cjs) + `rwIntelContextLog()` (code-intel-pretool.cjs)
**Env var:** `RW_CONTEXT_LOG_FULL=1`
**Arquivo de log:** `.logs/rw-context-log-full.log`
**Definida em:** Ambos os hooks

---

## O que e

Log unificado que registra **tudo que e injetado** no Claude Code a cada interacao. Responde a pergunta: "qual o contexto completo que o Claude esta recebendo?"

Substitui a necessidade de ativar `RW_HOOKS_LOG`, `RW_SYNAPSE_TRACE` e `RW_INTEL_CONTEXT_LOG` individualmente.

## O que captura

| Secao | Fonte | Quando |
|-------|-------|--------|
| `[USER PROMPT]` | Texto do usuario | Cada prompt |
| `[SESSION]` | ID da sessao + bracket | Cada prompt |
| `[SYNAPSE INJECTION]` | XML `<synapse-rules>` completo | Cada prompt |
| `[STATIC CONTEXT]` | Listagem de CLAUDE.md, rules/*.md, MEMORY.md | Cada prompt |
| `[CODE-INTEL INJECTION]` | XML `<code-intel-context>` | Cada Write/Edit |

## Como ativar

Adicione `RW_CONTEXT_LOG_FULL=1` em **ambos** os hooks em `.claude/settings.local.json`:

```json
{
  "hooks": {
    "PreToolUse": [{
      "hooks": [{
        "type": "command",
        "command": "RW_CONTEXT_LOG_FULL=1 node .claude/hooks/code-intel-pretool.cjs"
      }],
      "matcher": "Write|Edit"
    }],
    "UserPromptSubmit": [{
      "hooks": [{
        "type": "command",
        "command": "RW_CONTEXT_LOG_FULL=1 node .claude/hooks/synapse-engine.cjs"
      }]
    }]
  }
}
```

**Substituindo os individuais:** se ativar o full, pode remover os demais:

```json
"command": "RW_CONTEXT_LOG_FULL=1 node .claude/hooks/synapse-engine.cjs"
```

Em vez de:

```json
"command": "RW_HOOKS_LOG=1 RW_SYNAPSE_TRACE=1 node .claude/hooks/synapse-engine.cjs"
```

## Como desativar

Remova `RW_CONTEXT_LOG_FULL=1` de ambos os comandos de hook.

## Como visualizar

```bash
# Log completo
cat .logs/rw-context-log-full.log

# Tempo real (melhor para debug interativo)
tail -f .logs/rw-context-log-full.log

# Apenas prompts do usuario
grep -A 2 "USER PROMPT" .logs/rw-context-log-full.log

# Apenas injecoes do Synapse
grep -A 50 "SYNAPSE INJECTION" .logs/rw-context-log-full.log

# Apenas injecoes do code-intel
grep -A 20 "CODE-INTEL INJECTION" .logs/rw-context-log-full.log

# Apenas static context
grep -A 10 "STATIC CONTEXT" .logs/rw-context-log-full.log

# Apenas brackets
grep "bracket=" .logs/rw-context-log-full.log
```

## Formato do log

### Entry de UserPromptSubmit (cada prompt):

```
================================================================================
[2026-03-06T17:10:44.500Z] FULL CONTEXT -- UserPromptSubmit
================================================================================

[USER PROMPT]
crie o componente de login

[SESSION] id=abc123 | bracket=FRESH

[SYNAPSE INJECTION] (additionalContext)
<synapse-rules>
  [CONTEXT BRACKET]
  CONTEXT BRACKET: [FRESH] (99.1% remaining)
  [FRESH] CONTEXT RULES:
    1. Use ES2022 syntax with CommonJS modules...
  ...
  [CONSTITUTION] (NON-NEGOTIABLE)
    CLI First (NON-NEGOTIABLE)
    ...
</synapse-rules>

[STATIC CONTEXT] (loaded by Claude Code)
  .claude/CLAUDE.md                            (14.2 KB)
  .claude/rules/agent-authority.md             (3.1 KB)
  .claude/rules/agent-handoff.md               (2.9 KB)
  .claude/rules/mcp-usage.md                   (4.0 KB)
  .claude/rules/tool-examples.md               (1.5 KB)
  .claude/rules/workflow-execution.md          (5.2 KB)
```

### Entry de PreToolUse (cada Write/Edit):

```
--------------------------------------------------------------------------------
[2026-03-06T17:10:45.123Z] PreToolUse -- Edit
--------------------------------------------------------------------------------
[TOOL] Edit -> src/components/Login.tsx

[CODE-INTEL INJECTION] (additionalContext)
<code-intel-context>
  <target-file>src/components/Login.tsx</target-file>
  <existing-entity>
    <path>src/components/Login.tsx</path>
    <purpose>Login form component</purpose>
  </existing-entity>
  <referenced-by count="1">
    <ref file="src/App.tsx" context="Imported in main app" />
  </referenced-by>
</code-intel-context>
```

## Leitura cronologica

O log e cronologico. Uma sequencia tipica:

```
[17:10:44] FULL CONTEXT -- UserPromptSubmit   <-- usuario digitou
[17:10:45] PreToolUse -- Edit                  <-- agente editou arquivo 1
[17:10:46] PreToolUse -- Write                 <-- agente criou arquivo 2
[17:10:50] FULL CONTEXT -- UserPromptSubmit   <-- usuario digitou novamente
[17:10:51] PreToolUse -- Edit                  <-- agente editou arquivo 3
```

## Comportamento

- **Opt-in:** so grava quando `RW_CONTEXT_LOG_FULL=1`
- **Fire-and-forget:** nunca bloqueia a execucao dos hooks
- **Auto-create:** cria `.logs/` com `.gitignore` se nao existir
- **Append-only:** nunca sobrescreve, sempre adiciona
- **Pesado:** gera ~5-10KB por prompt (XML completo + static listing) — use para debug
- **Unificado:** ambos os hooks escrevem no mesmo arquivo `.logs/rw-context-log-full.log`

## Env vars — resumo

| Env var | O que ativa | Pode desativar com full? |
|---------|-------------|--------------------------|
| `RW_HOOKS_LOG=1` | Log operacional (hook-ops) | Sim |
| `RW_SYNAPSE_TRACE=1` | Trace do Synapse (prompt+XML) | Sim |
| `RW_INTEL_CONTEXT_LOG=1` | Log do code-intel (XML) | Sim |
| **`RW_CONTEXT_LOG_FULL=1`** | **Tudo unificado** | **N/A — e o master** |

---

*RIAWORKS — 2026-03-06*
