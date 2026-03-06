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

---

## Self-Service: Prompt para o Claude Aplicar Todos os Fixes e Logging

Copie o prompt abaixo e cole no Claude Code para que ele aplique todos os 8 bug fixes e os dois sistemas de logging no seu projeto AIOS/AIOX. O prompt inclui verificacao de integridade em cada etapa.

> **Pre-requisitos:** Seu projeto precisa ter a estrutura de hooks AIOX (`.claude/hooks/`, `.aiox-core/core/synapse/`, `.claude/settings.local.json`).

### Prompt Completo de Aplicacao

````
Preciso que voce aplique todos os fixes e extensoes de logging RIAWORKS neste projeto AIOS/AIOX.
Leia a documentacao abaixo, verifique cada arquivo alvo antes de editar, e aplique cada fix.

## FASE 1 — VERIFICACAO DE INTEGRIDADE (somente leitura, NAO edite ainda)

Leia esses arquivos e confirme que existem. Reporte o estado atual:

1. `.claude/settings.local.json` — verifique se hooks.UserPromptSubmit existe
2. `.claude/hooks/synapse-engine.cjs` — verifique se readStdin() e main() existem
3. `.aiox-core/core/synapse/runtime/hook-runtime.js` — verifique se buildHookOutput() existe
4. `.claude/hooks/code-intel-pretool.cjs` — verifique se referencia .aios-core ou .aiox-core
5. `.claude/hooks/precompact-session-digest.cjs` — verifique se existe

Para cada arquivo, reporte:
- EXISTE: sim/nao
- FUNCOES CHAVE ENCONTRADAS: liste-as
- JA CORRIGIDO: sim/nao (verifique se os fixes abaixo ja foram aplicados)

NAO prossiga para a Fase 2 ate reportar todos os achados e eu confirmar.

## FASE 2 — APLICAR FIXES (edite somente apos confirmacao da Fase 1)

### Fix 1: Registro de hooks no settings.local.json
Em `.claude/settings.local.json`, garanta que hooks estao registrados corretamente:
- `UserPromptSubmit` deve executar APENAS `synapse-engine.cjs`
- `PreCompact` deve executar APENAS `precompact-session-digest.cjs`
- `PreToolUse` com matcher `Write|Edit` deve executar `code-intel-pretool.cjs`
- Todos os comandos devem usar paths RELATIVOS (ex: `node .claude/hooks/synapse-engine.cjs`)
- NENHUM campo `timeout` em nenhum hook
- NENHUM `${CLAUDE_PROJECT_DIR}` em nenhum path

### Fix 2: hookEventName no buildHookOutput
Em `.aiox-core/core/synapse/runtime/hook-runtime.js`, encontre `buildHookOutput()`.
Adicione `hookEventName: 'UserPromptSubmit'` dentro de `hookSpecificOutput` se ausente:
```javascript
hookSpecificOutput: {
  hookEventName: 'UserPromptSubmit',
  additionalContext: xml || '',
},
```

### Fix 3: Remover process.exit() no synapse-engine.cjs
Em `.claude/hooks/synapse-engine.cjs`, encontre a chamada main() no final do arquivo.
Substitua:
```javascript
main().then(() => process.exit(0));
```
Por:
```javascript
main().then(() => {}).catch(() => {});
```

### Fix 4: Fallback createSession() no hook-runtime.js
Em `.aiox-core/core/synapse/runtime/hook-runtime.js`, encontre onde loadSession() e chamada.
Garanta que existe fallback para createSession() quando loadSession retorna null:
```javascript
let session = loadSession(sessionId, sessionsDir);
if (!session && sessionId) {
  session = createSession(sessionId, cwd, sessionsDir);
}
```

### Fix 5: Corrigir path .aios-core no code-intel-pretool.cjs
Em `.claude/hooks/code-intel-pretool.cjs`, substitua qualquer referencia a `.aios-core` por `.aiox-core`.

### Fix 6: Remover timeout do settings
Em `.claude/settings.local.json`, remova qualquer campo `"timeout": 10` ou similar das definicoes de hooks.

### Fix 7: Verificar runner do PreCompact
Verifique se `.aiox-core/hooks/unified/runners/precompact-runner.js` existe.
Se nao, verifique se `bin/utils/pro-detector.js` existe.
Reporte o que encontrou — NAO crie arquivos que nao existem no source.

