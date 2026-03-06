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
