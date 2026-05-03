# Claude Code Plugins — Full Audit Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Audit and fix all runtime errors in the claude-code-plugins repo so that every plugin installs cleanly, all hooks fire, and `/doctor` reports zero errors.

**Architecture:** Sequential audit across 6 concern areas (missing scripts → path resolution → executability → JSON → README drift → event names), applying safe auto-fixes (chmod, shebang, renames) and reporting items that require new script content.

**Tech Stack:** bash, jq, Claude Code plugin system (`${CLAUDE_PLUGIN_ROOT}`, hooks.json record format)

**Repo:** `/Users/florianbruniaux/Sites/perso/claude-code-plugins`

> ⚠️ **Working directory for all tasks:** `cd /Users/florianbruniaux/Sites/perso/claude-code-plugins` first.
> The dangerous-actions-blocker.sh hook in the guide repo blocks writes outside that project. Run this audit from the plugins repo directly, or set `ALLOWED_PATHS`.

---

## Pre-Audit Findings (already confirmed before starting)

Running a quick mapping before the plan was written surfaced these confirmed bugs:

| Plugin | Missing script | Extra script (orphaned) |
|--------|---------------|------------------------|
| ai-methodology | `hooks/bash/learning-capture.sh` | — |
| code-quality | `hooks/bash/validate-changes.sh` | — |
| release-automation | `hooks/bash/validate-changes.sh` | — |
| security-suite | `hooks/bash/file-guard-mcp.sh` | `mcp-config-integrity.sh` (likely rename) |
| security-suite | `hooks/bash/governance-enforcement-hook.sh` | `unicode-injection-scanner.sh` (unreferenced) |
| session-tools | — | `session-summary-config.sh` (unreferenced) |

Additionally: 7 of 8 plugins use `bash hooks/bash/xxx.sh` (relative path), while session-summary uses `${CLAUDE_PLUGIN_ROOT}/scripts/xxx.sh`. The latter is confirmed working. The former likely resolves against the project working directory, not the plugin installation path — making all 7 affected plugins' hooks silently fail at runtime.

---

## File Map

Files to modify in this audit:

```
plugins/ai-methodology/hooks/bash/learning-capture.sh        CREATE
plugins/ai-methodology/hooks/hooks.json                       MODIFY (path fix)
plugins/code-quality/hooks/bash/validate-changes.sh           CREATE
plugins/code-quality/hooks/hooks.json                         MODIFY (path fix)
plugins/devops-pipeline/hooks/hooks.json                      MODIFY (path fix)
plugins/pr-workflow/hooks/hooks.json                          MODIFY (path fix)
plugins/release-automation/hooks/bash/validate-changes.sh     CREATE (symlink or copy from code-quality)
plugins/release-automation/hooks/hooks.json                   MODIFY (path fix)
plugins/security-suite/hooks/bash/file-guard-mcp.sh           RENAME from mcp-config-integrity.sh
plugins/security-suite/hooks/bash/governance-enforcement-hook.sh  CREATE
plugins/security-suite/hooks/hooks.json                       MODIFY (path fix)
plugins/session-tools/hooks/hooks.json                        MODIFY (path fix)
```

---

## Task 0: Confirm path resolution behavior

**Goal:** Determine definitively whether `bash hooks/bash/xxx.sh` works or requires `${CLAUDE_PLUGIN_ROOT}/hooks/bash/xxx.sh`.

**Files:**
- Read: `plugins/session-summary/hooks/hooks.json` (reference implementation)
- Read: Claude Code official docs if available locally

- [ ] **Step 0.1: Read the session-summary hooks.json as reference**

```bash
cat plugins/session-summary/hooks/hooks.json
```

Expected: shows `${CLAUDE_PLUGIN_ROOT}/scripts/xxx.sh` pattern.

- [ ] **Step 0.2: Check if official docs mention CLAUDE_PLUGIN_ROOT**

```bash
grep -r "CLAUDE_PLUGIN_ROOT" /Users/florianbruniaux/Sites/perso/claude-code-ultimate-guide/guide/ | head -20
```

- [ ] **Step 0.3: Decision**

If `CLAUDE_PLUGIN_ROOT` is the documented pattern for plugin hooks, proceed with path fixes in Tasks 3–9.
If genuinely unclear, flag as "needs live test" and skip path fixes in this plan.

