# Check Cache Bugs (CC#40524)

Audit your Claude Code setup for two cache bugs discovered in March 2026 that can silently inflate API costs by 10-20x.

**Time**: ~20 seconds | **Scope**: version, config files, CLAUDE.md, skills, hooks, shell profiles, all claude binaries

**Reference**: `anthropics/claude-code#40524` | **Discovered by**: `u/skibidi-toaleta-2137` on r/ClaudeAI

---

## Background

Two independent bugs cause Claude Code's prompt cache to break silently:

- **Bug 1** (standalone binary only): A native string replacement in Anthropic's custom Bun fork injects a billing sentinel into every API request. If the literal string `cch=00000` appears in your conversation history (from config files, CLAUDE.md, or loaded context), the replacement targets your messages instead of the system prompt, invalidating the cache on every request.
- **Bug 2** (v2.1.69+): A cache prefix mismatch when resuming sessions via `--resume` or `--continue` causes a full conversation rewrite instead of a cache read.

Both confirmed as regressions by Anthropic. Status: assigned, fix pending.

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

- Version >= 2.1.69 → flag **BUG 2 RISK**
- Binary is Mach-O / ELF executable → standalone → **Bug 1 mechanism present**
- Binary is a symlink to `node_modules` or contains `cli.js` → npm/npx → **Bug 1 does not apply**

---

### Phase 2 — Sentinel scan (Bug 1)

Search for the literal string `cch=` in all static config files. Exclude `.jsonl` files (ephemeral conversation history — always contain the sentinel after discussing this bug) and exclude this command file itself.

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

# Broader scan: all CLAUDE.md files across home directory projects
find ~ -name "CLAUDE.md" \
  -not -path "*/node_modules/*" \
  -not -path "*/.git/*" \
  -not -path "*/.jsonl" \
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
- Any hit in commands/skills → flag **BUG 2 MANUAL** (exposure when invoked)
- Shell alias or project script → flag **BUG 2 MANUAL**

---

### Phase 4 — Multiple binaries check

```bash
which -a claude
```

If multiple binaries found, check each one:

```bash
for b in $(which -a claude); do
  echo "=== $b ==="
  $b --version 2>/dev/null || echo "unavailable"
  file $b 2>/dev/null
  ls -la $b 2>/dev/null
done
```

Flag any standalone binary >= 2.1.69 as **BUG 2 RISK**, regardless of which is active.
Note any npm/npx binary (safe for Bug 1, check version for Bug 2).

---

## Output Format

```
## Claude Code Cache Bug Audit — CC#40524

**Date**: [today]
**Active claude version**: [version]
**Install method**: [standalone binary | npm/npx | mixed]

---

### Bug 1 — Sentinel replacement (standalone binary)
**Status**: [SAFE / AT RISK / NOT APPLICABLE]

[If AT RISK] Trigger sentinel found in:
- [file path]: [matching line]
→ Fix: remove or rename the `cch=...` string from those files.
→ Temporary workaround: `npx @anthropic-ai/claude-code` (no Bun fork, no replacement).

[If SAFE] No trigger sentinel in any static config. Normal usage is unaffected.

[If NOT APPLICABLE] npm/npx installation — Bug 1 mechanism does not exist.

---

### Bug 2 — Cache prefix mismatch on --resume / --continue
**Status**: [SAFE / AT RISK (automated) / AT RISK (manual) / NOT APPLICABLE]

**Version in affected range (>= 2.1.69)**: [YES / NO]
**Automated usage detected**: [YES / NO — list locations if YES]
**Manual usage risk**: [YES if commands/skills/scripts invoke --resume or --continue]

[If AT RISK] 
→ Fix: avoid `--resume` and `--continue` until Anthropic ships a fix.
→ Monitor: github.com/anthropics/claude-code/issues/40524

[If NOT APPLICABLE] Version predates v2.1.69 — not affected.

---

### Multiple binaries
[List each binary, version, type, and Bug 1/2 status]

---

### Summary

| Check | Status | Action |
|-------|--------|--------|
| Bug 1 — sentinel in config | [SAFE / AT RISK / N/A] | [action or "none"] |
| Bug 2 — --resume/--continue | [SAFE / AT RISK / N/A] | [action or "none"] |
| Stale binaries | [CLEAN / PRESENT] | [brew uninstall / none] |

[If all SAFE/N/A]
✅ No exposure detected. Re-run after updating Claude Code or adding new config files.

[If any AT RISK]
⚠️ Fix actions listed above. Track: github.com/anthropics/claude-code/issues/40524
```