---
name: oss-contributor
description: Autonomous open source contribution workflow — discover good first issues, implement fixes, submit PRs, and log outcomes.
version: 1.0.0
author: BlutAgent
license: MIT
metadata:
  hermes:
    tags: [github, open-source, contribution, automation, pr-workflow]
    related_skills: [github-auth, github-issues, github-pr-workflow]
---

# OSS Contributor

Autonomous open source contribution workflow. Scans GitHub for good first issues in TypeScript or Python repos, implements fixes, submits PRs, and logs outcomes.

## Prerequisites

- GitHub authentication configured (see `github-auth` skill)
- Git configured with user.name and user.email

## Workflow

### Step 1: Check Authentication
### Step 1: Check Authentication

Run auth detection before proceeding:

```bash
# Check for gh CLI - must be in PATH and authenticated
if command -v gh &>/dev/null; then
  if gh auth status &>/dev/null 2>&1; then
    echo "AUTH=gh"
    # Note: gh auth login should include --scopes repo,public_repo for PR creation
    gh auth setup-git 2>/dev/null || true
  else
    echo "BLOCKED: gh CLI found but not authenticated"
    echo "Run: gh auth login --scopes repo,public_repo"
    exit 1
  fi
else
  # Check for git credentials
  if grep -q "github.com" ~/.git-credentials 2>/dev/null; then
    echo "AUTH=git"
  else
    echo "BLOCKED: No GitHub authentication configured"
    echo "Options:"
    echo "  1. Install gh CLI: brew install gh (macOS) or see https://cli.github.com"
    echo "  2. Set up SSH key: ssh-keygen -t ed25519 and add to GitHub"
    echo "  3. Configure git credentials: gh auth setup-git"
    echo "See github-auth skill for setup"
    exit 1
  fi
fi

# Verify git identity
if ! git config --global user.name &>/dev/null; then
  echo "BLOCKED: Git user.name not configured"
  exit 1
fi
```

### Step 1.5: Verify Push Capability (CRITICAL)

**Do this BEFORE cloning and implementing.** Auth check passing does NOT mean you can push.

```bash
# Create a test branch to verify push works
TEST_BRANCH="test/auth-verify-$(date +%s)"
git checkout -b "$TEST_BRANCH" 2>/dev/null || {
  echo "BLOCKED: Cannot create branch (no repo context)"
  echo "Push capability will be tested after cloning target repo"
}

# If we have a repo, try to push
if git rev-parse --git-dir &>/dev/null; then
  if git push -u origin "$TEST_BRANCH" &>/dev/null 2>&1; then
    echo "PUSH_OK: Can push to GitHub"
    git checkout - 2>/dev/null
    git branch -D "$TEST_BRANCH" 2>/dev/null
  else
    echo "BLOCKED: Auth check passed but push failed"
    echo "Fix options:"
    echo "  1. gh auth login --force"
    echo "  2. Set up SSH: ssh-keygen -t ed25519 && add to GitHub"
    echo "  3. Configure credentials: git credential fill"
    exit 1
  fi
fi
```

### Step 2: Discover Issues

Search for good first issues:

```bash
# TypeScript issues
curl -s "https://api.github.com/search/issues?q=label:\"good+first+issue\"+language:TypeScript+state:open&sort=updated&order=desc&per_page=15" > /tmp/ts_issues.json

# Python issues
curl -s "https://api.github.com/search/issues?q=label:\"good+first+issue\"+language:Python+state:open&sort=updated&order=desc&per_page=15" > /tmp/py_issues.json
```

### Step 3: Score and Select

Evaluate candidates using these criteria:

| Criterion | Points | Check |
|-----------|--------|-------|
| Stars 100-5000 | +30 | Repo metadata |
| Description >100 chars | +20 | Issue body length |
| Updated <30 days | +20 | updated_at field |
| "good first issue" label | +15 | Labels array |
| "bug" or "documentation" label | +15 | Labels array |

Select the highest-scored issue with clear, actionable description.

### Step 4: Implement Fix

```bash
# Clone repository (upstream)
git clone <repo-url> /tmp/contribution-work
cd /tmp/contribution-work

# Create feature branch
git checkout -b "fix/issue-<number>"

# Make changes using file tools (read_file, patch, write_file)
# Focus on minimal, targeted fixes

# Commit with conventional commit message
git add .
git commit -m "fix: <description>

Closes #<issue-number>"
```

### Step 4.5: Fork and Push (for repos without write access)

Most OSS contributions require forking. Do this AFTER implementing but BEFORE pushing:

```bash
# Fork the repo (creates blut-agent/repo-name)
gh repo fork --clone=false

# Add fork as remote (or replace origin)
git remote add fork https://github.com/<your-username>/<repo-name>.git
# OR
git remote set-url origin https://github.com/<your-username>/<repo-name>.git

# Push to YOUR fork
git push -u origin HEAD
# OR
git push -u fork HEAD
```

### Step 5: Create Pull Request

```bash
# Using gh CLI (from fork to upstream)
gh pr create \
  --repo <upstream-owner>/<repo-name> \
  --base <upstream-branch> \
  --head <your-username>:<your-branch> \
  --title "fix: <description>" \
  --body "## Summary
<What this fixes>

## Related
Closes #<issue-number>"
```

**If PR creation fails with "Resource not accessible by personal access token":**
- Your gh token lacks `repo` or `public_repo` scope
- **Fallback:** Create PR manually via GitHub UI:
  1. Go to: `https://github.com/<upstream-owner>/<repo-name>/compare/<upstream-branch>...<your-username>:<your-branch>`
  2. Click "Create pull request"
  3. Fill in title and body
  4. Submit