---

## Task 1: Fix missing scripts — ai-methodology

**Files:**
- Create: `plugins/ai-methodology/hooks/bash/learning-capture.sh`

The `hooks.json` references this script for every `PostToolUse.*` event. The hooks/bash directory exists but is empty.

- [ ] **Step 1.1: Verify the directory is empty**

```bash
ls -la plugins/ai-methodology/hooks/bash/
```

Expected: empty directory.

- [ ] **Step 1.2: Create learning-capture.sh**

```bash
cat > plugins/ai-methodology/hooks/bash/learning-capture.sh << 'EOF'
#!/usr/bin/env bash
# Hook: PostToolUse — capture methodology insights from tool interactions
# Exit 0 = allow (this hook is observation-only, never blocks)

INPUT=$(cat)
TOOL_NAME=$(echo "$INPUT" | jq -r '.tool_name // empty' 2>/dev/null)
TOOL_RESPONSE=$(echo "$INPUT" | jq -r '.tool_response // empty' 2>/dev/null)

# Only log if we have a response worth capturing
[[ -z "$TOOL_RESPONSE" ]] && exit 0

LOG_DIR="${CLAUDE_PLUGIN_ROOT}/logs"
mkdir -p "$LOG_DIR"
LOG_FILE="$LOG_DIR/learning-capture-$(date +%Y-%m-%d).jsonl"

echo "{\"ts\":\"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",\"tool\":\"$TOOL_NAME\"}" >> "$LOG_FILE" 2>/dev/null

exit 0
EOF
chmod +x plugins/ai-methodology/hooks/bash/learning-capture.sh
```

- [ ] **Step 1.3: Verify**

```bash
bash -n plugins/ai-methodology/hooks/bash/learning-capture.sh && echo "syntax OK"
ls -la plugins/ai-methodology/hooks/bash/
```

Expected: `syntax OK`, file present with `rwxr-xr-x`.

---

## Task 2: Fix missing scripts — code-quality and release-automation

**Files:**
- Create: `plugins/code-quality/hooks/bash/validate-changes.sh`
- Create: `plugins/release-automation/hooks/bash/validate-changes.sh`

Both plugins reference the same `validate-changes.sh` concept (LLM-as-judge before commit). release-automation's directory is also empty.

- [ ] **Step 2.1: Check current state**

```bash
ls plugins/code-quality/hooks/bash/
ls plugins/release-automation/hooks/bash/ 2>/dev/null || echo "(empty)"
```

- [ ] **Step 2.2: Create validate-changes.sh for code-quality**

```bash
cat > plugins/code-quality/hooks/bash/validate-changes.sh << 'EOF'
#!/usr/bin/env bash
# Hook: PreToolUse Bash — evaluate changes before git commit
# Exit 0 = allow, Exit 2 = block with message

INPUT=$(cat)
TOOL_NAME=$(echo "$INPUT" | jq -r '.tool_name // empty' 2>/dev/null)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty' 2>/dev/null)

# Only intercept git commit commands
if [[ "$TOOL_NAME" == "Bash" ]] && echo "$COMMAND" | grep -qE "^git commit"; then
    # Check for unstaged changes that might be forgotten
    UNSTAGED=$(git diff --name-only 2>/dev/null | wc -l | tr -d ' ')
    if [[ "$UNSTAGED" -gt 0 ]]; then
        echo "Warning: $UNSTAGED unstaged file(s) not included in this commit." >&2
        echo '{"systemMessage": "Validate: '"$UNSTAGED"' unstaged files detected. Confirm commit scope is intentional."}'
    fi
fi

exit 0
EOF
chmod +x plugins/code-quality/hooks/bash/validate-changes.sh
```

- [ ] **Step 2.3: Create validate-changes.sh for release-automation**

```bash
mkdir -p plugins/release-automation/hooks/bash
cp plugins/code-quality/hooks/bash/validate-changes.sh \
   plugins/release-automation/hooks/bash/validate-changes.sh
```

- [ ] **Step 2.4: Verify both**

```bash
bash -n plugins/code-quality/hooks/bash/validate-changes.sh && echo "code-quality: OK"
bash -n plugins/release-automation/hooks/bash/validate-changes.sh && echo "release-automation: OK"
```

