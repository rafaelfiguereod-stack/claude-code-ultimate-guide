---
title: "GitHub Actions Workflows for Claude Code"
description: "Ready-to-use CI/CD workflows integrating Claude Code into GitHub Actions"
tags: [ci-cd, devops, template, workflows]
---

# GitHub Actions Workflows for Claude Code

Ready-to-use GitHub Actions workflows that integrate Claude Code into your CI/CD pipeline.

## Prerequisites

1. **Install the Claude GitHub App** on your org/repo (required for Actions to comment on PRs/issues)
2. **Add API Key Secret**: In your repo, go to Settings → Secrets and variables → Actions → New repository secret
   - Name: `ANTHROPIC_API_KEY`
   - Value: Your Anthropic API key from [console.anthropic.com](https://console.anthropic.com)
3. **Copy Workflows**: Place these `.yml` files in `.github/workflows/` directory
4. **Test**: Open a test PR or issue to see them run

## Available Workflows

### 1. Code Review — Prompt-Based (`claude-code-review.yml`) ⭐ Recommended

**Robust pattern** with externalized prompt, anti-hallucination protocol, and `/claude-review` on-demand trigger.

The review logic lives in `.github/prompts/code-review.md`, so you can iterate on criteria without touching the workflow YAML. The prompt enforces a verification step before every finding — Claude must confirm an issue with `Read`/`Grep` before reporting it.

**Features:**
- Triggers on PR open/sync/ready **and** `/claude-review` comment
- Externalized prompt: edit `code-review.md` to tune criteria for your stack
- Anti-hallucination protocol: no invented line numbers or unverified claims
- Structured output: `🔴 MUST FIX` / `🟡 SHOULD FIX` / `🟢 CAN SKIP` table + inline comments
- Read-only `allowed_tools` (no write access to repo)
- OAuth token support (no API key needed if Claude GitHub App is installed)

**Setup:**
```bash
# Copy both files into your repo
cp examples/github-actions/claude-code-review.yml .github/workflows/
mkdir -p .github/prompts
cp examples/github-actions/prompts/code-review.md .github/prompts/

# Add secret: CLAUDE_CODE_OAUTH_TOKEN (or ANTHROPIC_API_KEY)
# Install Claude GitHub App: https://github.com/apps/claude
```

**Customization:**
Edit `.github/prompts/code-review.md` to add your stack conventions:
```markdown
## Stack Context
- TypeScript strict mode, no `any`
- React Server Components — no `useEffect` for data fetching
- All DB writes must go through the repository layer
- New API routes require integration tests
```

---

### 2. Auto PR Review (`claude-pr-auto-review.yml`)

**Enhanced version** with comprehensive review criteria and smart filtering.

Creates a structured review with inline comments as soon as a PR opens or updates.

**Features:**
- Automatic code review on PR open/update
- 8 focus areas: Correctness, Security, Performance, Readability, Maintainability, Testing, Best Practices, Breaking Changes
- Priority-based feedback: 🔴 Critical, 🟡 Important, 🟢 Suggestion, 💡 Tip
- Smart file filtering (skips build artifacts, lock files)
- Skips draft PRs to save costs
- Summary review with risk assessment
- Error handling and fallback notifications
- Inline comments on specific lines

**Usage:**
```bash
# Copy the workflow file
cp examples/github-actions/claude-pr-auto-review.yml .github/workflows/

# Open a PR - Claude will automatically review it
```

**Customization:**
Add project-specific context by uncommenting the `append_system_prompt` section:
```yaml
append_system_prompt: |
  Project conventions:
  - Use TypeScript strict mode
  - Follow functional programming patterns
  - All functions must have JSDoc comments
  - Test coverage must be >80%
```

---

### 2. Security Review (`claude-security-review.yml`)

Runs a focused security scan and comments findings directly on the PR.

**Features:**
- Security-focused analysis on every PR
- Identifies potential vulnerabilities
- OWASP Top 10 considerations
- Posts findings as PR comments

**Configuration:**
```yaml
# Optional parameters in the workflow file:
exclude-directories: "docs,examples"    # Skip certain directories
claudecode-timeout: "20"                # Timeout in minutes
claude-model: "claude-3-5-sonnet-20240620"  # Model to use
```

**Usage:**
```bash
# Copy the workflow file
cp examples/github-actions/claude-security-review.yml .github/workflows/

# Every PR will be automatically scanned for security issues
```

---

### 3. Issue Triage (`claude-issue-triage.yml`)

When a new issue opens, Claude proposes labels/severity and posts a tidy triage comment.

**Features:**
- Automatic issue classification
- Label suggestions
- Severity assessment (low, medium, high, critical)
- Duplicate detection
- Markdown triage comment

**Auto-apply Labels (Optional):**
To automatically apply suggested labels, edit the workflow file and change:
```yaml
- name: Apply labels (optional)
  if: ${{ false }}  # Change to true to auto-apply labels
```

**Usage:**
```bash
# Copy the workflow file
cp examples/github-actions/claude-issue-triage.yml .github/workflows/

# Open a new issue - Claude will automatically triage it
```

---

## Customization

### Model Selection
Set `CLAUDE_MODEL` or `claude-model` parameter in workflows:
```yaml
env:
  CLAUDE_MODEL: claude-3-5-sonnet-20240620
```

### Permissions
Each workflow declares minimal required permissions:
- `pull-requests: write` for PR reviews
- `issues: write` for issue triage
- `contents: read` for reading repository content

Adjust only if your organization requires stricter policies.

### Scope Filtering
Use `paths:` filters to limit when workflows run:
```yaml
on:
  pull_request:
    paths:
      - 'src/**'
      - '!docs/**'
```

## Troubleshooting

**No comments appear on PRs:**
- Verify the Claude GitHub App is installed
- Check workflow has `pull-requests: write` permission

**403 when applying labels:**
- Ensure the job has `issues: write` permission
- Verify `GITHUB_TOKEN` has access to this repo

**Anthropic API errors:**
- Confirm `ANTHROPIC_API_KEY` is set at repository level
- Check the key is not expired

**YAML syntax errors:**
- Validate spacing: two spaces per nesting level, no tabs
- Use a YAML validator: [yamllint.com](https://www.yamllint.com/)

## Advanced Usage

### Combining Workflows
Run multiple workflows together for comprehensive automation:
- PR Review + Security Review on every PR
- Issue Triage + Auto-labeling for new issues

### Custom Prompts
Edit the `direct_prompt` section in workflows to customize Claude's focus:
```yaml
direct_prompt: |
  Review this PR focusing on:
  1. TypeScript type safety
  2. React performance patterns
  3. Accessibility compliance
  4. Test coverage
```

### Integration with Other Actions
Combine with existing workflows:
```yaml
jobs:
  tests:
    runs-on: ubuntu-latest
    steps:
      - name: Run tests
        run: npm test

  claude-review:
    needs: tests  # Run after tests pass
    runs-on: ubuntu-latest
    steps:
      - uses: anthropics/claude-code-action@main
        # ...
```

## Cost Considerations

These workflows consume Anthropic API credits:
- **PR Review**: ~$0.10-$0.50 per review (depending on diff size)
- **Security Review**: ~$0.20-$0.80 per scan
- **Issue Triage**: ~$0.05-$0.20 per issue

**Tips to reduce costs:**
- Use `paths:` filters to skip docs/config changes
- Set conditions: `if: github.event.pull_request.draft == false`
- Review logs and adjust model selection

## Examples in This Directory

```
examples/github-actions/
├── README.md                        # This file
├── claude-code-review.yml           # Prompt-based review (recommended)
├── claude-pr-auto-review.yml        # Inline prompt auto-review
├── claude-security-review.yml       # Security scanning workflow
├── claude-issue-triage.yml          # Issue triage workflow
└── prompts/
    └── code-review.md               # Externalized review prompt (copy to .github/prompts/)
```

## Resources

- [Claude Code Documentation](https://claude.ai/code)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Anthropic API Documentation](https://docs.anthropic.com/)
- [Claude GitHub App](https://github.com/apps/claude)

## License

These workflows are provided as examples. Adapt them to your needs.