### Step 6: Log Outcome

Append entry to `~/.hermes/wiki/contributions.md`:

```markdown
## YYYY-MM-DD - <Repo> #<Issue>

**Status:** <submitted|blocked|merged|closed>
**Repo:** <owner/repo>
**Issue:** #<number> - <title>
**PR:** <url or "N/A">
**Summary:** <what was done>
**Outcome:** <pending review|merged|needs work>
**Lessons:** <what was learned>
```

## Cron Integration

```python
cronjob(
  action='create',
  name='weekly-oss-contrib',
  schedule='0 10 * * 1',
  prompt='Run oss-contributor: find good first issue in TypeScript/Python repo, implement fix, submit PR, log to contributions.md',
  deliver='origin'
)
```

## Pitfalls

| Issue | Solution |
|-------|----------|
| No auth configured | Check first, fail fast with clear message |
| gh CLI not in PATH | Auth check may pass but gh might not be executable. Verify with `command -v gh` before relying on it. If unavailable, set up SSH key or git credentials as fallback. |
| SSH key not configured | Terminal sessions may lack SSH access. Set up SSH key with `ssh-keygen -t ed25519` and add to GitHub, or use `gh auth setup-git` to configure credential helper. |
| Auth check passes but push fails | The auth check only verifies credentials exist, not that they work. **Always test push capability early** — create a test branch and attempt push before implementing the full fix. See "Step 0: Verify Push Capability" below. |
| Cannot push to upstream repo | You don't have write access. **Fork first:** `gh repo fork`, then push to your fork and create PR from fork:branch to upstream:base. |
| PR creation fails with "Resource not accessible by personal access token" | Your gh token lacks `repo` or `public_repo` scope. **Fallback:** Create PR manually via GitHub UI at `https://github.com/<upstream-owner>/<repo-name>/compare/<base-branch>...<your-username>:<your-branch>`. Click "Create pull request", fill in title/body, submit. |
| Terminal execution tools unavailable | If you cannot run gh/git commands (session limitations, Docker issues), send complete copy-paste instructions to the user via send_message. Include: (1) the exact gh pr create command, (2) the GitHub UI fallback URL, (3) expected output. User can then execute manually. |
| GITHUB_TOKEN env var overrides keyring | gh CLI prioritizes GITHUB_TOKEN environment variable over keyring tokens. If PR creation fails with "Resource not accessible", the env var token may have limited scopes. **Fix:** `unset GITHUB_TOKEN` then retry, or ensure env var token has `repo` scope. |
| Issue already claimed | Check for existing PRs, skip if found |
| Unclear requirements | Comment on issue asking for clarification |
| Tests fail locally | Run test suite before pushing |
| CI fails after push | Monitor checks, fix in follow-up commit |
| No maintainer response >2 weeks | Log and move to next opportunity |

## Recovery — Terminal Lost Mid-Session

If terminal access is lost after push but before PR creation:

1. **Log immediately** — Add entry to `~/.hermes/wiki/contributions.md` as `IN_PROGRESS`:
   ```markdown
   ## YYYY-MM-DD - <Repo> #<Issue>
   **Status:** IN_PROGRESS (pushed, PR pending)
   **Branch:** <your-username>:<branch-name>
   **Upstream:** <owner>/<repo>
   **Notes:** Terminal lost after push. PR creation pending.
   ```

2. **On next session with terminal access**, restore and create PR:
   ```bash
   # Clone your fork (if not already present)
   git clone https://github.com/<your-username>/<repo-name>.git
   cd <repo-name>
   
   # Or if repo exists, fetch and checkout the branch
   git fetch origin <branch-name>
   git checkout <branch-name>
   
   # Create PR
   gh pr create \
     --repo <upstream-owner>/<repo-name> \
     --base <upstream-branch> \
     --head $(gh api user --jq .login):<branch-name> \
     --title "fix: <description>" \
     --body "Closes #<issue-number>"
   ```

3. **Never leave a pushed branch without a PR** — always complete the loop. An orphaned branch is wasted work and clutters your fork.

4. **Alternative: GitHub UI** — If terminal remains unavailable, create PR manually:
   - Go to: `https://github.com/<upstream-owner>/<repo-name>/compare/<base>...<your-username>:<branch>`
   - Click "Create pull request"
   - Fill in title/body and submit

## Target Profile

Based on user preferences:

- **Languages:** TypeScript (primary), Python
- **Repo size:** 100-5000 stars
- **Domains:** CLI tools, developer tooling, AI/ML adjacent
- **PR style:** Small-scoped changes, always link issue

## Remember

```
1. Auth with scopes — gh auth login --scopes repo,public_repo (needed for PR creation)
2. TEST PUSH BEFORE IMPLEMENTING — auth passing ≠ push working. Create test branch and push it.
3. No write access? Fork first — gh repo fork, then push to your fork
4. PR creation blocked? Use GitHub UI: github.com/OWNER/REPO/compare/base...user:branch
5. Terminal tools unavailable? Send complete copy-paste instructions to user via send_message
6. Terminal lost mid-session? Log branch/repo to contributions.md, complete PR on next session
7. Never leave pushed branch without PR — always complete the loop
8. Score issues — stars, recency, description quality matter
9. Minimal fixes — small PRs merge faster
10. Log everything — both successes and blockers
11. Move on if stuck — many opportunities exist
```