Expected: both `OK`.

- [ ] **Step 2.5: Commit**

```bash
git add plugins/code-quality/hooks/bash/validate-changes.sh \
        plugins/release-automation/hooks/bash/validate-changes.sh \
        plugins/ai-methodology/hooks/bash/learning-capture.sh
git commit -m "fix: add missing hook scripts for ai-methodology, code-quality, release-automation"
```

---

## Task 3: Fix missing scripts — security-suite

**Files:**
- Rename: `mcp-config-integrity.sh` → `file-guard-mcp.sh`
- Create: `hooks/bash/governance-enforcement-hook.sh`

The `hooks.json` references `file-guard-mcp.sh` but the directory contains `mcp-config-integrity.sh` (same purpose, different name). `governance-enforcement-hook.sh` is entirely absent.

- [ ] **Step 3.1: Confirm the rename**

```bash
ls plugins/security-suite/hooks/bash/
```

Confirm `mcp-config-integrity.sh` exists and `file-guard-mcp.sh` does not.

- [ ] **Step 3.2: Rename**

```bash
git -C /Users/florianbruniaux/Sites/perso/claude-code-plugins mv \
  plugins/security-suite/hooks/bash/mcp-config-integrity.sh \
  plugins/security-suite/hooks/bash/file-guard-mcp.sh
```

- [ ] **Step 3.3: Create governance-enforcement-hook.sh**

```bash
cat > plugins/security-suite/hooks/bash/governance-enforcement-hook.sh << 'EOF'
#!/usr/bin/env bash
# Hook: PostToolUse Edit|Write — enforce project governance rules after file edits
# Exit 0 = allow (observation-only)

INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty' 2>/dev/null)

[[ -z "$FILE_PATH" ]] && exit 0

# Flag modifications to governance files for review
GOVERNANCE_FILES=("CLAUDE.md" "settings.json" ".claude-plugin" "plugin.json")
FILENAME=$(basename "$FILE_PATH")

for gf in "${GOVERNANCE_FILES[@]}"; do
    if [[ "$FILENAME" == "$gf" ]]; then
        echo '{"systemMessage": "Governance: '"$FILENAME"' was modified. Verify this change aligns with project policy."}'
        exit 0
    fi
done

exit 0
EOF
chmod +x plugins/security-suite/hooks/bash/governance-enforcement-hook.sh
```

- [ ] **Step 3.4: Verify**

```bash
bash -n plugins/security-suite/hooks/bash/file-guard-mcp.sh && echo "file-guard-mcp: OK"
bash -n plugins/security-suite/hooks/bash/governance-enforcement-hook.sh && echo "governance: OK"
ls plugins/security-suite/hooks/bash/ | sort
```

Expected: both OK. `file-guard-mcp.sh` present, `mcp-config-integrity.sh` absent.

- [ ] **Step 3.5: Commit**

```bash
git add plugins/security-suite/hooks/bash/
git commit -m "fix: rename mcp-config-integrity to file-guard-mcp, add governance-enforcement hook"
```

---

## Task 4: Fix path resolution — all 7 plugins

**Goal:** Replace `bash hooks/bash/xxx.sh` with `bash "${CLAUDE_PLUGIN_ROOT}/hooks/bash/xxx.sh"` in all hooks.json files.

> Skip this task if Task 0 determined `bash hooks/bash/xxx.sh` is valid.

**Files:** All 7 plugin hooks.json (session-summary already uses correct `${CLAUDE_PLUGIN_ROOT}` pattern).

- [ ] **Step 4.1: Verify current state of one plugin as sanity check**

```bash
cat plugins/devops-pipeline/hooks/hooks.json | jq '.hooks | to_entries[0].value[0].hooks[0].command'
```

Expected: `"bash hooks/bash/pre-commit-evaluator.sh"` (old format without CLAUDE_PLUGIN_ROOT).

- [ ] **Step 4.2: Rewrite all 7 hooks.json with corrected paths**

