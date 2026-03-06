# AIOS-AIOX-RIAWORKS

> Fixes e extensoes RIAWORKS para o sistema de hooks do [Synkra AIOS](https://github.com/synkra-ai/aios-core) / AIOX, focados em contexto no Claude Code.

**[Read in English](./README.md)**

---

## Visao Geral

O AIOS usa hooks do Claude Code para injetar regras SYNAPSE (coding standards, constitution, dominio) a cada prompt. No repositorio original, esses hooks tem bugs documentados que causam **perda silenciosa de contexto** — o Claude Code opera sem regras de projeto sem nenhum aviso.

Este repositorio contem os fixes aplicados e dois sistemas de logging criados pela RIAWORKS para diagnosticar e rastrear a injecao de contexto.

## Arquivos

### Fixes

| Arquivo | Descricao |
|---------|-----------|
| [`fix-hooks-bugs.md`](./fix-hooks-bugs.md) | 7 bugs corrigidos nos hooks do Claude Code: registration errada no settings.json, `hookEventName` ausente, `process.exit()` matando stdout no Windows, sessions nao persistidas, paths absolutos incompativeis, timeout de 10ms, e runner do PreCompact nao encontrado. |
| [`fix-windows-json-escape.md`](./fix-windows-json-escape.md) | Fix para bug intermitente onde o Claude Code envia paths Windows sem escapar backslashes no JSON (`C:\dir` em vez de `C:\\dir`), causando falha no `JSON.parse()` e perda de regras SYNAPSE naquele prompt. |

### Logging (extensoes RIAWORKS)

| Arquivo | Descricao |
|---------|-----------|
| [`rw-hooks-log.md`](./rw-hooks-log.md) | Documentacao do `rwHooksLog()` — log operacional leve que registra status de execucao dos hooks (session criada, runtime resolvido, erros). Ativado via `RW_HOOKS_LOG=1`. Grava em `.logs/hook-ops.log`. |
| [`rw-synapse-trace.md`](./rw-synapse-trace.md) | Documentacao do `rwSynapseTrace()` — trace detalhado que registra o prompt do usuario, session ID, bracket e o XML completo injetado como `additionalContext`. Ativado via `RW_SYNAPSE_TRACE=1`. Grava em `.logs/synapse-trace.log`. |

## Nomenclatura

Todas as extensoes RIAWORKS usam prefixo `rw` para diferenciacao do codigo original:

| Funcao | Env var | Arquivo de log |
|--------|---------|---------------|
| `rwHooksLog()` | `RW_HOOKS_LOG=1` | `.logs/hook-ops.log` |
| `rwSynapseTrace()` | `RW_SYNAPSE_TRACE=1` | `.logs/synapse-trace.log` |

## Arquivos modificados no projeto

| Arquivo | Mudancas |
|---------|---------|
| `.claude/settings.local.json` | Paths relativos, sem timeout, hooks registrados nos eventos corretos |
| `.aiox-core/core/synapse/runtime/hook-runtime.js` | `hookEventName`, `createSession()`, `rwHooksLog()`, `cleanOrphanTmpFiles()` |
| `.claude/hooks/synapse-engine.cjs` | `sanitizeJsonString()`, `rwSynapseTrace()`, remocao de `process.exit()` |
| `.claude/hooks/code-intel-pretool.cjs` | Path `.aios-core` corrigido para `.aiox-core` |
| `.aiox-core/hooks/unified/runners/precompact-runner.js` | Runner copiado do fork com paths adaptados |
| `bin/utils/pro-detector.js` | Dependencia do precompact runner |
| `.logs/` | Diretorio criado com `.gitignore` |

## Ativacao

As variaveis de logging **nao** sao ativadas via `export` no terminal. Elas sao definidas como **env vars inline** diretamente no comando do hook dentro de `.claude/settings.local.json`.

### Onde configurar

Arquivo: `.claude/settings.local.json` → `hooks.UserPromptSubmit[0].hooks[0].command`

### Exemplos

**Padrao (logging desativado):**
```json
"command": "node .claude/hooks/synapse-engine.cjs"
```

**Ativar apenas log de hooks:**
```json
"command": "RW_HOOKS_LOG=1 node .claude/hooks/synapse-engine.cjs"
```

**Ativar apenas trace SYNAPSE:**
```json
"command": "RW_SYNAPSE_TRACE=1 node .claude/hooks/synapse-engine.cjs"
```

**Ativar ambos (recomendado para debug):**
```json
"command": "RW_HOOKS_LOG=1 RW_SYNAPSE_TRACE=1 node .claude/hooks/synapse-engine.cjs"
```

### Saida dos logs

| Variavel | Arquivo de log | Conteudo |
|----------|---------------|----------|
| `RW_HOOKS_LOG=1` | `.logs/hook-ops.log` | Ciclo de vida do hook: session criada, runtime resolvido, erros |
| `RW_SYNAPSE_TRACE=1` | `.logs/synapse-trace.log` | XML SYNAPSE completo injetado como `additionalContext`, session ID, bracket |

```bash
# Acompanhar logs em tempo real
tail -f .logs/hook-ops.log
tail -f .logs/synapse-trace.log
```

### Para desativar

Remova os prefixos de env var do comando, deixando apenas:
```json
"command": "node .claude/hooks/synapse-engine.cjs"
```

---

## Self-Service: Prompt para o Claude ativar/desativar o logging

Como a ativacao requer editar o `settings.local.json`, voce pode pedir ao proprio Claude para fazer. Use os prompts abaixo — eles incluem **verificacao de integridade** para que o Claude confira se os caminhos e metodos ainda existem antes de alterar.

### Prompt para ATIVAR logging

```
Leia o arquivo .claude/settings.local.json e localize o comando do hook UserPromptSubmit
que executa o synapse-engine.cjs. Antes de modificar, verifique:

1. O arquivo .claude/hooks/synapse-engine.cjs existe
2. Ele contem as funcoes rwHooksLog() e rwSynapseTrace()
3. O comando atual no settings.local.json aponta para o caminho correto

Se todas as verificacoes passarem, atualize o comando para:
"command": "RW_HOOKS_LOG=1 RW_SYNAPSE_TRACE=1 node .claude/hooks/synapse-engine.cjs"

Se qualquer verificacao falhar, reporte o que mudou e sugira o fix correto em vez de
aplicar a edicao cegamente.
```

### Prompt para DESATIVAR logging

```
Leia o arquivo .claude/settings.local.json e localize o comando do hook UserPromptSubmit.
Verifique que .claude/hooks/synapse-engine.cjs existe e o caminho no comando esta correto.

Se verificado, atualize o comando para:
"command": "node .claude/hooks/synapse-engine.cjs"

Se o caminho ou estrutura mudou, reporte a discrepancia antes de editar.
```

### Por que a verificacao de integridade importa

O framework AIOX evolui entre sessoes. Arquivos podem ser renomeados, metodos refatorados, ou o registro de hooks reestruturado. Os prompts acima forcam o Claude a:

1. **Verificar existencia do arquivo** — confirmar que `synapse-engine.cjs` ainda esta no caminho esperado
2. **Verificar assinaturas dos metodos** — confirmar que `rwHooksLog()` e `rwSynapseTrace()` ainda existem no source
3. **Verificar estrutura do settings** — confirmar que o hook `UserPromptSubmit` ainda esta registrado no `settings.local.json`
4. **Reportar antes de agir** — se algo mudou, o Claude explica a discrepancia em vez de aplicar uma edicao quebrada

## Repositorio Original

- **AIOS Core:** [github.com/synkra-ai/aios-core](https://github.com/synkra-ai/aios-core)
- **AIOX** e o fork interno que estende o AIOS com orquestracao multi-agente

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
