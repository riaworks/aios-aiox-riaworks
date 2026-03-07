# AIOS-AIOX-RIAWORKS

> Fixes e extensoes de logging RIAWORKS para o sistema de hooks do [Synkra AIOX](https://github.com/SynkraAI/aiox-core), focados em injecao de contexto no Claude Code.

**[Read in English](./README.md)**

## Indice

- [Visao Geral](#visao-geral)
- [Pre-requisitos](#pre-requisitos)
- [Inicio Rapido](#inicio-rapido)
- [Arquivos](#arquivos)
- [Referencia de Logs (rw-)](#referencia-de-logs-rw-)
  - [rw-hooks-log](#1-rw-hooks-log--status-operacional)
  - [rw-synapse-trace](#2-rw-synapse-trace--trace-xml-do-synapse)
  - [rw-intel-context-log](#3-rw-intel-context-log--injecao-code-intel)
  - [rw-context-log-full](#4-rw-context-log-full--log-unificado-completo)
  - [Individual vs Full](#individual-vs-full)
- [Ativacao](#ativacao)
- [Prompt: Aplicar Todos os Fixes](#prompt-aplicar-todos-os-fixes)
- [Prompt: Ativar/Desativar Logging](#prompt-ativardesativar-logging)
- [Repositorio Original](#repositorio-original)
- [Licenca](#licenca)

---

## Visao Geral

O AIOX usa hooks do Claude Code para injetar regras SYNAPSE (coding standards, constitution, dominio) a cada prompt. No repositorio original, esses hooks tem bugs documentados que causam **perda silenciosa de contexto** — o Claude Code opera sem regras de projeto sem nenhum aviso.

Este repositorio contem:
- **10 bug fixes** para o sistema de hooks
- **4 extensoes de logging** (`rw-*`) para diagnosticar e rastrear a injecao de contexto

## Pre-requisitos

### Diretorio de execucao

O prompt **deve ser executado na raiz do projeto** — o diretorio que contem `.aiox-core/`. E o mesmo diretorio onde o Claude Code esta executando o seu projeto.

```
seu-projeto/               <-- execute aqui
├── .aiox-core/
├── .claude/
├── .synapse/
├── aios-aiox-riaworks/    <-- este repo deve estar aqui
└── ...
```

### Este repositorio

A pasta `aios-aiox-riaworks/` deve existir na raiz do projeto. Se nao existir, o prompt de self-service vai perguntar se pode clonar:

```bash
git clone https://github.com/riaworks/aios-aiox-riaworks.git
```

> **Importante:** O Claude Code pode ler toda a documentacao de fixes e trechos de codigo diretamente dos arquivos em `aios-aiox-riaworks/`. Os prompts referenciam esses arquivos — nao e preciso copiar codigo manualmente.

## Inicio Rapido

1. Certifique-se de estar na raiz do projeto (onde `.aiox-core/` existe)
2. Certifique-se de que `aios-aiox-riaworks/` existe (ou deixe o prompt clonar)
3. Copie o [prompt de aplicar fixes](#prompt-aplicar-todos-os-fixes) no Claude Code
4. Apos os fixes aplicados, opcionalmente [ative o logging](#ativacao)

## Arquivos

### Setup e Instalacao

| Arquivo | Descricao |
|---------|-----------|
| [`01-fix-hook-synapse.md`](./01-fix-hook-synapse.md) | Guia de setup e instalacao do SYNAPSE. Como obter `.synapse/` do [repositorio oficial do AIOX](https://github.com/SynkraAI/aiox-core), estrutura esperada, configuracao dos hooks e diagnostico. |

### Fixes

| Arquivo | Descricao |
|---------|-----------|
| [`02-fix-hooks-bugs.md`](./02-fix-hooks-bugs.md) | 9 bugs corrigidos: registration errada, `hookEventName` ausente, `process.exit()` matando stdout no Windows, sessions nao persistidas, paths absolutos, timeout de 10ms, runner do PreCompact, `process.exit()` no code-intel-pretool matando pipe, `console.log/error` no PreCompact runner causando hook errors. |
| [`03-fix-windows-json-escape.md`](./03-fix-windows-json-escape.md) | Fix para falha intermitente de JSON parse no Windows quando o Claude Code envia backslashes sem escape (`C:\dir` em vez de `C:\\dir`). |

### Logging (extensoes rw-)

| Arquivo | Descricao |
|---------|-----------|
| [`rw-hooks-log.md`](./rw-hooks-log.md) | `rwHooksLog()` — status operacional dos hooks |
| [`rw-synapse-trace.md`](./rw-synapse-trace.md) | `rwSynapseTrace()` — trace XML completo do SYNAPSE |
| [`rw-intel-context-log.md`](./rw-intel-context-log.md) | `rwIntelContextLog()` — log de injecao code-intel |
| [`rw-context-log-full.md`](./rw-context-log-full.md) | `rwContextLogFull()` — log unificado completo |

---

## Referencia de Logs (rw-)

Todas as extensoes de logging RIAWORKS usam o prefixo `rw-`. Sao **4 logs**: 3 individuais + 1 unificado.

### Referencia Rapida

| # | Log | Env Var | Arquivo de Log | Hook | Peso |
|---|-----|---------|----------------|------|------|
| 1 | **rw-hooks-log** | `RW_HOOKS_LOG=1` | `.logs/rw-hooks-log.log` | UserPromptSubmit | Leve (~100B/prompt) |
| 2 | **rw-synapse-trace** | `RW_SYNAPSE_TRACE=1` | `.logs/rw-synapse-trace.log` | UserPromptSubmit | Pesado (~4KB/prompt) |
| 3 | **rw-intel-context-log** | `RW_INTEL_CONTEXT_LOG=1` | `.logs/rw-intel-context-log.log` | PreToolUse (Write/Edit) | Condicional |
| 4 | **rw-context-log-full** | `RW_CONTEXT_LOG_FULL=1` | `.logs/rw-context-log-full.log` | Ambos | Pesado (~5-10KB/prompt) |

---

### 1. rw-hooks-log — Status Operacional

**Objetivo:** Responde "o hook esta funcionando ou falhando?"

Registra eventos do ciclo de vida: session criada, runtime resolvido, erros. **Nao** registra conteudo (prompts, XML).

**Quando usar:** Primeira linha de diagnostico. Se os hooks estao falhando silenciosamente, este log mostra exatamente onde o fluxo quebrou.

| Entrada no Log | Significado | SYNAPSE Injetado? |
|-----------------|-------------|-------------------|
| `Session created: {id}` | Primeira execucao da sessao | Sim (proximo log confirma) |
| `Runtime resolved` | Hook executou com sucesso | Sim |
| `Hook output: N rules` | Regras geradas e escritas no stdout | Sim |
| `No .synapse/ directory` | Diretorio .synapse/ nao existe | Nao |
| `Failed to resolve runtime` | Erro no engine/session | Nao |
| `Hook crashed` | Erro fatal | Nao |

**Ativar individualmente:**
```json
"command": "RW_HOOKS_LOG=1 node .claude/hooks/synapse-engine.cjs"
```

**Acompanhar:**
```bash
tail -f .logs/rw-hooks-log.log
```

Documentacao completa: [`rw-hooks-log.md`](./rw-hooks-log.md)

---

### 2. rw-synapse-trace — Trace XML do SYNAPSE

**Objetivo:** Responde "que regras exatamente foram injetadas?"

Registra o XML SYNAPSE completo injetado como `additionalContext` a cada prompt, incluindo prompt do usuario, session ID e bracket.

**Quando usar:** Quando precisa ver as regras exatas que o Claude recebeu. Util para detectar regras faltando, bracket errado ou injecao vazia.

**Ativar individualmente:**
```json
"command": "RW_SYNAPSE_TRACE=1 node .claude/hooks/synapse-engine.cjs"
```

**Acompanhar:**
```bash
tail -f .logs/rw-synapse-trace.log
```

Documentacao completa: [`rw-synapse-trace.md`](./rw-synapse-trace.md)

---

### 3. rw-intel-context-log — Injecao Code-Intel

**Objetivo:** Responde "que contexto de codigo foi injetado quando o agente editou este arquivo?"

Registra o XML `<code-intel-context>` injetado em operacoes **Write/Edit** apenas. Mostra entidades, referencias e dependencias do arquivo sendo editado.

**Quando usar:** Quando suspeita que o code-intel nao esta fornecendo contexto para arquivos editados, ou quer verificar que dados de entidade o Claude ve.

> **Nota:** Este log roda em um hook diferente (`PreToolUse`) dos outros dois. So dispara quando o agente escreve ou edita um arquivo, nao a cada prompt.

**Ativar individualmente:**
```json
"command": "RW_INTEL_CONTEXT_LOG=1 node .claude/hooks/code-intel-pretool.cjs"
```

> Isso vai na secao do hook `PreToolUse`, **nao** no `UserPromptSubmit`.

**Acompanhar:**
```bash
tail -f .logs/rw-intel-context-log.log
```

Documentacao completa: [`rw-intel-context-log.md`](./rw-intel-context-log.md)

---

### 4. rw-context-log-full — Log Unificado Completo

**Objetivo:** Responde "qual o contexto completo que o Claude esta recebendo?"

Captura **tudo** em um unico arquivo de log cronologico:

| Secao | Fonte | Quando |
|-------|-------|--------|
| `[USER PROMPT]` | Texto do usuario | Cada prompt |
| `[SESSION]` | Session ID + bracket | Cada prompt |
| `[SYNAPSE INJECTION]` | XML `<synapse-rules>` completo | Cada prompt |
| `[STATIC CONTEXT]` | Listagem CLAUDE.md, rules/*.md, MEMORY.md | Cada prompt |
| `[CODE-INTEL INJECTION]` | XML `<code-intel-context>` | Cada Write/Edit |

**Quando usar:** Quando quer ver tudo em um lugar so, em vez de checar multiplos arquivos de log.

**Ativar:**

> **Importante:** `RW_CONTEXT_LOG_FULL=1` deve ser adicionado em **ambos** os hooks:

```json
{
  "hooks": {
    "UserPromptSubmit": [{
      "hooks": [{
        "type": "command",
        "command": "RW_CONTEXT_LOG_FULL=1 node .claude/hooks/synapse-engine.cjs"
      }]
    }],
    "PreToolUse": [{
      "hooks": [{
        "type": "command",
        "command": "RW_CONTEXT_LOG_FULL=1 node .claude/hooks/code-intel-pretool.cjs"
      }],
      "matcher": "Write|Edit"
    }]
  }
}
```

**Acompanhar:**
```bash
tail -f .logs/rw-context-log-full.log
```

Documentacao completa: [`rw-context-log-full.md`](./rw-context-log-full.md)

---

### Individual vs Full

Voce pode usar qualquer log individualmente, ou ativar o log unificado completo. Aqui esta a relacao:

| Log Individual | Incluido no Full? | Pode ser usado sozinho? |
|----------------|--------------------|-----------------------|
| `RW_HOOKS_LOG` | **Sim** | Sim |
| `RW_SYNAPSE_TRACE` | **Sim** | Sim |
| `RW_INTEL_CONTEXT_LOG` | **Sim** | Sim |
| `RW_CONTEXT_LOG_FULL` | N/A — **e o unificado master** | Sim |

**Pontos-chave:**

- **Full substitui os 3 individuais.** Quando `RW_CONTEXT_LOG_FULL=1` esta ativo, voce **nao** precisa de `RW_HOOKS_LOG`, `RW_SYNAPSE_TRACE` ou `RW_INTEL_CONTEXT_LOG`.
- **Individuais podem ser combinados.** Voce pode ativar qualquer combinacao dos 3 individuais. Exemplo: `RW_HOOKS_LOG=1 RW_SYNAPSE_TRACE=1` para operacional + trace XML sem code-intel.
- **Full escreve em um unico arquivo.** Os 3 individuais escrevem em arquivos separados. Full escreve tudo em `.logs/rw-context-log-full.log`.
- **Pode misturar.** Ativar `RW_CONTEXT_LOG_FULL=1` junto com um individual vai escrever em ambos (sem conflito, mas redundante).

#### Combinacoes recomendadas

| Cenario | Configuracao |
|---------|-------------|
| Verificacao rapida de saude | `RW_HOOKS_LOG=1` apenas |
| Debug de regras faltando | `RW_HOOKS_LOG=1 RW_SYNAPSE_TRACE=1` |
| Debug de code-intel apenas | `RW_INTEL_CONTEXT_LOG=1` no PreToolUse |
| Sessao de diagnostico completa | `RW_CONTEXT_LOG_FULL=1` em ambos os hooks |

---

## Ativacao

As variaveis de logging **nao** sao ativadas via `export` no terminal. Sao definidas como **env vars inline** diretamente no comando do hook em `.claude/settings.local.json`.

### Onde configurar

Arquivo: `.claude/settings.local.json` → secao `hooks`

Existem **dois hooks separados** que aceitam logging:

| Evento do Hook | Script | Aceita |
|----------------|--------|--------|
| `UserPromptSubmit` | `synapse-engine.cjs` | `RW_HOOKS_LOG`, `RW_SYNAPSE_TRACE`, `RW_CONTEXT_LOG_FULL` |
| `PreToolUse` (Write\|Edit) | `code-intel-pretool.cjs` | `RW_INTEL_CONTEXT_LOG`, `RW_CONTEXT_LOG_FULL` |

### Exemplos

**Padrao (todo logging desativado):**
```json
"command": "node .claude/hooks/synapse-engine.cjs"
"command": "node .claude/hooks/code-intel-pretool.cjs"
```

**Apenas log de hooks (leve):**
```json
"command": "RW_HOOKS_LOG=1 node .claude/hooks/synapse-engine.cjs"
```

**Log de hooks + trace SYNAPSE (recomendado para debug):**
```json
"command": "RW_HOOKS_LOG=1 RW_SYNAPSE_TRACE=1 node .claude/hooks/synapse-engine.cjs"
```

**Full unificado (ambos os hooks):**
```json
"command": "RW_CONTEXT_LOG_FULL=1 node .claude/hooks/synapse-engine.cjs"
"command": "RW_CONTEXT_LOG_FULL=1 node .claude/hooks/code-intel-pretool.cjs"
```

### Para desativar

Remova os prefixos de env var dos comandos, deixando apenas:
```json
"command": "node .claude/hooks/synapse-engine.cjs"
"command": "node .claude/hooks/code-intel-pretool.cjs"
```

### Comportamento (todos os logs)

- **Opt-in:** So grava quando a env var esta definida como `1`
- **Fire-and-forget:** Nunca bloqueia a execucao do hook
- **Auto-create:** Cria `.logs/` com `.gitignore` se nao existir
- **Append-only:** Nunca sobrescreve, sempre adiciona

---

## Prompt: Aplicar Todos os Fixes

Copie o prompt abaixo e cole no Claude Code. Ele vai ler a documentacao de fixes em `aios-aiox-riaworks/` e aplicar todos os fixes no seu projeto.

> **Pre-requisitos:** Seu projeto precisa ter a estrutura de hooks AIOX (`.claude/hooks/`, `.aiox-core/core/synapse/`, `.claude/settings.local.json`).

### Prompt de Aplicacao Self-Service

````
Preciso que voce aplique todos os fixes e extensoes de logging RIAWORKS neste projeto AIOX.

## PASSO 0 — VERIFICAR REPOSITORIO

Verifique se a pasta `aios-aiox-riaworks/` existe na raiz do projeto (mesmo nivel que `.aiox-core/`).

Se NAO existir, me pergunte:
"O repositorio aios-aiox-riaworks nao esta presente. Posso clonar de
https://github.com/riaworks/aios-aiox-riaworks.git neste diretorio?"

Aguarde minha confirmacao antes de clonar. Se ja existir, continue.

## PASSO 1 — LER TODA A DOCUMENTACAO

Leia estes arquivos do diretorio `aios-aiox-riaworks/`. Eles contem todos os detalhes dos fixes,
trechos de codigo e comportamento esperado. NAO invente — use o codigo exato desses arquivos:

1. `aios-aiox-riaworks/01-fix-hook-synapse.md` — Requisitos de setup do SYNAPSE
2. `aios-aiox-riaworks/02-fix-hooks-bugs.md` — Todos os 9 bug fixes com codigo
3. `aios-aiox-riaworks/03-fix-windows-json-escape.md` — Fix de escape JSON com codigo
4. `aios-aiox-riaworks/rw-hooks-log.md` — Funcao rwHooksLog() e uso
5. `aios-aiox-riaworks/rw-synapse-trace.md` — Funcao rwSynapseTrace() e uso
6. `aios-aiox-riaworks/rw-intel-context-log.md` — Funcao rwIntelContextLog() e uso
7. `aios-aiox-riaworks/rw-context-log-full.md` — Funcao rwContextLogFull() e uso

Apos ler, reporte um resumo do que encontrou.

## PASSO 2 — VERIFICACAO DE INTEGRIDADE (somente leitura, NAO edite ainda)

Leia e verifique estes arquivos alvo:

1. `.claude/settings.local.json` — verifique se hooks.UserPromptSubmit existe
2. `.claude/hooks/synapse-engine.cjs` — verifique se readStdin() e main() existem
3. `.aiox-core/core/synapse/runtime/hook-runtime.js` — verifique se buildHookOutput() existe
4. `.claude/hooks/code-intel-pretool.cjs` — verifique se referencia .aios-core ou .aiox-core
5. `.claude/hooks/precompact-session-digest.cjs` — verifique se existe

Para cada arquivo, reporte:
- EXISTE: sim/nao
- FUNCOES CHAVE ENCONTRADAS: liste-as
- JA CORRIGIDO: sim/nao (compare com os fixes da documentacao)

NAO prossiga para o Passo 3 ate reportar os achados e eu confirmar.

## PASSO 3 — APLICAR FIXES

Aplique cada fix descrito nos arquivos de documentacao lidos no Passo 1.
Use o codigo exato da documentacao — NAO invente ou modifique.

Para cada fix:
- Se JA APLICADO, pule e reporte "ja corrigido"
- Se arquivo alvo NAO EXISTE, reporte e pergunte antes de prosseguir

## PASSO 4 — APLICAR EXTENSOES DE LOGGING

Aplique as 4 funcoes de logging (rwHooksLog, rwSynapseTrace, rwIntelContextLog,
rwContextLogFull) conforme descrito nos respectivos arquivos de documentacao.

## PASSO 5 — VALIDACAO

Execute o comando de verificacao do `02-fix-hooks-bugs.md` e reporte o resultado.
Crie o diretorio `.logs/` com `.gitignore` contendo `*` se nao existir.

## REGRAS
- NAO pule o Passo 1. Ler a documentacao e obrigatorio.
- NAO pule o Passo 2. Verificacao de integridade e obrigatoria.
- TODO codigo vem dos arquivos de documentacao — nunca invente codigo.
- Se um arquivo alvo nao existe ou mudou de estrutura, pergunte antes de prosseguir.
````

---

## Prompt: Ativar/Desativar Logging

Apos os fixes estarem instalados, use estes prompts para ativar ou desativar o logging.

### Prompt para ATIVAR logging

```
Leia `aios-aiox-riaworks/README.pt-BR.md`, secao "Ativacao".
Depois leia `.claude/settings.local.json` e localize os comandos de hooks.

Antes de modificar, verifique:
1. `.claude/hooks/synapse-engine.cjs` existe e contem rwHooksLog/rwSynapseTrace
2. `.claude/hooks/code-intel-pretool.cjs` existe e contem rwIntelContextLog
3. Os comandos no settings.local.json apontam para os caminhos corretos

Se todas as verificacoes passarem, atualize os comandos para ativar o log unificado completo:
- UserPromptSubmit: "RW_CONTEXT_LOG_FULL=1 node .claude/hooks/synapse-engine.cjs"
- PreToolUse: "RW_CONTEXT_LOG_FULL=1 node .claude/hooks/code-intel-pretool.cjs"

Se alguma verificacao falhar, reporte o que mudou e sugira o fix correto.
```

### Prompt para DESATIVAR logging

```
Leia `.claude/settings.local.json` e localize os comandos de hooks.
Verifique que ambos os scripts de hook existem nos caminhos esperados.

Se verificado, atualize ambos os comandos removendo todas as env vars RW_:
- UserPromptSubmit: "node .claude/hooks/synapse-engine.cjs"
- PreToolUse: "node .claude/hooks/code-intel-pretool.cjs"

Se os caminhos ou estrutura mudaram, reporte antes de editar.
```

### Por que a verificacao de integridade importa

O framework AIOX evolui entre sessoes. Arquivos podem ser renomeados, metodos refatorados, ou o registro de hooks reestruturado. Os prompts acima forcam o Claude a:

1. **Ler a documentacao primeiro** — todo codigo vem dos arquivos em `aios-aiox-riaworks/`, nao da memoria
2. **Verificar existencia dos arquivos** — confirmar que scripts estao nos caminhos esperados
3. **Verificar assinaturas dos metodos** — confirmar que funcoes rw existem no source
4. **Reportar antes de agir** — explicar discrepancias em vez de aplicar edits quebrados

---

## Arquivos Modificados no Projeto

| Arquivo | Mudancas |
|---------|---------|
| `.claude/settings.local.json` | Paths relativos, sem timeout, hooks nos eventos corretos |
| `.aiox-core/core/synapse/runtime/hook-runtime.js` | `hookEventName`, `createSession()`, `rwHooksLog()`, `cleanOrphanTmpFiles()` |
| `.claude/hooks/synapse-engine.cjs` | `sanitizeJsonString()`, `rwSynapseTrace()`, `rwContextLogFull()`, remocao de `process.exit()` |
| `.claude/hooks/code-intel-pretool.cjs` | Path `.aios-core` → `.aiox-core`, `rwIntelContextLog()`, `rwContextLogFull()`, remocao de `process.exit()` (Bug 8) |
| `.aiox-core/hooks/unified/runners/precompact-runner.js` | Runner copiado do fork com paths adaptados, remocao de `console.log/error` (Bug 9) |
| `bin/utils/pro-detector.js` | Dependencia do precompact runner |
| `.logs/` | Diretorio criado com `.gitignore` |

## Nomenclatura

Todas as extensoes RIAWORKS usam prefixo `rw` para diferenciacao do codigo original do AIOX:

| Funcao | Env Var | Arquivo de Log |
|--------|---------|---------------|
| `rwHooksLog()` | `RW_HOOKS_LOG=1` | `.logs/rw-hooks-log.log` |
| `rwSynapseTrace()` | `RW_SYNAPSE_TRACE=1` | `.logs/rw-synapse-trace.log` |
| `rwIntelContextLog()` | `RW_INTEL_CONTEXT_LOG=1` | `.logs/rw-intel-context-log.log` |
| `rwContextLogFull()` | `RW_CONTEXT_LOG_FULL=1` | `.logs/rw-context-log-full.log` |

## Repositorio Original

- **AIOX Core:** [github.com/SynkraAI/aiox-core](https://github.com/SynkraAI/aiox-core)
- **Este repo:** [github.com/riaworks/aios-aiox-riaworks](https://github.com/riaworks/aios-aiox-riaworks)
- **AIOX** e o AI-Orchestrated System for Full Stack Development da Synkra AI

## Licenca

MIT License

Copyright (c) 2026 RIAWORKS

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
