# Fix: Claude Code Hooks — Bugs Conhecidos

**Data:** 2026-03-07 (atualizado)
**Plataforma:** Windows 11, Claude Code v2.1.63+, Node 22.x
**Contexto:** Claude Code executa hooks via stdin/stdout JSON protocol. Bugs nesse fluxo causam perda de injecao de regras SYNAPSE.

> **O que e o SYNAPSE, como instalar e configurar:** veja `01-fix-hook-synapse.md`

---

## Bugs Corrigidos

### Bug 1: Hook registration errada no settings.json

**Sintoma:** "UserPromptSubmit hook error" em todo prompt.

**Causa:** `precompact-session-digest.cjs` registrado como UserPromptSubmit (tipo errado — e PreCompact). Paths absolutos com `${CLAUDE_PROJECT_DIR}` que nao expande no Windows. `timeout: 10` (10ms) mata o hook antes de terminar.

**Fix:** Usar paths relativos, sem timeout, registrar cada hook no evento correto.

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
    ],
    "PreCompact": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node .claude/hooks/precompact-session-digest.cjs"
          }
        ]
      }
    ]
  }
}
```

---

### Bug 2: hookEventName ausente no output

**Sintoma:** Claude Code rejeita o output do hook silenciosamente — regras nao injetadas.

**Causa:** `buildHookOutput()` no `hook-runtime.js` nao incluia `hookEventName`. Claude Code v2.1.68+ exige esse campo.

**Fix:** Adicionar `hookEventName: 'UserPromptSubmit'` no JSON de output.

```javascript
function buildHookOutput(xml) {
  return {
    hookSpecificOutput: {
      hookEventName: 'UserPromptSubmit',  // REQUIRED
      additionalContext: xml || '',
    },
  };
}
```

---

### Bug 3: stdout cortado no Windows (process.exit)

**Sintoma:** Hook funciona intermitentemente — as vezes injeta regras, as vezes nao.

**Causa:** `process.exit(0)` mata o pipe stdout antes do Claude Code ler a resposta no Windows.

**Fix:** Remover `process.exit()`. Deixar Node encerrar naturalmente apos o stdout flush.

```javascript
// ERRADO — mata pipe no Windows
main().then(() => process.exit(0));

// CORRETO — Node encerra sozinho
main().then(() => {}).catch(() => {});
```

---

### Bug 4: Sessions nunca persistidas

**Sintoma:** `prompt_count` sempre 0, bracket transitions nao funcionam.

**Causa:** `createSession()` nunca chamada — `loadSession()` retorna null no primeiro prompt e o codigo usa fallback `{ prompt_count: 0 }` sem persistir.

**Fix:** Chamar `createSession()` quando `loadSession()` retorna null.

```javascript
let session = loadSession(sessionId, sessionsDir);
if (!session && sessionId) {
  session = createSession(sessionId, cwd, sessionsDir);
}
```

---

### Bug 5: precompact runner not found

**Sintoma:** PreCompact hook falha silenciosamente.

**Causa:** Path do runner errado. O hook busca em `.aios-core/hooks/` mas o runner esta em `node_modules/aios-core/.aios-core/hooks/` (ou precisa ser copiado para `.aiox-core/hooks/`).

**Fix:** Copiar o runner + pro-detector para o projeto, ou corrigir o path.

---

### Bug 6: Paths absolutos nao funcionam no Windows

**Sintoma:** `Bad escaped character in JSON` ou hook nao encontra arquivos.

**Causa:** `${CLAUDE_PROJECT_DIR}` e variavel bash, nao expande em cmd.exe. Paths absolutos com backslashes causam problemas de JSON escape.

**Fix:** Usar paths relativos: `node .claude/hooks/synapse-engine.cjs`.

---

### Bug 7: timeout 10ms mata hooks

**Sintoma:** Hook retorna erro de timeout antes de processar.

**Causa:** `timeout: 10` no settings.json e 10 milissegundos — insuficiente. O hook precisa de ~50-200ms.

**Fix:** Remover `timeout` do settings.json. Claude Code gerencia timeouts internamente.

---

### Bug 8: code-intel-pretool.cjs usa process.exit() (Windows pipe kill)

**Sintoma:** `PreToolUse:Write hook error` ao usar a ferramenta Write ou Edit.

**Causa:** Mesmo problema do Bug 3, mas no hook `code-intel-pretool.cjs`. A funcao `safeExit()` chamava `process.exit(0)` que no Windows mata o pipe stdout antes do Claude Code ler a resposta JSON.

**Fix:** Remover `safeExit()` e `process.exit()`. Usar o mesmo padrao do `synapse-engine.cjs` — deixar Node encerrar naturalmente.

```javascript
// ERRADO — mata pipe no Windows
function safeExit(code) {
  if (process.env.JEST_WORKER_ID) return;
  process.exit(code);
}
function run() {
  const timer = setTimeout(() => safeExit(0), HOOK_TIMEOUT_MS);
  timer.unref();
  main()
    .then(() => safeExit(0))
    .catch(() => { safeExit(0); });
}

// CORRETO — Node encerra sozinho
function run() {
  const timer = setTimeout(() => {}, HOOK_TIMEOUT_MS);
  timer.unref();
  main()
    .then(() => {})
    .catch(() => {});
}
```

---

### Bug 9: precompact-runner.js console.log/console.error causa hook error

**Sintoma:** `PreCompact hook error` intermitente.

**Causa:** O `precompact-runner.js` usa `console.log()` e `console.error()` para logging. No protocolo de hooks do Claude Code, qualquer saida em stderr e interpretada como erro, e saida stdout que nao e JSON valido tambem causa problemas.

Linhas problematicas:
- `console.log('[PreCompact] aiox-pro not available, skipping session digest')` → stdout
- `console.error('[PreCompact] Digest extractor not found...')` → stderr
- `console.error('[PreCompact] Digest extraction failed...')` → stderr
- `console.error('[PreCompact] Hook runner error...')` → stderr

**Fix:** Remover todas as chamadas `console.log()` e `console.error()`. Hooks devem operar silenciosamente — falhas sao graceful degradation.

```javascript
// ERRADO — stderr dispara "hook error" no Claude Code
if (!proAvailable) {
  console.log('[PreCompact] aiox-pro not available, skipping session digest');
  return;
}

// CORRETO — silent no-op
if (!proAvailable) {
  return; // Graceful degradation - no-op
}
```

---

## Verificacao

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

---

## Arquivos modificados

| Arquivo | Mudanca |
|---------|---------|
| `.claude/settings.local.json` | Paths relativos, sem timeout, hooks no evento correto |
| `.aiox-core/core/synapse/runtime/hook-runtime.js` | hookEventName, createSession, rwHooksLog, cleanOrphanTmpFiles |
| `.claude/hooks/synapse-engine.cjs` | Remover process.exit, sanitizeJsonString, rwSynapseTrace |
| `.claude/hooks/precompact-session-digest.cjs` | Path do runner corrigido |
| `.claude/hooks/code-intel-pretool.cjs` | Path .aios-core → .aiox-core, remover process.exit() (Bug 8) |
| `.aiox-core/hooks/unified/runners/precompact-runner.js` | Remover console.log/console.error (Bug 9) |
| `.logs/` | Diretorio criado com .gitignore |

---

*Documentado em 2026-03-07 — RIAWORKS*