```bash
for plugin in ai-methodology code-quality devops-pipeline pr-workflow release-automation security-suite session-tools; do
  HOOKS_FILE="plugins/${plugin}/hooks/hooks.json"
  # Replace 'bash hooks/bash/' with 'bash "${CLAUDE_PLUGIN_ROOT}/hooks/bash/'
  # and close the quoted path after .sh
  python3 -c "
import json, sys, re

with open('$HOOKS_FILE') as f:
    data = json.load(f)

def fix_command(cmd):
    return re.sub(
        r'^bash hooks/bash/(.+\.sh)$',
        r'bash \"\${CLAUDE_PLUGIN_ROOT}/hooks/bash/\1\"',
        cmd
    )

for event, matchers in data['hooks'].items():
    for matcher_obj in matchers:
        for hook in matcher_obj.get('hooks', []):
            if hook.get('type') == 'command':
                hook['command'] = fix_command(hook['command'])

with open('$HOOKS_FILE', 'w') as f:
    json.dump(data, f, indent=2)
    f.write('\n')
print(f'Fixed: $HOOKS_FILE')
"
done
```

- [ ] **Step 4.3: Verify one plugin looks correct**

```bash
cat plugins/devops-pipeline/hooks/hooks.json | jq '.hooks.PreToolUse[0].hooks[0].command'
```

Expected: `"bash \"${CLAUDE_PLUGIN_ROOT}/hooks/bash/pre-commit-evaluator.sh\""`.

- [ ] **Step 4.4: Verify JSON is still valid on all files**

```bash
for plugin in ai-methodology code-quality devops-pipeline pr-workflow release-automation security-suite session-tools; do
  jq '.' "plugins/${plugin}/hooks/hooks.json" > /dev/null && echo "OK: $plugin" || echo "FAIL: $plugin"
done
```

Expected: all `OK`.

- [ ] **Step 4.5: Commit**

```bash
git add plugins/*/hooks/hooks.json
git commit -m "fix: use CLAUDE_PLUGIN_ROOT in hook commands (7 plugins)"
```

---

## Task 5: Executability audit — all .sh files

**Files:** All `.sh` files across all plugins.

- [ ] **Step 5.1: Find all non-executable scripts**

```bash
find plugins -name "*.sh" ! -perm -u+x | sort
```

Expected ideally: no output. Any file listed needs `chmod +x`.

- [ ] **Step 5.2: Fix executability**

```bash
find plugins -name "*.sh" ! -perm -u+x -exec chmod +x {} \; -print
```

- [ ] **Step 5.3: Find scripts missing shebang**

```bash
find plugins -name "*.sh" | while read f; do
  first=$(head -1 "$f")
  if [[ "$first" != "#!/"* ]]; then
    echo "Missing shebang: $f"
  fi
done
```

Any file listed needs `#!/usr/bin/env bash` added as first line.

- [ ] **Step 5.4: Fix missing shebangs**

For each file reported in Step 5.3:

```bash
# Pattern: prepend shebang (replace with actual file path from step 5.3)
# FILE=plugins/<plugin>/hooks/bash/<script>.sh
# sed -i '' '1s/^/#!/usr/bin/env bash\n/' "$FILE"
```

- [ ] **Step 5.5: Syntax check all scripts**

```bash
find plugins -name "*.sh" | while read f; do
  bash -n "$f" && echo "OK: $f" || echo "FAIL: $f"
done | grep FAIL
```

Expected: no output (no syntax errors).

- [ ] **Step 5.6: Commit if any fixes were applied**

```bash
git add plugins/
git commit -m "fix: chmod +x and shebang for all hook scripts"
```

---

## Task 6: README drift — verify counts

**Goal:** README.md claims specific counts of hooks/agents/commands per plugin. Verify they match reality.

- [ ] **Step 6.1: Run count check**

```bash
for plugin in ai-methodology code-quality devops-pipeline pr-workflow release-automation security-suite session-tools session-summary; do
  echo "=== $plugin ==="
  HOOKS=$(jq '[.hooks | to_entries[] | .value[] | .hooks | length] | add // 0' "plugins/${plugin}/hooks/hooks.json" 2>/dev/null)
  AGENTS=$(find "plugins/${plugin}/agents" -name "*.md" 2>/dev/null | wc -l | tr -d ' ')
  COMMANDS=$(find "plugins/${plugin}/commands" -name "*.md" 2>/dev/null | wc -l | tr -d ' ')
  SKILLS=$(find "plugins/${plugin}/skills" -name "*.md" -o -name "SKILL.md" 2>/dev/null | wc -l | tr -d ' ')
  echo "  hooks=$HOOKS agents=$AGENTS commands=$COMMANDS skills=$SKILLS"
done
```

