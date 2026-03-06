# Fix: Windows JSON Escape — Backslash em Paths

**Data:** 2026-03-05
**Componente:** `.claude/hooks/synapse-engine.cjs`
**Severidade:** Low (intermitente, auto-recovery)

---

## Problema

```
[ERROR] Hook crashed: Bad escaped character in JSON at position 53 (line 1 column 54)
```

## Por que acontece

Claude Code envia input via stdin para hooks como JSON. No Windows, o campo `cwd` contem paths com backslashes. Intermitentemente, o Claude Code envia esses paths **sem escapar**:

```
Esperado: "cwd":"C:\\diretorio\\project-dir"
Recebido: "cwd":"C:\diretorio\project-dir"
```

O `\_` nao e um escape JSON valido → `JSON.parse()` falha → hook nao executa → **SYNAPSE rules nao sao injetadas naquele prompt**.

## Impacto na ausencia de SYNAPSE

Quando o JSON parse falha:
- O hook captura o erro e sai silenciosamente
- Claude Code nao recebe o `additionalContext` com as regras
- O prompt roda sem Constitution, coding standards e domain rules
- O proximo prompt geralmente funciona (bug intermitente)

## Correcao: sanitizeJsonString()

Adicionado fallback com sanitizacao de backslashes em `synapse-engine.cjs`:

```javascript
function sanitizeJsonString(raw) {
  // Escapa backslashes que nao sao escape sequences JSON validas
  // Validos: \" \\ \/ \b \f \n \r \t \uXXXX
  return raw.replace(/\\(?!["\\/bfnrtu])/g, '\\\\');
}
```

**Estrategia try-catch duplo no `readStdin()`:**
1. Tenta `JSON.parse(data)` normalmente
2. Se falhar, tenta `JSON.parse(sanitizeJsonString(data))`
3. Se ambos falharem, rejeita (hook sai silenciosamente)

## Validacao

- SYNAPSE rules injetadas normalmente (verificado via `[CONTEXT BRACKET]`)
- Erro nao aparece mais nos logs
- Zero impacto em performance (regex so executa no path de fallback)

## Arquivo modificado

- `.claude/hooks/synapse-engine.cjs` — funcoes `readStdin()` e `sanitizeJsonString()`

---

*Documentado em 2026-03-05 — RIAWORKS*
