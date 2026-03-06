# AIOS-AIOX-RIAWORKS

> RIAWORKS fixes and logging extensions for the [Synkra AIOX](https://github.com/SynkraAI/aiox-core) hook system, focused on Claude Code context injection.

**[Leia em Portugues (pt-BR)](./README.pt-BR.md)**

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Files](#files)
- [Logging Reference (rw- logs)](#logging-reference-rw--logs)
  - [rw-hooks-log](#1-rw-hooks-log--operational-status)
  - [rw-synapse-trace](#2-rw-synapse-trace--synapse-xml-trace)
  - [rw-intel-context-log](#3-rw-intel-context-log--code-intel-injection)
  - [rw-context-log-full](#4-rw-context-log-full--unified-full-log)
  - [Individual vs Full](#individual-vs-full)
- [Activation](#activation)
- [Prompt: Apply All Fixes](#prompt-apply-all-fixes)
- [Prompt: Toggle Logging](#prompt-toggle-logging)
- [Original Repository](#original-repository)
- [License](#license)

---

## Overview

AIOX uses Claude Code hooks to inject SYNAPSE rules (coding standards, constitution, domain context) on every prompt. In the original repository, these hooks have documented bugs that cause **silent context loss** — Claude Code operates without project rules with no warning.

This repository contains:
- **8 bug fixes** for the hook system
- **4 logging extensions** (`rw-*`) to diagnose and trace context injection

## Prerequisites

### Execution directory

The prompt **must be executed at the project root** — the directory that contains `.aiox-core/`. This is the same directory where Claude Code is running your project.

```
your-project/              <-- execute here
├── .aiox-core/
├── .claude/
├── .synapse/
├── aios-aiox-riaworks/    <-- this repo must be here
└── ...
```

### This repository

The `aios-aiox-riaworks/` folder must exist at the project root. If it does not exist, the self-service prompt will ask for permission to clone it:

```bash
git clone https://github.com/riaworks/aios-aiox-riaworks.git
```

> **Important:** Claude Code can read all fix documentation and code snippets directly from the files in `aios-aiox-riaworks/`. The prompts reference these files — no need to copy code manually.

## Quick Start

1. Make sure you are at the project root (where `.aiox-core/` lives)
2. Make sure `aios-aiox-riaworks/` exists (or let the prompt clone it)
3. Copy the [Apply All Fixes prompt](#prompt-apply-all-fixes) into Claude Code
4. After fixes are applied, optionally [enable logging](#activation)

## Files

### Setup & Installation

| File | Description |
|------|-------------|
| [`01-fix-hook-synapse.md`](./01-fix-hook-synapse.md) | SYNAPSE setup and installation guide. How to obtain `.synapse/` from the [official AIOX repository](https://github.com/SynkraAI/aiox-core), expected structure, hook configuration, and diagnostics. |

### Fixes

| File | Description |
|------|-------------|
| [`02-fix-hooks-bugs.md`](./02-fix-hooks-bugs.md) | 7 bugs fixed: wrong hook registration, missing `hookEventName`, `process.exit()` killing stdout on Windows, sessions not persisted, absolute paths, 10ms timeout, PreCompact runner not found. |
| [`03-fix-windows-json-escape.md`](./03-fix-windows-json-escape.md) | Fix for intermittent JSON parse failure on Windows when Claude Code sends unescaped backslashes (`C:\dir` instead of `C:\\dir`). |

### Logging (rw- extensions)

| File | Description |
|------|-------------|
| [`rw-hooks-log.md`](./rw-hooks-log.md) | `rwHooksLog()` — operational hook status |
| [`rw-synapse-trace.md`](./rw-synapse-trace.md) | `rwSynapseTrace()` — full SYNAPSE XML trace |
| [`rw-intel-context-log.md`](./rw-intel-context-log.md) | `rwIntelContextLog()` — code-intel injection log |
| [`rw-context-log-full.md`](./rw-context-log-full.md) | `rwContextLogFull()` — unified full log |

---

## Logging Reference (rw- logs)

All RIAWORKS logging extensions use the `rw-` prefix. There are **4 logs**: 3 individual + 1 unified.

### Quick Reference

| # | Log | Env Var | Log File | Hook | Weight |
|---|-----|---------|----------|------|--------|
| 1 | **rw-hooks-log** | `RW_HOOKS_LOG=1` | `.logs/rw-hooks-log.log` | UserPromptSubmit | Light (~100B/prompt) |
| 2 | **rw-synapse-trace** | `RW_SYNAPSE_TRACE=1` | `.logs/rw-synapse-trace.log` | UserPromptSubmit | Heavy (~4KB/prompt) |
| 3 | **rw-intel-context-log** | `RW_INTEL_CONTEXT_LOG=1` | `.logs/rw-intel-context-log.log` | PreToolUse (Write/Edit) | Conditional |
| 4 | **rw-context-log-full** | `RW_CONTEXT_LOG_FULL=1` | `.logs/rw-context-log-full.log` | Both | Heavy (~5-10KB/prompt) |

---

### 1. rw-hooks-log — Operational Status

**Purpose:** Answers "is the hook working or failing?"

Records hook lifecycle events: session created, runtime resolved, errors. Does **not** record content (prompts, XML).

**When to use:** First line of diagnosis. If hooks are silently failing, this log shows exactly where the flow breaks.

| Log Entry | Meaning | SYNAPSE Injected? |
|-----------|---------|-------------------|
| `Session created: {id}` | First session run | Yes (next log confirms) |
| `Runtime resolved` | Hook executed successfully | Yes |
| `Hook output: N rules` | Rules generated and written to stdout | Yes |
| `No .synapse/ directory` | Missing .synapse/ | No |
| `Failed to resolve runtime` | Engine/session error | No |
| `Hook crashed` | Fatal error | No |

**Activate individually:**
```json
"command": "RW_HOOKS_LOG=1 node .claude/hooks/synapse-engine.cjs"
```

**Watch:**
```bash
tail -f .logs/rw-hooks-log.log
```

Full documentation: [`rw-hooks-log.md`](./rw-hooks-log.md)

---

### 2. rw-synapse-trace — SYNAPSE XML Trace

**Purpose:** Answers "what rules exactly were injected?"

Records the full SYNAPSE XML injected as `additionalContext` on every prompt, including user prompt, session ID, and bracket.

**When to use:** When you need to see the exact rules Claude received. Useful to detect missing rules, wrong bracket, or empty injection.

**Activate individually:**
```json
"command": "RW_SYNAPSE_TRACE=1 node .claude/hooks/synapse-engine.cjs"
```

**Watch:**
```bash
tail -f .logs/rw-synapse-trace.log
```

Full documentation: [`rw-synapse-trace.md`](./rw-synapse-trace.md)

---

### 3. rw-intel-context-log — Code-Intel Injection

**Purpose:** Answers "what code context was injected when the agent edited this file?"

Records the `<code-intel-context>` XML injected on **Write/Edit** operations only. Shows entities, references, and dependencies for the file being edited.

**When to use:** When you suspect code-intel is not providing context for edited files, or want to verify what entity data Claude sees.

> **Note:** This log runs on a different hook (`PreToolUse`) than the other two. It only fires when the agent writes or edits a file, not on every prompt.

**Activate individually:**
```json
"command": "RW_INTEL_CONTEXT_LOG=1 node .claude/hooks/code-intel-pretool.cjs"
```

> This goes in the `PreToolUse` hook section, **not** in `UserPromptSubmit`.

**Watch:**
```bash
tail -f .logs/rw-intel-context-log.log
```

Full documentation: [`rw-intel-context-log.md`](./rw-intel-context-log.md)

---

### 4. rw-context-log-full — Unified Full Log

**Purpose:** Answers "what is the complete context Claude is receiving?"

Captures **everything** in a single chronological log file:

| Section | Source | When |
|---------|--------|------|
| `[USER PROMPT]` | User text | Every prompt |
| `[SESSION]` | Session ID + bracket | Every prompt |
| `[SYNAPSE INJECTION]` | Full `<synapse-rules>` XML | Every prompt |
| `[STATIC CONTEXT]` | CLAUDE.md, rules/*.md, MEMORY.md listing | Every prompt |
| `[CODE-INTEL INJECTION]` | `<code-intel-context>` XML | Every Write/Edit |

**When to use:** When you want a single place to see everything, instead of checking multiple log files.

**Activate:**

> **Important:** `RW_CONTEXT_LOG_FULL=1` must be added to **both** hooks:

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

**Watch:**
```bash
tail -f .logs/rw-context-log-full.log
```

Full documentation: [`rw-context-log-full.md`](./rw-context-log-full.md)

---

### Individual vs Full

You can use any log individually, or activate the full unified log. Here is the relationship:

| Individual Log | Included in Full? | Can be used alone? |
|----------------|--------------------|--------------------|
| `RW_HOOKS_LOG` | **Yes** | Yes |
| `RW_SYNAPSE_TRACE` | **Yes** | Yes |
| `RW_INTEL_CONTEXT_LOG` | **Yes** | Yes |
| `RW_CONTEXT_LOG_FULL` | N/A — **is the unified master** | Yes |

**Key points:**

- **Full replaces all 3 individuals.** When `RW_CONTEXT_LOG_FULL=1` is active, you do **not** need `RW_HOOKS_LOG`, `RW_SYNAPSE_TRACE`, or `RW_INTEL_CONTEXT_LOG`.
- **Individuals can be combined.** You can activate any combination of the 3 individual logs. Example: `RW_HOOKS_LOG=1 RW_SYNAPSE_TRACE=1` for operational + XML trace without code-intel.
- **Full writes to a single file.** All 3 individual logs write to separate files. Full writes everything to `.logs/rw-context-log-full.log`.
- **You can mix.** Activating both `RW_CONTEXT_LOG_FULL=1` and an individual log will write to both files (no conflict, but redundant).

#### Recommended combinations

| Scenario | Configuration |
|----------|---------------|
| Quick health check | `RW_HOOKS_LOG=1` only |
| Debug missing rules | `RW_HOOKS_LOG=1 RW_SYNAPSE_TRACE=1` |
| Debug code-intel only | `RW_INTEL_CONTEXT_LOG=1` on PreToolUse |
| Full diagnostic session | `RW_CONTEXT_LOG_FULL=1` on both hooks |

---

## Activation

Logging variables are **not** activated via `export` in the terminal. They are set as **inline env vars** directly in the hook command inside `.claude/settings.local.json`.

### Where to configure

File: `.claude/settings.local.json` → `hooks` section

There are **two separate hooks** that accept logging:

| Hook Event | Script | Accepts |
|------------|--------|---------|
| `UserPromptSubmit` | `synapse-engine.cjs` | `RW_HOOKS_LOG`, `RW_SYNAPSE_TRACE`, `RW_CONTEXT_LOG_FULL` |
| `PreToolUse` (Write\|Edit) | `code-intel-pretool.cjs` | `RW_INTEL_CONTEXT_LOG`, `RW_CONTEXT_LOG_FULL` |

### Examples

**Default (all logging disabled):**
```json
"command": "node .claude/hooks/synapse-engine.cjs"
"command": "node .claude/hooks/code-intel-pretool.cjs"
```

**Hook log only (lightweight):**
```json
"command": "RW_HOOKS_LOG=1 node .claude/hooks/synapse-engine.cjs"
```

**Hook log + SYNAPSE trace (recommended for debug):**
```json
"command": "RW_HOOKS_LOG=1 RW_SYNAPSE_TRACE=1 node .claude/hooks/synapse-engine.cjs"
```

**Full unified (both hooks):**
```json
"command": "RW_CONTEXT_LOG_FULL=1 node .claude/hooks/synapse-engine.cjs"
"command": "RW_CONTEXT_LOG_FULL=1 node .claude/hooks/code-intel-pretool.cjs"
```

### To disable

Remove the env var prefixes from the command, leaving only:
```json
"command": "node .claude/hooks/synapse-engine.cjs"
"command": "node .claude/hooks/code-intel-pretool.cjs"
```

### Behavior (all logs)

- **Opt-in:** Only writes when the env var is set to `1`
- **Fire-and-forget:** Never blocks hook execution
- **Auto-create:** Creates `.logs/` with `.gitignore` if it doesn't exist
- **Append-only:** Never overwrites, always appends

---

## Prompt: Apply All Fixes

Copy the prompt below and paste it into Claude Code. It will read the fix documentation from `aios-aiox-riaworks/` and apply all fixes to your project.

> **Prerequisites:** Your project must have the AIOX hook structure (`.claude/hooks/`, `.aiox-core/core/synapse/`, `.claude/settings.local.json`).

### Self-Service Apply Prompt

````
I need you to apply all RIAWORKS fixes and logging extensions to this AIOX project.

## STEP 0 — VERIFY REPOSITORY

Check if the folder `aios-aiox-riaworks/` exists at the project root (same level as `.aiox-core/`).

If it does NOT exist, ask me:
"The aios-aiox-riaworks repository is not present. Can I clone it from
https://github.com/riaworks/aios-aiox-riaworks.git into this directory?"

Wait for my confirmation before cloning. If it already exists, continue.

## STEP 1 — READ ALL DOCUMENTATION

Read these files from the `aios-aiox-riaworks/` directory. They contain all fix details,
code snippets, and expected behavior. Do NOT guess — use the exact code from these files:

1. `aios-aiox-riaworks/01-fix-hook-synapse.md` — SYNAPSE setup requirements
2. `aios-aiox-riaworks/02-fix-hooks-bugs.md` — All 7 bug fixes with code
3. `aios-aiox-riaworks/03-fix-windows-json-escape.md` — JSON escape fix with code
4. `aios-aiox-riaworks/rw-hooks-log.md` — rwHooksLog() function and usage
5. `aios-aiox-riaworks/rw-synapse-trace.md` — rwSynapseTrace() function and usage
6. `aios-aiox-riaworks/rw-intel-context-log.md` — rwIntelContextLog() function and usage
7. `aios-aiox-riaworks/rw-context-log-full.md` — rwContextLogFull() function and usage

After reading, report a summary of what you found.

## STEP 2 — INTEGRITY CHECK (read-only, do NOT edit yet)

Read and verify these target files:

1. `.claude/settings.local.json` — check if hooks.UserPromptSubmit exists
2. `.claude/hooks/synapse-engine.cjs` — check if readStdin() and main() exist
3. `.aiox-core/core/synapse/runtime/hook-runtime.js` — check if buildHookOutput() exists
4. `.claude/hooks/code-intel-pretool.cjs` — check if it references .aios-core or .aiox-core
5. `.claude/hooks/precompact-session-digest.cjs` — check if it exists

For each file, report:
- EXISTS: yes/no
- KEY FUNCTIONS FOUND: list them
- ALREADY PATCHED: yes/no (compare against the fixes in the documentation)

Do NOT proceed to Step 3 until you report findings and I confirm.

## STEP 3 — APPLY FIXES

Apply each fix described in the documentation files you read in Step 1.
Use the exact code from the documentation — do NOT invent or modify.

For each fix:
- If ALREADY APPLIED, skip and report "already patched"
- If target file MISSING, report and ask before proceeding

## STEP 4 — APPLY LOGGING EXTENSIONS

Apply the 4 logging functions (rwHooksLog, rwSynapseTrace, rwIntelContextLog,
rwContextLogFull) as described in their respective documentation files.

## STEP 5 — VALIDATION

Run the verification command from `02-fix-hooks-bugs.md` and report the result.
Create `.logs/` directory with `.gitignore` containing `*` if it doesn't exist.

## RULES
- Do NOT skip Step 1. Reading the documentation is mandatory.
- Do NOT skip Step 2. Integrity check is mandatory.
- ALL code comes from the documentation files — never invent code.
- If a target file does not exist or changed structure, ask before proceeding.
````

---

## Prompt: Toggle Logging

After the fixes are installed, use these prompts to enable or disable logging.

### Prompt to ENABLE logging

```
Read `aios-aiox-riaworks/README.md`, section "Activation".
Then read `.claude/settings.local.json` and locate the hook commands.

Before modifying, verify:
1. `.claude/hooks/synapse-engine.cjs` exists and contains rwHooksLog/rwSynapseTrace
2. `.claude/hooks/code-intel-pretool.cjs` exists and contains rwIntelContextLog
3. The commands in settings.local.json point to the correct paths

If all checks pass, update the commands to enable full unified logging:
- UserPromptSubmit: "RW_CONTEXT_LOG_FULL=1 node .claude/hooks/synapse-engine.cjs"
- PreToolUse: "RW_CONTEXT_LOG_FULL=1 node .claude/hooks/code-intel-pretool.cjs"

If any check fails, report what changed and suggest the correct fix.
```

### Prompt to DISABLE logging

```
Read `.claude/settings.local.json` and locate the hook commands.
Verify that both hook scripts exist at their expected paths.

If verified, update both commands to remove all RW_ env vars:
- UserPromptSubmit: "node .claude/hooks/synapse-engine.cjs"
- PreToolUse: "node .claude/hooks/code-intel-pretool.cjs"

If paths or structure changed, report before editing.
```

### Why integrity checks matter

The AIOX framework evolves across sessions. Files may be renamed, methods refactored, or hook registration restructured. The prompts above force Claude to:

1. **Read the documentation first** — all code comes from `aios-aiox-riaworks/` files, not from memory
2. **Verify file existence** — confirm scripts are still at expected paths
3. **Verify method signatures** — confirm rw functions exist in source
4. **Report before acting** — explain discrepancies instead of applying broken edits

---

## Modified Files in the Project

| File | Changes |
|------|---------|
| `.claude/settings.local.json` | Relative paths, no timeout, hooks on correct events |
| `.aiox-core/core/synapse/runtime/hook-runtime.js` | `hookEventName`, `createSession()`, `rwHooksLog()`, `cleanOrphanTmpFiles()` |
| `.claude/hooks/synapse-engine.cjs` | `sanitizeJsonString()`, `rwSynapseTrace()`, `rwContextLogFull()`, removal of `process.exit()` |
| `.claude/hooks/code-intel-pretool.cjs` | Path `.aios-core` → `.aiox-core`, `rwIntelContextLog()`, `rwContextLogFull()` |
| `.aiox-core/hooks/unified/runners/precompact-runner.js` | Runner copied from fork with adapted paths |
| `bin/utils/pro-detector.js` | Precompact runner dependency |
| `.logs/` | Directory created with `.gitignore` |

## Naming Convention

All RIAWORKS extensions use the `rw` prefix to differentiate from original AIOX code:

| Function | Env Var | Log File |
|----------|---------|----------|
| `rwHooksLog()` | `RW_HOOKS_LOG=1` | `.logs/rw-hooks-log.log` |
| `rwSynapseTrace()` | `RW_SYNAPSE_TRACE=1` | `.logs/rw-synapse-trace.log` |
| `rwIntelContextLog()` | `RW_INTEL_CONTEXT_LOG=1` | `.logs/rw-intel-context-log.log` |
| `rwContextLogFull()` | `RW_CONTEXT_LOG_FULL=1` | `.logs/rw-context-log-full.log` |

## Original Repository

- **AIOX Core:** [github.com/SynkraAI/aiox-core](https://github.com/SynkraAI/aiox-core)
- **This repo:** [github.com/riaworks/aios-aiox-riaworks](https://github.com/riaworks/aios-aiox-riaworks)
- **AIOX** is the AI-Orchestrated System for Full Stack Development by Synkra AI

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