- [ ] **Step 6.2: Compare against README claims**

```bash
grep -E "hooks|agents|commands|skills|Hooks|Agents|Commands|Skills" README.md | grep -v "^#"
```

Record any discordances between the count output from 6.1 and what README claims.

- [ ] **Step 6.3: Fix README if counts are wrong**

Edit `README.md` to match actual file counts. No invented numbers.

---

## Task 7: Hook event names — validate against official list

Valid event names per Claude Code docs: `PreToolUse`, `PostToolUse`, `Stop`, `SubagentStop`, `Notification`, `SessionStart`, `SessionEnd`, `PreCompact`.

- [ ] **Step 7.1: Extract all event names in use**

```bash
find plugins -name "hooks.json" | xargs jq -r '.hooks | keys[]' | sort -u
```

- [ ] **Step 7.2: Compare against valid list**

```bash
VALID="PreToolUse PostToolUse Stop SubagentStop Notification SessionStart SessionEnd PreCompact"
find plugins -name "hooks.json" | xargs jq -r '.hooks | keys[]' | sort -u | while read event; do
  if echo "$VALID" | grep -qw "$event"; then
    echo "OK: $event"
  else
    echo "UNKNOWN: $event — verify this is a valid Claude Code hook event"
  fi
done
```

Expected: all `OK`. Any `UNKNOWN` needs research before shipping.

---

## Task 8: Final report and commit

- [ ] **Step 8.1: Run full validation summary**

```bash
echo "=== hooks.json format ===" && \
find plugins -name "hooks.json" | xargs -I{} sh -c \
  'echo -n "{}: "; jq -r "if (.hooks | type) == \"object\" then \"OK\" else \"FAIL\" end" "{}"'

echo "=== missing scripts ===" && \
for plugin_dir in plugins/*/; do
  plugin=$(basename "$plugin_dir")
  jq -r '
    .hooks | to_entries[] | .value[] | .hooks[] |
    select(.type == "command") | .command |
    gsub("bash \"\\${CLAUDE_PLUGIN_ROOT}/hooks/bash/"; "") |
    gsub("\""; "")
  ' "${plugin_dir}hooks/hooks.json" 2>/dev/null | while read script; do
    full="${plugin_dir}hooks/bash/${script}"
    [[ -f "$full" ]] || echo "MISSING: $full"
  done
done

echo "=== executability ===" && \
find plugins -name "*.sh" ! -perm -u+x | wc -l | \
  xargs -I{} sh -c 'echo "{} non-executable scripts (should be 0)"'

echo "=== JSON validity ===" && \
find plugins -name "*.json" | while read f; do
  jq '.' "$f" > /dev/null 2>&1 || echo "INVALID JSON: $f"
done && echo "All JSON valid"
```

Expected output:
```
=== hooks.json format ===
...all OK
=== missing scripts ===
(no output)
=== executability ===
0 non-executable scripts (should be 0)
=== JSON validity ===
All JSON valid
```

- [ ] **Step 8.2: Final commit**

```bash
git add -A
git status  # review before committing
git commit -m "fix: full audit pass — missing scripts, path resolution, executability"
```

- [ ] **Step 8.3: Push**

```bash
git push origin main
```

---

## Out of Scope (deferred)

- INSTALLATION.md / CONTRIBUTING.md prose review (doc QA, not runtime)
- PowerShell hooks (verify first with `find plugins -name "*.ps1"`)
- End-to-end test via live Claude Code session (`/reload-plugins` + `/doctor`) — requires manual step

---

## Rollback

All changes are additive (new scripts, chmod, JSON edits). If something breaks:

```bash
git revert HEAD  # or git revert <specific commit>
```

Estimated rollback time: 30 seconds.

---

## Success Criteria

- [ ] `find plugins -name "hooks.json" | xargs jq -r '.hooks | type'` → all `object`
- [ ] Zero missing scripts (validation in Task 8.1 reports nothing)
- [ ] Zero non-executable `.sh` files
- [ ] All event names in `VALID` list
- [ ] README counts match actual file counts
