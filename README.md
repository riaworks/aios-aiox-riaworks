# AIOS-AIOX-RIAWORKS

> RIAWORKS fixes and extensions for the [Synkra AIOS](https://github.com/synkra-ai/aios-core) / AIOX hook system, focused on Claude Code context injection.

**[Leia em Portugues (pt-BR)](./README.pt-BR.md)**

---

## Overview

AIOS uses Claude Code hooks to inject SYNAPSE rules (coding standards, constitution, domain context) on every prompt. In the original repository, these hooks have documented bugs that cause **silent context loss** — Claude Code operates without project rules with no warning.

This repository contains the applied fixes and two logging systems created by RIAWORKS to diagnose and trace context injection.

## Files

### Fixes

| File | Description |
|------|-------------|
| [`fix-hooks-bugs.md`](./fix-hooks-bugs.md) | 7 bugs fixed in Claude Code hooks: wrong registration in settings.json, missing `hookEventName`, `process.exit()` killing stdout on Windows, sessions not persisted, absolute paths incompatible across IDEs, 10ms timeout, and PreCompact runner not found. |
| [`fix-windows-json-escape.md`](./fix-windows-json-escape.md) | Fix for an intermittent bug where Claude Code sends Windows paths without escaping backslashes in JSON (`C:\dir` instead of `C:\\dir`), causing `JSON.parse()` failure and SYNAPSE rules loss on that prompt. |

### Logging (RIAWORKS Extensions)

| File | Description |
|------|-------------|
| [`rw-hooks-log.md`](./rw-hooks-log.md) | Documentation for `rwHooksLog()` — lightweight operational log that records hook execution status (session created, runtime resolved, errors). Activated via `RW_HOOKS_LOG=1`. Writes to `.logs/hook-ops.log`. |
| [`rw-synapse-trace.md`](./rw-synapse-trace.md) | Documentation for `rwSynapseTrace()` — detailed trace that records user prompt, session ID, bracket, and the full XML injected as `additionalContext`. Activated via `RW_SYNAPSE_TRACE=1`. Writes to `.logs/synapse-trace.log`. |

## Naming Convention

All RIAWORKS extensions use the `rw` prefix to differentiate from original code:

| Function | Env Var | Log File |
|----------|---------|----------|
| `rwHooksLog()` | `RW_HOOKS_LOG=1` | `.logs/hook-ops.log` |
| `rwSynapseTrace()` | `RW_SYNAPSE_TRACE=1` | `.logs/synapse-trace.log` |

## Modified Files in the Project

| File | Changes |
|------|---------|
| `.claude/settings.local.json` | Relative paths, no timeout, hooks registered on correct events |
| `.aiox-core/core/synapse/runtime/hook-runtime.js` | `hookEventName`, `createSession()`, `rwHooksLog()`, `cleanOrphanTmpFiles()` |
| `.claude/hooks/synapse-engine.cjs` | `sanitizeJsonString()`, `rwSynapseTrace()`, removal of `process.exit()` |
| `.claude/hooks/code-intel-pretool.cjs` | Path `.aios-core` corrected to `.aiox-core` |
| `.aiox-core/hooks/unified/runners/precompact-runner.js` | Runner copied from fork with adapted paths |
| `bin/utils/pro-detector.js` | Precompact runner dependency |
| `.logs/` | Directory created with `.gitignore` |

## Activation

The logging variables are **not** activated via `export` in the terminal. They are set as **inline env vars** directly in the hook command inside `.claude/settings.local.json`.

### Where to configure

File: `.claude/settings.local.json` → `hooks.UserPromptSubmit[0].hooks[0].command`

### Examples

**Default (logging disabled):**
```json
"command": "node .claude/hooks/synapse-engine.cjs"
```

**Enable hook execution log only:**
```json
"command": "RW_HOOKS_LOG=1 node .claude/hooks/synapse-engine.cjs"
```

**Enable SYNAPSE trace only:**
```json
"command": "RW_SYNAPSE_TRACE=1 node .claude/hooks/synapse-engine.cjs"
```

**Enable both (recommended for debugging):**
```json
"command": "RW_HOOKS_LOG=1 RW_SYNAPSE_TRACE=1 node .claude/hooks/synapse-engine.cjs"
```

### Log output

| Variable | Log File | Content |
|----------|----------|---------|
| `RW_HOOKS_LOG=1` | `.logs/hook-ops.log` | Hook lifecycle: session created, runtime resolved, errors |
| `RW_SYNAPSE_TRACE=1` | `.logs/synapse-trace.log` | Full SYNAPSE XML injected as `additionalContext`, session ID, bracket |

```bash
# Watch logs in real time
tail -f .logs/hook-ops.log
tail -f .logs/synapse-trace.log
```

### To disable

Remove the env var prefixes from the command, leaving only:
```json
"command": "node .claude/hooks/synapse-engine.cjs"
```

---

## Self-Service: Prompt for Claude to Toggle Logging

Since the activation requires editing `settings.local.json`, you can ask Claude itself to do it. Use the prompts below — they include **integrity verification** so Claude checks if file paths and method signatures still match before making changes.

### Prompt to ENABLE logging

```
Read the file .claude/settings.local.json and locate the UserPromptSubmit hook command
that runs synapse-engine.cjs. Before modifying, verify:

1. The file .claude/hooks/synapse-engine.cjs exists
2. It contains the functions rwHooksLog() and rwSynapseTrace()
3. The current command in settings.local.json points to the correct path

If all checks pass, update the command to:
"command": "RW_HOOKS_LOG=1 RW_SYNAPSE_TRACE=1 node .claude/hooks/synapse-engine.cjs"

If any check fails, report what changed and suggest the correct fix instead of
blindly applying the edit.
```

### Prompt to DISABLE logging

```
Read the file .claude/settings.local.json and locate the UserPromptSubmit hook command.
Verify that .claude/hooks/synapse-engine.cjs exists and the command path is correct.

If verified, update the command to:
"command": "node .claude/hooks/synapse-engine.cjs"

If the file path or structure has changed, report the discrepancy before editing.
```

### Why integrity checks matter

The AIOX framework evolves across sessions. Files may be renamed, methods refactored, or hook registration restructured. The prompts above force Claude to:

1. **Verify file existence** — confirm `synapse-engine.cjs` is still at the expected path
2. **Verify method signatures** — confirm `rwHooksLog()` and `rwSynapseTrace()` still exist in the source
3. **Verify settings structure** — confirm `UserPromptSubmit` hook is still registered in `settings.local.json`
4. **Report before acting** — if anything changed, Claude explains the discrepancy instead of applying a broken edit

---

## Self-Service: Prompt for Claude to Apply All Fixes and Logging

Copy the prompt below and paste it in Claude Code to have it apply all 8 bug fixes and both logging systems to your AIOS/AIOX project. The prompt includes integrity verification at every step.

> **Prerequisites:** Your project must have the AIOX hook structure (`.claude/hooks/`, `.aiox-core/core/synapse/`, `.claude/settings.local.json`).

### Full Apply Prompt

````
I need you to apply all RIAWORKS fixes and logging extensions to this AIOS/AIOX project.
Read the documentation below, verify every target file before editing, and apply each fix.

## PHASE 1 — INTEGRITY CHECK (read-only, do NOT edit yet)

Read these files and confirm they exist. Report their current state:

1. `.claude/settings.local.json` — check if hooks.UserPromptSubmit exists
2. `.claude/hooks/synapse-engine.cjs` — check if readStdin() and main() exist
3. `.aiox-core/core/synapse/runtime/hook-runtime.js` — check if buildHookOutput() exists
4. `.claude/hooks/code-intel-pretool.cjs` — check if it references .aios-core or .aiox-core
5. `.claude/hooks/precompact-session-digest.cjs` — check if it exists

For each file, report:
- EXISTS: yes/no
- KEY FUNCTIONS FOUND: list them
- ALREADY PATCHED: yes/no (check if fixes below are already applied)

Do NOT proceed to Phase 2 until you report all findings and I confirm.

## PHASE 2 — APPLY FIXES (edit only after Phase 1 confirmation)

### Fix 1: Hook registration in settings.local.json
In `.claude/settings.local.json`, ensure hooks are registered correctly:
- `UserPromptSubmit` must run ONLY `synapse-engine.cjs`
- `PreCompact` must run ONLY `precompact-session-digest.cjs`
- `PreToolUse` with matcher `Write|Edit` must run `code-intel-pretool.cjs`
- All commands must use RELATIVE paths (e.g., `node .claude/hooks/synapse-engine.cjs`)
- NO `timeout` field on any hook
- NO `${CLAUDE_PROJECT_DIR}` in any path

### Fix 2: hookEventName in buildHookOutput
In `.aiox-core/core/synapse/runtime/hook-runtime.js`, find `buildHookOutput()`.
Add `hookEventName: 'UserPromptSubmit'` inside `hookSpecificOutput` if missing:
```javascript
hookSpecificOutput: {
  hookEventName: 'UserPromptSubmit',
  additionalContext: xml || '',
},
```

### Fix 3: Remove process.exit() in synapse-engine.cjs
In `.claude/hooks/synapse-engine.cjs`, find the main() call at the bottom.
Replace:
```javascript
main().then(() => process.exit(0));
```
With:
```javascript
main().then(() => {}).catch(() => {});
```

### Fix 4: createSession() fallback in hook-runtime.js
In `.aiox-core/core/synapse/runtime/hook-runtime.js`, find where loadSession() is called.
Ensure there is a fallback to createSession() when loadSession returns null:
```javascript
let session = loadSession(sessionId, sessionsDir);
if (!session && sessionId) {
  session = createSession(sessionId, cwd, sessionsDir);
}
```

### Fix 5: Fix .aios-core path in code-intel-pretool.cjs
In `.claude/hooks/code-intel-pretool.cjs`, replace any reference to `.aios-core` with `.aiox-core`.

### Fix 6: Remove timeout from settings
In `.claude/settings.local.json`, remove any `"timeout": 10` or similar timeout fields from hook definitions.

### Fix 7: Verify PreCompact runner
Check if `.aiox-core/hooks/unified/runners/precompact-runner.js` exists.
If not, check if `bin/utils/pro-detector.js` exists.
Report what is found — do NOT create files that don't exist in the source.

### Fix 8: sanitizeJsonString() for Windows JSON escape
In `.claude/hooks/synapse-engine.cjs`, add this function if it doesn't exist:
```javascript
function sanitizeJsonString(raw) {
  return raw.replace(/\\(?!["\\/bfnrtu])/g, '\\\\');
}
```
Then in `readStdin()`, wrap JSON.parse in a try-catch with fallback:
```javascript
try {
  return JSON.parse(data);
} catch (e) {
  try {
    return JSON.parse(sanitizeJsonString(data));
  } catch (e2) {
    throw e; // throw original error
  }
}
```

## PHASE 3 — APPLY LOGGING EXTENSIONS

### rwHooksLog() in hook-runtime.js
In `.aiox-core/core/synapse/runtime/hook-runtime.js`, add this function if it doesn't exist:
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
Call it at key points: after session created, after runtime resolved, on errors.

### rwSynapseTrace() in synapse-engine.cjs
In `.claude/hooks/synapse-engine.cjs`, add this function if it doesn't exist:
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
Call it after the SYNAPSE engine generates the XML output.

## PHASE 4 — VALIDATION

After all edits, run this verification command:
```bash
echo '{"prompt":"test","session_id":"verify","cwd":"'$(pwd)'"}' \
  | node .claude/hooks/synapse-engine.cjs 2>/dev/null \
  | node -e "let d='';process.stdin.on('data',c=>d+=c);process.stdin.on('end',()=>{try{const j=JSON.parse(d);console.log('hookEventName:',j.hookSpecificOutput?.hookEventName);console.log('rules:',j.hookSpecificOutput?.additionalContext?.includes('CONSTITUTION')?'YES':'NO');console.log('STATUS: OK');}catch(e){console.log('STATUS: FAIL',e.message);}})"
```

Expected output:
```
hookEventName: UserPromptSubmit
rules: YES
STATUS: OK
```

Report the result. If FAIL, diagnose using the error message and fix before finishing.

## RULES
- Do NOT skip Phase 1. Integrity check is mandatory.
- Do NOT create files that don't exist — only modify existing ones.
- If a fix is ALREADY APPLIED, skip it and report "already patched".
- If a target file is MISSING or its structure changed, report the discrepancy and ask before proceeding.
- After Phase 4, create `.logs/` directory with a `.gitignore` containing `*` if it doesn't exist.
````

## Original Repository

- **AIOS Core:** [github.com/synkra-ai/aios-core](https://github.com/synkra-ai/aios-core)
- **AIOX** is the internal fork that extends AIOS with multi-agent orchestration

## License

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
