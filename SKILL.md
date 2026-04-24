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

### Step 2.5: Verify Issue Is Unclaimed (CRITICAL)

**Before investing time, check if someone is already working on the issue:**

```bash
# Check for existing PRs referencing the issue
gh search prs --repo <owner>/<repo> --state=open "<issue-number>"

# Check issue comments for "I'd like to work on this" type claims
gh api repos/<owner>/<repo>/issues/<number>/comments --jq '.[].body'
```

Skip if a PR exists or a maintainer has already assigned someone.

### Step 2.6: Check CONTRIBUTING Guide and AI Policy (CRITICAL)

**Before cloning or implementing, verify the repo welcomes external contributions and permits AI assistance:**

```bash
# Fetch CONTRIBUTING.md (case-insensitive) from default branch
OWNER=<owner>
REPO=<repo>
BRANCH=$(gh api repos/$OWNER/$REPO --jq '.default_branch')

CONTRIBUTING_URL="https://raw.githubusercontent.com/$OWNER/$REPO/$BRANCH/CONTRIBUTING.md"
CONTRIBUTING=$(curl -sL "$CONTRIBUTING_URL")

if [ -z "$CONTRIBUTING" ] || [ "$CONTRIBUTING" = "404: Not Found" ]; then
  # Try lowercase variants
  CONTRIBUTING=$(curl -sL "https://raw.githubusercontent.com/$OWNER/$REPO/$BRANCH/CONTRIBUTING")
fi

if [ -n "$CONTRIBUTING" ] && [ "$CONTRIBUTING" != "404: Not Found" ]; then
  echo "CONTRIBUTING found ($(echo "$CONTRIBUTING" | wc -l) lines)"
  echo "$CONTRIBUTING"

  # Check for explicit AI prohibition
  AI_BAN=$(echo "$CONTRIBUTING" | grep -iE 'no ai|no.*artificial intelligence|ai.*not allowed|prohibit.*ai|ban.*ai|do not use ai|no generative ai|no llm|no copilot' || true)
  if [ -n "$AI_BAN" ]; then
    echo "BLOCKED: CONTRIBUTING.md contains AI prohibition:"
    echo "$AI_BAN"
    echo "Skipping this repo entirely."
    exit 1
  fi

  # Check for explicit AI allowance
  AI_ALLOW=$(echo "$CONTRIBUTING" | grep -iE 'ai allowed|ai.*permit|use of ai|generative ai|copilot|llm.*permit' || true)
  if [ -n "$AI_ALLOW" ]; then
    echo "AI policy: explicitly allowed or acknowledged"
  fi
else
  echo "No CONTRIBUTING.md found — proceed with caution"
fi

# Also scan README for AI policy
README=$(curl -sL "https://raw.githubusercontent.com/$OWNER/$REPO/$BRANCH/README.md" | head -n 100 || true)
README_AI_BAN=$(echo "$README" | grep -iE 'no ai|no.*artificial intelligence|ai.*not allowed|prohibit.*ai|ban.*ai' || true)
if [ -n "$README_AI_BAN" ]; then
  echo "BLOCKED: README contains AI prohibition"
  echo "$README_AI_BAN"
  exit 1
fi
```

**Decision rules:**
- If CONTRIBUTING explicitly bans AI → **Skip repo completely**
- If CONTRIBUTING explicitly allows AI → **Proceed**
- If no CONTRIBUTING exists → **Proceed with extra care; write a conservative, well-explained PR**
- If CONTRIBUTING exists but is silent on AI → **Proceed; treat it like any external contributor**

Also scan for contribution barriers:
- CLA required? (look for "Contributor License Agreement" / "CLA")
- Specific commit message formats required?
- Issue assignment required before PR?
- Branch naming conventions?

Log any notable policies in the contribution log.

### Step 3: Score and Select

Evaluate candidates using these criteria:

| Criterion | Points | Check |
|-----------|--------|-------|
| Stars 100-5000 | +30 | Repo metadata (fetch separately — search API doesn't include stars) |
| Description >100 chars | +20 | Issue body length |
| Updated <30 days | +20 | updated_at field |
| "good first issue" label | +15 | Labels array |
| "bug" or "documentation" label | +15 | Labels array |

Select the highest-scored issue with clear, actionable description.

**Note:** The GitHub search API returns `repository_url` but NOT star counts. You must fetch each unique repo's metadata separately:
```bash
gh api repos/<owner>/<repo> --jq '{stargazers_count, language, description}'
```

### Step 4: Implement Fix

**Working directory:** Use `~/oss-contribution/<repo-name>` instead of `/tmp/`. The `/tmp` directory is unreliable across terminal calls (paths get corrupted, `/private/tmp` resolution issues on macOS).

```bash
# Create stable working directory
mkdir -p ~/oss-contribution && cd ~/oss-contribution

# Clone repository (upstream)
git clone <repo-url> <repo-name>
cd <repo-name>

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

**Before drafting the PR, check if the repository provides a PR template:**

```bash
OWNER=<owner>
REPO=<repo>
BRANCH=$(gh api repos/$OWNER/$REPO --jq '.default_branch')

# Check common template locations
TEMPLATE_PATHS=(
  ".github/PULL_REQUEST_TEMPLATE.md"
  ".github/pull_request_template.md"
  "PULL_REQUEST_TEMPLATE.md"
  "docs/PULL_REQUEST_TEMPLATE.md"
)

PR_BODY=""
for path in "${TEMPLATE_PATHS[@]}"; do
  TEMPLATE=$(curl -sL "https://raw.githubusercontent.com/$OWNER/$REPO/$BRANCH/$path" || true)
  if [ -n "$TEMPLATE" ] && [ "$TEMPLATE" != "404: Not Found" ] && [ "$TEMPLATE" != "Not Found" ]; then
    echo "Found PR template at: $path"
    PR_BODY="$TEMPLATE"
    break
  fi
done

# If no template found, use the default body
if [ -z "$PR_BODY" ]; then
  PR_BODY="## Summary
<What this fixes>

## Related
Closes #<issue-number>"
  echo "No PR template found — using default body"
else
  echo "Using repository PR template"
fi
```

**Fill the template carefully.** Replace placeholders (e.g., `<!-- Description -->`) with actual content:
- Summarize the change in plain language
- Reference the issue: `Closes #<issue-number>` or `Fixes #<issue-number>`
- Add test notes if tests couldn't be run locally (e.g., Python version mismatch)
- Mention any AI assistance only if the CONTRIBUTING guide explicitly asks for disclosure

**Create the PR:**

```bash
# Using gh CLI (from fork to upstream)
gh pr create \
  --repo <upstream-owner>/<repo-name> \
  --base <upstream-branch> \
  --head <your-username>:<your-branch> \
  --title "fix: <description>" \
  --body "$PR_BODY"
```

**If PR creation fails with "Resource not accessible by personal access token":**
- `GITHUB_TOKEN` env var may be overriding the keyring's properly scoped token
- **First try:** `unset GITHUB_TOKEN && gh pr create ...` (re-run the same command)
- **If still blocked:** your gh token lacks `repo` or `public_repo` scope
- **Fallback:** Create PR manually via GitHub UI:
  1. Go to: `https://github.com/<upstream-owner>/<repo-name>/compare/<upstream-branch>...<your-username>:<your-branch>`
  2. GitHub will auto-populate the PR template if one exists in the repo
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

## Discovering Issues When "good first issue" Is Absent

Many productive repos (especially large ones) don't use the `good first issue` label. When this happens, score open issues heuristically:

```python
# Example heuristic scoring for Hermes-agent style labels
for issue in open_issues:
    score = 0
    labels = [l['name'] for l in issue['labels']]
    
    # Priority: lower = easier
    if 'P3' in labels: score += 3
    if 'P2' in labels: score += 2
    if 'P1' in labels: score += 1
    
    # Type: docs/tests easiest, bugs next, features last
    if 'type/docs' in labels: score += 5
    if 'type/test' in labels: score += 4
    if 'type/bug' in labels: score += 2
    if 'type/feature' in labels: score += 1
    
    # Component: surface UI easier than core logic
    easy_components = {'comp/cli', 'comp/tui', 'area/config'}
    hard_components = {'comp/agent', 'comp/gateway', 'comp/plugins'}
    if any(c in labels for c in easy_components): score += 2
    if any(c in labels for c in hard_components): score -= 2
    
    issue['_score'] = score
```

Sort descending and pick the highest-scored unassigned issue with a clear description.

## Pitfalls

| Issue | Solution |
|-------|----------|
| No auth configured | Check first, fail fast with clear message |
| Repo bans AI contributions | Check CONTRIBUTING.md and README for AI prohibition keywords before cloning. Skip immediately if banned. |
| CONTRIBUTING requires assignment before PR | Read CONTRIBUTING thoroughly; comment on issue requesting assignment if required |
| PR template ignored | Always fetch `.github/PULL_REQUEST_TEMPLATE.md` (or variants) before drafting. Use it verbatim, filling in placeholders. |
| PR template has checklists | Complete checklists honestly (e.g., tests run, docs updated). Do not check boxes you haven't done. |
| gh CLI not in PATH | Auth check may pass but gh might not be executable. Verify with `command -v gh` before relying on it. If unavailable, set up SSH key or git credentials as fallback. |
| SSH key not configured | Terminal sessions may lack SSH access. Set up SSH key with `ssh-keygen -t ed25519` and add to GitHub, or use `gh auth setup-git` to configure credential helper. |
| Auth check passes but push fails | The auth check only verifies credentials exist, not that they work. **Always test push capability early** — create a test branch and attempt push before implementing the full fix. See "Step 0: Verify Push Capability" below. |
| Cannot push to upstream repo | You don't have write access. **Fork first:** `gh repo fork`, then push to your fork and create PR from fork:branch to upstream:base. |
| Fork push fails with "Host key verification failed" | The fork remote may default to SSH. If SSH keys aren't configured in the session, switch to HTTPS: `git remote set-url fork https://github.com/<user>/<repo>.git` then retry push. |
| PR creation fails with "Resource not accessible by personal access token" | Your gh token lacks `repo` or `public_repo` scope. **Fallback:** Create PR manually via GitHub UI at `https://github.com/<upstream-owner>/<repo-name>/compare/<upstream-branch>...<your-username>:<your-branch>`. Click "Create pull request", fill in title/body, submit. |
| Terminal execution tools unavailable | If you cannot run gh/git commands (session limitations, Docker issues), send complete copy-paste instructions to the user via send_message. Include: (1) the exact gh pr create command, (2) the GitHub UI fallback URL, (3) expected output. User can then execute manually. |
| GITHUB_TOKEN env var overrides keyring | gh CLI prioritizes GITHUB_TOKEN environment variable over keyring tokens. If PR creation fails with "Resource not accessible", the env var token may have limited scopes. **Fix:** `unset GITHUB_TOKEN` then retry, or ensure env var token has `repo` scope. |
| Issue already claimed | Check for existing PRs, skip if found |
| Unclear requirements | Comment on issue asking for clarification |
| Tests fail locally | Run test suite before pushing. **Python version caveat:** If the project requires Python 3.12+ (PEP 695 type params like `def identity[T]()`), your sandbox Python 3.11 will fail with `SyntaxError`. This does NOT mean your test code is wrong — it means you can't run tests locally. Verify test syntax by reading existing test files in the repo for patterns, then trust CI to validate. Add a note in your PR body if tests couldn't be run locally due to Python version. |
| Inline code is hard to test | If you patch logic directly inside a long function (e.g. `start_server`), it becomes untestable without spawning the whole server. **Extract a helper early** — move the logic into a small pure function, patch the function, write unit tests against it, then call it from the original location. |
| Large patch accidentally removes nearby code | When using `patch` with a big `old_string`/`new_string` block, adjacent lines (like `mount_spa(app)`) can get swallowed. **After every large patch, run `git diff` immediately** to verify no unintended deletions. If something is missing, restore it in a follow-up patch. |
| CI fails after push | Monitor checks, fix in follow-up commit |
| No maintainer response >2 weeks | Log and move to next opportunity |
| `/tmp` directory unreliable | On macOS, `/tmp` resolves to `/private/tmp` and can get corrupted between terminal calls. Always use `~/oss-contribution/` or another stable directory under home. |

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
3. CHECK CONTRIBUTING.md BEFORE CLONING — scan for AI prohibition, CLA, assignment rules, commit formats
4. CHECK README for AI bans — some repos state AI policy there instead of CONTRIBUTING
5. No write access? Fork first — gh repo fork, then push to your fork
6. Fork push blocked by SSH? Switch to HTTPS: git remote set-url fork https://github.com/<user>/<repo>.git
7. PR creation blocked? `unset GITHUB_TOKEN` first, then retry. Still blocked? Use GitHub UI: github.com/OWNER/REPO/compare/base...user:branch
8. Terminal tools unavailable? Send complete copy-paste instructions to user via send_message
9. Terminal lost mid-session? Log branch/repo to contributions.md, complete PR on next session
10. Never leave pushed branch without PR — always complete the loop
11. ALWAYS CHECK FOR PR TEMPLATE — fetch .github/PULL_REQUEST_TEMPLATE.md (or variants) before drafting PR body
12. Use the template verbatim — fill placeholders, complete checklists honestly, don't fabricate checks
13. Score issues — stars, recency, description quality matter
14. No "good first issue" label? Use heuristic scoring on priority, type, and component labels
15. Inline logic hard to test? Extract a helper early, test the helper, call it from the original function
16. After large patches, run `git diff` immediately to catch accidentally removed lines
17. Minimal fixes — small PRs merge faster
18. Log everything — both successes and blockers
19. Move on if stuck — many opportunities exist
20. Use ~/oss-contribution/ NOT /tmp/ — /tmp is unreliable across terminal calls
21. Check issue for claims BEFORE investing time — look for PRs and comments
22. Python 3.12+ projects can't be tested locally on Python 3.11 — trust CI, verify patterns from existing tests
23. Search API lacks star counts — fetch repo metadata separately via gh api repos/{owner}/{repo}
24. CONTRIBUTING silent on AI? Proceed cautiously — treat like any external contributor
```