### Fix 8: sanitizeJsonString() para escape JSON no Windows
Em `.claude/hooks/synapse-engine.cjs`, adicione esta funcao se nao existir:
```javascript
function sanitizeJsonString(raw) {
  return raw.replace(/\\(?!["\\/bfnrtu])/g, '\\\\');
}
```
Depois em `readStdin()`, envolva JSON.parse em try-catch com fallback:
```javascript
try {
  return JSON.parse(data);
} catch (e) {
  try {
    return JSON.parse(sanitizeJsonString(data));
  } catch (e2) {
    throw e; // joga o erro original
  }
}
```

## FASE 3 — APLICAR EXTENSOES DE LOGGING

### rwHooksLog() no hook-runtime.js
Em `.aiox-core/core/synapse/runtime/hook-runtime.js`, adicione esta funcao se nao existir:
```javascript
function rwHooksLog(cwd, level, message) {
  if (process.env.RW_HOOKS_LOG !== '1') return;
  try {
    const fs = require('fs');
    const path = require('path');
    const logsDir = path.join(cwd, '.logs');
    if (!fs.existsSync(logsDir)) {
      fs.mkdirSync(logsDir, { recursive: true });
      fs.writeFileSync(path.join(logsDir, '.gitignore'), '*\n');
    }
    const timestamp = new Date().toISOString();
    const line = `[${timestamp}] [${level}] ${message}\n`;
    fs.appendFileSync(path.join(logsDir, 'hook-ops.log'), line);
  } catch (_) { /* fire-and-forget */ }
}
```
Chame nos pontos chave: apos session criada, apos runtime resolvido, em erros.

### rwSynapseTrace() no synapse-engine.cjs
Em `.claude/hooks/synapse-engine.cjs`, adicione esta funcao se nao existir:
```javascript
function rwSynapseTrace(cwd, { prompt, sessionId, bracket, xml }) {
  if (process.env.RW_SYNAPSE_TRACE !== '1') return;
  try {
    const fs = require('fs');
    const path = require('path');
    const logsDir = path.join(cwd, '.logs');
    if (!fs.existsSync(logsDir)) {
      fs.mkdirSync(logsDir, { recursive: true });
      fs.writeFileSync(path.join(logsDir, '.gitignore'), '*\n');
    }
    const timestamp = new Date().toISOString();
    const sep = '='.repeat(80);
    const entry = [
      sep,
      `[${timestamp}] USER PROMPT`,
      sep,
      prompt || '(empty)',
      '',
      `[${timestamp}] SESSION ID: ${sessionId || '(none)'}`,
      `[${timestamp}] BRACKET: ${bracket || '(unknown)'}`,
      '',
      sep,
      `[${timestamp}] SYNAPSE OUTPUT (injected as additionalContext)`,
      sep,
      xml || '(empty)',
      '',
    ].join('\n');
    fs.appendFileSync(path.join(logsDir, 'synapse-trace.log'), entry);
  } catch (_) { /* fire-and-forget */ }
}
```
Chame apos o SYNAPSE engine gerar o XML de output.

## FASE 4 — VALIDACAO

Apos todas as edicoes, execute este comando de verificacao:
```bash
echo '{"prompt":"test","session_id":"verify","cwd":"'$(pwd)'"}' \
  | node .claude/hooks/synapse-engine.cjs 2>/dev/null \
  | node -e "let d='';process.stdin.on('data',c=>d+=c);process.stdin.on('end',()=>{try{const j=JSON.parse(d);console.log('hookEventName:',j.hookSpecificOutput?.hookEventName);console.log('rules:',j.hookSpecificOutput?.additionalContext?.includes('CONSTITUTION')?'YES':'NO');console.log('STATUS: OK');}catch(e){console.log('STATUS: FAIL',e.message);}})"
```

Saida esperada:
```
hookEventName: UserPromptSubmit
rules: YES
STATUS: OK
```

Reporte o resultado. Se FAIL, diagnostique usando a mensagem de erro e corrija antes de finalizar.

## REGRAS
- NAO pule a Fase 1. Verificacao de integridade e obrigatoria.
- NAO crie arquivos que nao existem — apenas modifique os existentes.
- Se um fix JA FOI APLICADO, pule e reporte "ja corrigido".
- Se um arquivo alvo NAO EXISTE ou sua estrutura mudou, reporte a discrepancia e pergunte antes de prosseguir.
- Apos a Fase 4, crie o diretorio `.logs/` com um `.gitignore` contendo `*` se nao existir.
````

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
