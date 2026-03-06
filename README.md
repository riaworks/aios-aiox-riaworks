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

## Quick Start

```bash
# Enable hook execution logging
export RW_HOOKS_LOG=1

# Enable SYNAPSE injection tracing
export RW_SYNAPSE_TRACE=1

# Watch logs in real time
tail -f .logs/hook-ops.log
tail -f .logs/synapse-trace.log
```

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
