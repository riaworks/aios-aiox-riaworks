# rwSynapseTrace — Trace Detalhado do SYNAPSE

**Funcao:** `rwSynapseTrace(cwd, { prompt, sessionId, bracket, xml })`
**Env var:** `RW_SYNAPSE_TRACE=1`
**Arquivo de log:** `.logs/rw-synapse-trace.log`
**Definida em:** `.claude/hooks/synapse-engine.cjs`

---

## O que e

Trace detalhado que registra **tudo que o SYNAPSE injeta** no Claude Code a cada prompt. Responde a pergunta: "que regras exatamente foram injetadas?"

Para status operacional (hook funcionou/falhou), use `rwHooksLog` (ver `rw-hooks-log.md`).

## Por que existe

O Claude Code recebe multiplas camadas de contexto, mas apenas o SYNAPSE e invisivel — as regras sao injetadas via `additionalContext` e nao aparecem em nenhum arquivo visivel. Sem trace, e impossivel saber:

- Que prompt o usuario digitou
- Que bracket o SYNAPSE detectou (FRESH, AGENT, MID, etc.)
- Quais regras foram injetadas (Constitution, coding standards, dominio)
- Se alguma regra esta faltando ou duplicada

## Camadas de contexto do Claude Code

| Camada | Origem | Visivel em arquivo? |
|--------|--------|---------------------|
| Mensagem do usuario | Digitada | Nao |
| **SYNAPSE rules** | `.synapse/` domains + engine | **Nao — precisa de trace** |
| CLAUDE.md | `.claude/CLAUDE.md` | Sim |
| Rules contextuais | `.claude/rules/*.md` | Sim |
| MEMORY.md | Auto-memory | Sim |
| Skill prompt | Skill ativo | Nao |

O trace captura **apenas a camada SYNAPSE** — a unica que e completamente invisivel.

## O que causa falta de SYNAPSE

Quando o trace mostra `(empty)` no XML output, as causas possiveis sao:

| Causa | Diagnostico no trace |
|-------|---------------------|
| Hook nao executou | Nenhuma entry no log |
| JSON parse falhou (Windows backslash) | Entry com `(empty)` em todos os campos |
| `.synapse/` nao existe | Nenhuma entry (hook sai antes do trace) |
| Engine nao encontrou regras | BRACKET mostra, XML vazio |
| Dominio nao configurado | XML presente mas sem regras do dominio esperado |

## Como ativar

Adicione `RW_SYNAPSE_TRACE=1` antes do comando do hook em `.claude/settings.local.json`:

```json
"UserPromptSubmit": [{
  "hooks": [{
    "type": "command",
    "command": "RW_SYNAPSE_TRACE=1 node .claude/hooks/synapse-engine.cjs"
  }]
}]
```

**Ativar ambos os logs:**
```json
"command": "RW_HOOKS_LOG=1 RW_SYNAPSE_TRACE=1 node .claude/hooks/synapse-engine.cjs"
```

## Como desativar

Remova `RW_SYNAPSE_TRACE=1` do comando:

```json
"command": "node .claude/hooks/synapse-engine.cjs"
```

## Como visualizar

```bash
# Log completo
cat .logs/rw-synapse-trace.log

# Tempo real
tail -f .logs/rw-synapse-trace.log

# Apenas prompts do usuario
grep -A 5 "USER PROMPT" .logs/rw-synapse-trace.log

# Apenas XML injetado
grep -A 50 "SYNAPSE OUTPUT" .logs/rw-synapse-trace.log

# Apenas brackets
grep "BRACKET:" .logs/rw-synapse-trace.log
```

## Formato do log

4 blocos por prompt:

```
================================================================================
[2026-03-06T16:20:07.433Z] USER PROMPT
================================================================================
teste de nomenclatura rw

[2026-03-06T16:20:07.433Z] SESSION ID: rw-naming-test
[2026-03-06T16:20:07.433Z] BRACKET: FRESH

================================================================================
[2026-03-06T16:20:07.433Z] SYNAPSE OUTPUT (injected as additionalContext)
================================================================================
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
```

## Comportamento

- **Opt-in:** so grava quando `RW_SYNAPSE_TRACE=1`
- **Fire-and-forget:** nunca bloqueia a execucao do hook
- **Auto-create:** cria `.logs/` com `.gitignore` se nao existir
- **Append-only:** nunca sobrescreve, sempre adiciona
- **Pesado:** gera ~4KB por prompt (XML completo) — use apenas para debug

---

*RIAWORKS — 2026-03-06*
