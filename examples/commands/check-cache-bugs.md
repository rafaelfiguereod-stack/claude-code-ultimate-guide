---
name: check-cache-bugs
description: Audit Claude Code setup for cache bugs (CC#40524) — sentinel, --resume/--continue, attribution header
---

# Check Cache Bugs (CC#40524)

Audit your Claude Code setup for three cache bugs discovered in March 2026 that silently inflate API costs by 2-5x on input tokens.

**Time**: ~20 seconds | **Scope**: version, config files, CLAUDE.md, skills, hooks, shell profiles, all claude binaries

> **Note on cache contamination**: This skill loads content containing `cch=` strings into the current session's message array. If you are on a standalone binary (Bun-based, not npm), this is the exact trigger condition for Bug 1. For Bug 3, loading this skill into a long session may also shift cache offsets. For cleanest results, run this command at the very start of a fresh session before any other context is loaded — or run it via `claude -p "$(cat .claude/commands/check-cache-bugs.md)"` as a one-shot print-mode invocation outside your current session.

**Reference**: `anthropics/claude-code#40524` | **Discovered by**: `@jmarianski` (GitHub / u/skibidi-toaleta-2137, r/ClaudeAI) + `@whiletrue0x`

---

## Background

Three independent bugs break Anthropic's prefix-based prompt caching, causing `cache_creation` charges
instead of `cache_read` on the ~12K-token system prompt. Confirmed via source code analysis of the
leaked npm sourcemap (March 2026). Cost impact: 2-5x inflation on input tokens (not 10-20x total —
early community estimates conflated system prompt tokens with total session cost).

- **Bug 1** (standalone binary, v2.1.36+, low/speculative): Bun's native HTTP stack performs a
  same-length byte replacement of the `cch=00000` attestation placeholder in the serialized request
  body after `JSON.stringify` but before TLS. If `cch=00000` appears literally in `messages[]`
  content (e.g., a CLAUDE.md discussing this bug), the Zig layer may replace it in the wrong
  location. Whether the search is bounded or naive is unconfirmed from the TypeScript side — failure
  mode is likely a 400 error, not a silent cache miss. Edge case in normal usage.

- **Bug 2** (v2.1.69+, HIGH): The session JSONL writer strips `deferred_tools_delta` attachment
  records before writing to disk. On `--resume`, those records are gone — the deferred tools layer
  has no prior announcement history and re-announces all tools from scratch. This shifts every
  message position in the restored conversation, breaking the messages-level cache prefix entirely.
  Concrete evidence: every resume event drops `cache_read` to 0 and rebuilds 87-118K tokens as
  `cache_creation`. 3-4 resumes per session = 300-400K tokens of avoidable cost. Scales with number
  of skills/deferred tools — 10+ skills = worst case. Anthropic tracking internally (inc-4747).
  Status: still active in v2.1.88.

- **Bug 3** (v2.1.69+, low-to-medium): Claude Code injects a billing header as the **first block**
  of the system prompt on every API request. The header contains a 3-character SHA-256 hash derived
  from characters at positions [4, 7, 20] of the first user message + CC version, making it unique
  per session/subagent/side-query. Since the cache is prefix-based, this causes a cold miss on the
  ~12K-token system prompt on every invocation. Per the original RE analyst (jmarianski): "marginal
  impact" in practice relative to total session cost — Bug 2 is larger for heavy users. Empirical:
  48% → 99.98% cache hit with env var fix (combined effect). Status: still active in v2.1.88.

Partial fix shipped in v2.1.88 (tool schema bytes). Bugs 2 and 3 remain active.

---

## Instructions

You are an auditor. Run all phases in order, collect every result, and produce the final report. Do not skip phases or stop early.

---

### Phase 1 — Claude Code version and install method

```bash
# Version check
claude --version

# All installed claude binaries
which -a claude 2>/dev/null

# Check if active binary is standalone or npm
file $(which claude) 2>/dev/null
ls -la $(which claude) 2>/dev/null
```

- Version >= 2.1.36 AND standalone binary → flag **BUG 1 MECHANISM PRESENT** (edge case, only triggers if sentinel in messages)
- Version >= 2.1.69 → flag **BUG 2 RISK** and **BUG 3 RISK**
- Binary is Mach-O / ELF executable → standalone → **Bug 1 mechanism present**
- Binary is a symlink to `node_modules` or contains `cli.js` → npm/npx → **Bug 1 does not apply**

---

### Phase 2 — Sentinel scan (Bug 1)

Search for the literal string `cch=` in all static config files. Exclude `.jsonl` files (ephemeral conversation history) and this command file itself.

```bash
# Global config files
grep -r "cch=" \
  ~/.claude/CLAUDE.md \
  ~/.claude/MEMORY.md \
  ~/.claude/TONE.md \
  ~/.claude/FLAGS.md \
  ~/.claude/RULES.md \
  ~/.claude/RTK.md \
  ~/.claude/ANTI_AI.md \
  2>/dev/null

# Global skills, commands, agents, hooks (excluding this command file)
grep -rl "cch=" ~/.claude/skills/ 2>/dev/null
grep -rl "cch=" ~/.claude/commands/ --exclude="check-cache-bugs.md" 2>/dev/null
grep -rl "cch=" ~/.claude/agents/ 2>/dev/null
grep -rl "cch=" ~/.claude/hooks/ 2>/dev/null

# Project-level config
grep -r "cch=" CLAUDE.md .claude/CLAUDE.md .claude/MEMORY.md 2>/dev/null
grep -rl "cch=" .claude/skills/ 2>/dev/null
grep -rl "cch=" .claude/commands/ --exclude="check-cache-bugs.md" 2>/dev/null
grep -rl "cch=" .claude/agents/ 2>/dev/null
grep -rl "cch=" .claude/hooks/ 2>/dev/null

# Broader scan: all CLAUDE.md files across projects
find ~ -name "CLAUDE.md" \
  -not -path "*/node_modules/*" \
  -not -path "*/.git/*" \
  2>/dev/null | xargs grep -l "cch=" 2>/dev/null
```

Flag as **BUG 1 RISK** if any match found outside `.jsonl` files.

---

### Phase 3 — Resume/continue usage (Bug 2)

```bash
# settings.json
grep -i -- "--resume\|--continue" ~/.claude/settings.json 2>/dev/null
grep -i -- "--resume\|--continue" .claude/settings.json 2>/dev/null

# Hooks
grep -rn -- "--resume\|--continue" ~/.claude/hooks/ 2>/dev/null
grep -rn -- "--resume\|--continue" .claude/hooks/ 2>/dev/null

# Commands and skills
grep -rn -- "--resume\|--continue" ~/.claude/commands/ 2>/dev/null
grep -rn -- "--resume\|--continue" .claude/commands/ 2>/dev/null

# Shell profiles (aliases, functions)
grep -n -- "--resume\|--continue" ~/.zshrc ~/.bashrc ~/.bash_profile ~/.zprofile 2>/dev/null

# Project scripts
find . -name "*.sh" -o -name "Makefile" 2>/dev/null | \
  xargs grep -l -- "--resume\|--continue" 2>/dev/null
```

- Any hit in hooks/settings → flag **BUG 2 AUTOMATED** (constant exposure)
- Any hit in commands/skills/scripts → flag **BUG 2 MANUAL** (exposure when invoked)

---

### Phase 4 — Attribution header check (Bug 3)

Check whether the billing header env var is already disabled.

```bash
# Global settings
grep -i "CLAUDE_CODE_ATTRIBUTION_HEADER\|ENABLE_TOOL_SEARCH" \
  ~/.claude/settings.json 2>/dev/null

# Project settings
grep -i "CLAUDE_CODE_ATTRIBUTION_HEADER\|ENABLE_TOOL_SEARCH" \
  .claude/settings.json 2>/dev/null

# Shell profiles
grep -i "CLAUDE_CODE_ATTRIBUTION_HEADER" \
  ~/.zshrc ~/.bashrc ~/.bash_profile ~/.zprofile 2>/dev/null
```

- `CLAUDE_CODE_ATTRIBUTION_HEADER` not set to `false` AND version >= 2.1.69 → flag **BUG 3 ACTIVE**
- Already set to `false` → **BUG 3 MITIGATED**

---

### Phase 5 — Multiple binaries check

```bash
for b in $(which -a claude 2>/dev/null | sort -u); do
  echo "=== $b ==="
  $b --version 2>/dev/null || echo "unavailable"
  file $b 2>/dev/null
  ls -la $b 2>/dev/null
done
```

Flag any standalone binary >= 2.1.69 as at risk for all three bugs.
Note stale npm binaries that could be mistakenly invoked.

---

## Output Format

```
## Claude Code Cache Bug Audit — CC#40524

**Date**: [today]
**Active claude version**: [version]
**Install method**: [standalone binary | npm/npx | mixed]

---

### Bug 1 — Sentinel replacement (standalone binary v2.1.36+, edge case)
**Status**: [SAFE / AT RISK / NOT APPLICABLE]

**Mechanism active (standalone binary >= v2.1.36)**: [YES / NO]
**Trigger sentinel found in static config**: [YES — locations | NO]

[If AT RISK] `cch=00000` found in static config files:
- [file path]: [matching line]
→ Fix: remove the `cch=00000` string from those files.
→ Temporary workaround: `npx @anthropic-ai/claude-code` (npm package uses standard Bun, no replacement).
→ Note: sentinel in `.jsonl` conversation history is normal and harmless — only static config matters.

[If SAFE] Mechanism present in binary but no trigger in static config. Normal usage unaffected.
[If NOT APPLICABLE] npm/npx install — Bug 1 mechanism absent at the binary level.

---

### Bug 2 — Cache prefix mismatch on --resume / --continue (v2.1.69+)
**Status**: [SAFE / AT RISK (automated) / AT RISK (manual) / NOT APPLICABLE]

**Version in affected range (>= 2.1.69)**: [YES / NO]
**Automated usage (hooks/settings)**: [YES — locations | NO]
**Manual usage (commands/skills/scripts)**: [YES — locations | NO]

**Root cause**: `deferred_tools_delta` attachment introduced in v2.1.69 causes `messages[0]` to differ between fresh and resumed sessions, breaking cache prefix matching independently of Bug 1.

[If AT RISK]
→ Avoid `--resume` and `--continue` until fix ships.
→ Downgrade workaround: `npm install -g @anthropic-ai/claude-code@2.1.68` (last version before regression)

[If NOT APPLICABLE] Version < 2.1.69 — not affected.

---

### Bug 3 — Attribution header per-session hash (widest impact)
**Status**: [ACTIVE / MITIGATED / NOT APPLICABLE]

**Version in affected range (>= 2.1.69)**: [YES / NO]
**CLAUDE_CODE_ATTRIBUTION_HEADER=false already set**: [YES / NO]

[If ACTIVE] Every session start and every subagent call misses the system prompt cache (~12K tokens rebuilt at cache_creation rate).
→ Fix: add to ~/.claude/settings.json:
  {
    "env": {
      "CLAUDE_CODE_ATTRIBUTION_HEADER": "false",
      "ENABLE_TOOL_SEARCH": "false"
    }
  }
→ Expected impact: cache hit ratio 48% → ~99.98% (measured, source: @whiletrue0x CC#40524)

[If MITIGATED] Header already disabled. No action needed.
[If NOT APPLICABLE] Version < 2.1.69 — not affected.

---

### Multiple binaries
[List each binary found, version, type (standalone/npm), and per-bug status]

---

### Summary

| Bug | Impact | Status | Action |
|-----|--------|--------|--------|
| Bug 1 — sentinel in config | Edge case | [SAFE / AT RISK / N/A] | [action or "none"] |
| Bug 2 — --resume/--continue | Per-resume cache miss | [SAFE / AT RISK / N/A] | [action or "none"] |
| Bug 3 — attribution header | Every session + subagent | [ACTIVE / MITIGATED / N/A] | [action or "none"] |
| Stale binaries | — | [CLEAN / PRESENT] | [remove or none] |

[If Bug 3 ACTIVE — always show this]
⚡ Quick win: add CLAUDE_CODE_ATTRIBUTION_HEADER=false to settings.json — immediate effect, no restart needed.

[If all SAFE/MITIGATED/N/A]
✅ No active exposure. Re-run after updating Claude Code.

⚠️ Track fixes: github.com/anthropics/claude-code/issues/40524
```
