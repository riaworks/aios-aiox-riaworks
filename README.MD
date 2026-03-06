# AIOS-AIOX-RIAWORKS

Fixes e extensoes RIAWORKS para o sistema de hooks do [Synkra AIOS](https://github.com/synkra-ai/aios-core) / AIOX, focados em contexto no Claude Code.

## Contexto

O AIOS usa hooks do Claude Code para injetar regras SYNAPSE (coding standards, constitution, dominio) a cada prompt. No repositorio original, esses hooks tem bugs documentados que causam perda silenciosa de contexto — o Claude Code opera sem regras de projeto sem nenhum aviso.

Este repositorio contem os fixes aplicados e dois sistemas de logging criados pela RIAWORKS para diagnosticar e rastrear a injecao de contexto.

## Arquivos

### Fixes

| Arquivo | Descricao |
|---------|-----------|
| `fix-hooks-bugs.md` | 7 bugs corrigidos nos hooks do Claude Code: registration errada no settings.json, `hookEventName` ausente, `process.exit()` matando stdout no Windows, sessions nao persistidas, paths absolutos incompativeis, timeout de 10ms, e runner do PreCompact nao encontrado. |
| `fix-windows-json-escape.md` | Fix para bug intermitente onde o Claude Code envia paths Windows sem escapar backslashes no JSON (`C:\dir` em vez de `C:\\dir`), causando falha no `JSON.parse()` e perda de regras SYNAPSE naquele prompt. |

### Logging (extensoes RIAWORKS)

| Arquivo | Descricao |
|---------|-----------|
| `rw-hooks-log.md` | Documentacao do `rwHooksLog()` — log operacional leve que registra status de execucao dos hooks (session criada, runtime resolvido, erros). Ativado via `RW_HOOKS_LOG=1`. Grava em `.logs/hook-ops.log`. |
| `rw-synapse-trace.md` | Documentacao do `rwSynapseTrace()` — trace detalhado que registra o prompt do usuario, session ID, bracket e o XML completo injetado como `additionalContext`. Ativado via `RW_SYNAPSE_TRACE=1`. Grava em `.logs/synapse-trace.log`. |

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
