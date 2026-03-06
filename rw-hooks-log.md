# rwHooksLog — Log Operacional de Hooks

**Funcao:** `rwHooksLog(cwd, level, message)`
**Env var:** `RW_HOOKS_LOG=1`
**Arquivo de log:** `.logs/hook-ops.log`
**Definida em:** `.aiox-core/core/synapse/runtime/hook-runtime.js`

---

## O que e

Log operacional leve que registra o **status de execucao** dos hooks do Claude Code. Responde a pergunta: "o hook esta funcionando ou falhando?"

Nao registra conteudo (prompts, XML). Para isso use `rwSynapseTrace` (ver `rw-synapse-trace.md`).

## Por que existe

O repositorio original do AIOS nao tem nenhum sistema de logging para hooks. Quando um hook falha, o Claude Code apenas mostra "hook error" sem detalhes. Sem logs, e impossivel diagnosticar:

- Se o hook esta executando
- Se a session foi criada
- Se o runtime resolveu corretamente
- Qual erro causou a falha
- Quantas regras foram injetadas

## O que causa falta de SYNAPSE

Qualquer falha no hook impede a injecao de regras. O `rwHooksLog` registra exatamente onde o fluxo quebrou:

| Log | Significado | SYNAPSE injetado? |
|-----|-------------|-------------------|
| `No .synapse/ directory` | Diretorio .synapse/ nao existe | Nao |
| `Session created: {id}` | Primeira execucao da sessao | Sim (proximo log confirma) |
| `Runtime resolved` | Hook executou com sucesso | Sim |
| `Hook output: N rules` | Regras geradas e escritas no stdout | Sim |
| `Failed to resolve runtime` | Erro ao carregar engine/session | Nao |
| `Hook crashed` | Erro fatal no hook | Nao |

## Como ativar

Adicione `RW_HOOKS_LOG=1` antes do comando do hook em `.claude/settings.local.json`:

```json
"UserPromptSubmit": [{
  "hooks": [{
    "type": "command",
    "command": "RW_HOOKS_LOG=1 node .claude/hooks/synapse-engine.cjs"
  }]
}]
```

## Como desativar

Remova `RW_HOOKS_LOG=1` do comando:

```json
"command": "node .claude/hooks/synapse-engine.cjs"
```

## Como visualizar

```bash
# Log completo
cat .logs/hook-ops.log

# Tempo real
tail -f .logs/hook-ops.log

# Apenas erros
grep ERROR .logs/hook-ops.log

# Ultimas 20 linhas
tail -20 .logs/hook-ops.log
```

## Formato do log

```
[2026-03-06T16:03:28.510Z] [INFO] Session created: e38b56d8-af23-47fb-954c
[2026-03-06T16:03:28.587Z] [INFO] Runtime resolved — session=e38b56d8, prompt_count=0, bracket=FRESH
[2026-03-06T16:03:28.592Z] [INFO] Hook output: 59 rules, bracket=FRESH, xml=3881 bytes
```

Cada prompt gera 2-3 linhas. Leve, sem impacto em performance.

## Comportamento

- **Opt-in:** so grava quando `RW_HOOKS_LOG=1`
- **Fire-and-forget:** nunca bloqueia a execucao do hook
- **Auto-create:** cria `.logs/` com `.gitignore` se nao existir
- **Append-only:** nunca sobrescreve, sempre adiciona

---

*RIAWORKS — 2026-03-06*
