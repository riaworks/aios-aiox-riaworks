# SYNAPSE — Setup e Instalacao

**Data:** 2026-03-06
**Plataforma:** Windows 11, Claude Code v2.1.63+, Node 22.x

---

## O que e o SYNAPSE

O SYNAPSE e o motor de contexto do AIOX. Ele injeta regras (coding standards, constitution, dominio) no Claude Code a cada prompt via hook `UserPromptSubmit`.

**Fluxo normal:**
```
Usuario digita prompt
       |
       v
Claude Code dispara UserPromptSubmit hook
       |
       v
synapse-engine.cjs le stdin JSON, chama SynapseEngine
       |
       v
SynapseEngine gera <synapse-rules> XML
       |
       v
synapse-engine.cjs escreve JSON no stdout
       |
       v
Claude Code injeta XML como additionalContext
```

**Quando o hook falha**, o Claude Code nao recebe as regras SYNAPSE e opera sem contexto de projeto — ignora conventions, constitution, dominio, etc.

---

## Dependencias

O SYNAPSE precisa de dois diretorios para funcionar:

| Diretorio | Conteudo | Obrigatorio |
|-----------|----------|-------------|
| `.synapse/` | Dominos de contexto, sessions, cache, agents, constitution, workflows | SIM |
| `.aiox-core/core/synapse/` | Engine, runtime, session manager | SIM |

Ambos vem do repositorio oficial do AIOX.

---

## Instalacao do .synapse/

Se o diretorio `.synapse/` nao existir no projeto, o hook sai silenciosamente e nenhuma regra e injetada.

### Obter do repositorio oficial

**Repositorio:** https://github.com/SynkraAI/aiox-core

```bash
# Clonar o repositorio oficial
git clone https://github.com/SynkraAI/aiox-core.git /tmp/aiox-core

# Copiar .synapse/ para o projeto
cp -r /tmp/aiox-core/.synapse/ <MEU-PROJETO>/.synapse/

# Limpar
rm -rf /tmp/aiox-core
```

### Alternativa: sparse checkout (apenas .synapse/)

```bash
cd <MEU-PROJETO>
git clone --depth 1 --filter=blob:none --sparse \
  https://github.com/SynkraAI/aiox-core.git /tmp/aiox-sparse
cd /tmp/aiox-sparse
git sparse-checkout set .synapse
cp -r .synapse/ <MEU-PROJETO>/.synapse/
rm -rf /tmp/aiox-sparse
```

### Estrutura esperada

```
.synapse/
├── .gitignore
├── agent-aiox-master
├── agent-analyst
├── agent-architect
├── agent-data-engineer
├── agent-dev
├── agent-devops
├── agent-pm
├── agent-po
├── agent-qa
├── agent-sm
├── agent-squad-creator
├── agent-ux
├── cache/
├── commands
├── constitution
├── context
├── global
├── manifest
├── metrics/
├── sessions/
├── workflow-arch-review
├── workflow-epic-create
└── workflow-story-dev
```

---

## Verificacao

### 1. Verificar se .synapse/ existe

```bash
ls .synapse/manifest && echo "OK" || echo "FALTANDO"
```

### 2. Verificar se o hook funciona

```bash
echo '{"prompt":"test","session_id":"verify","cwd":"DIR-MEU-PROJETO"}' \
  | node .claude/hooks/synapse-engine.cjs 2>/dev/null \
  | node -e "let d='';process.stdin.on('data',c=>d+=c);process.stdin.on('end',()=>{try{const j=JSON.parse(d);console.log('hookEventName:',j.hookSpecificOutput?.hookEventName);console.log('rules:',j.hookSpecificOutput?.additionalContext?.includes('CONSTITUTION')?'YES':'NO');console.log('STATUS: OK');}catch(e){console.log('STATUS: FAIL',e.message);}})"
```

Esperado:
```
hookEventName: UserPromptSubmit
rules: YES
STATUS: OK
```

### 3. Diagnostico de falhas

| Causa | Diagnostico |
|-------|-------------|
| `.synapse/` nao existe | Hook sai silenciosamente, nenhuma entry no log |
| `.aiox-core/core/synapse/` falta | Hook crasha com `Cannot find module` |
| Engine nao encontrou regras | Bracket aparece no trace, XML vazio |
| Dominio nao configurado | XML presente mas sem regras do dominio esperado |

---

## Configuracao dos hooks

Os hooks devem estar registrados em `.claude/settings.local.json`:

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node .claude/hooks/synapse-engine.cjs"
          }
        ]
      }
    ]
  }
}
```

**Com logs ativados (debug):**

```json
"command": "RW_HOOKS_LOG=1 RW_SYNAPSE_TRACE=1 node .claude/hooks/synapse-engine.cjs"
```

Para bugs conhecidos nos hooks, veja `fix-hooks-bugs.md`.

---

## Arquivos envolvidos

| Arquivo | Papel |
|---------|-------|
| `.synapse/` | Dominios de contexto, constitution, agents, workflows |
| `.aiox-core/core/synapse/engine.js` | SynapseEngine — processa dominios e gera XML |
| `.aiox-core/core/synapse/runtime/hook-runtime.js` | Runtime: resolve session, build output |
| `.aiox-core/core/synapse/session/session-manager.js` | Gerencia sessions e bracket transitions |
| `.claude/hooks/synapse-engine.cjs` | Entry point do hook — le stdin, chama engine, escreve stdout |
| `.claude/settings.local.json` | Registro dos hooks no Claude Code |

---

*Documentado em 2026-03-06 — RIAWORKS*
