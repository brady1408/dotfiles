# Create MR/PR with Review Focus Areas

Create a merge/pull request with a comprehensive description and inline comments highlighting focus areas for reviewers.

Arguments: $ARGUMENTS (optional - target branch, defaults to main)

## Overview

This command:

1. Runs pre-push validation (checks for blocking issues)
2. Generates a comprehensive description with review focus areas
3. Creates the MR/PR
4. **MUST post inline comments at focus areas** - this is NOT optional

## CRITICAL REQUIREMENT

**You MUST post inline comments on the MR/PR after creating it.** The inline comments are the most valuable part of this command - they guide reviewers directly to the code that needs attention. DO NOT consider this command complete until you have:

1. Created the MR/PR
2. Posted inline comments at each focus area (3-5 comments)
3. Confirmed the comments were posted successfully

If you skip the inline comments, the command has failed its primary purpose.

## Instructions

### Step 0: Detect Platform

```bash
# Check if we're in a GitLab or GitHub repo
git remote -v | head -1
```

- If remote contains `gitlab` → use `glab`
- If remote contains `github` → use `gh`

### Step 1: Pre-Push Validation

Before creating the MR/PR, validate the changes won't have blocking issues:

```bash
# Get current branch
BRANCH=$(git branch --show-current)

# Get base branch (from args or default to main)
BASE_BRANCH="${ARGUMENTS:-main}"

# Check we're not on main
if [ "$BRANCH" = "main" ] || [ "$BRANCH" = "master" ]; then
  echo "ERROR: Cannot create MR from main/master branch"
  exit 1
fi

# Get diff stats
git diff $BASE_BRANCH...HEAD --stat
```

**Run these checks:**

1. No blocking security issues (hardcoded secrets, SQL injection)
2. No uncommitted changes
3. Branch has commits ahead of base

If blocking issues found, stop and report them. Don't create the MR/PR.

### Step 2: Analyze Changes for Description

```bash
# Get all commits
git log $BASE_BRANCH..HEAD --oneline

# Get full diff
git diff $BASE_BRANCH...HEAD

# Get changed files
git diff $BASE_BRANCH...HEAD --name-only
```

**Analyze and identify:**

1. What this change does (2-3 sentence summary)
2. List of changes (grouped logically)
3. Top 3-5 focus areas with file:line references
4. Testing notes
5. Rollback plan (if applicable)

### Step 3: Push Branch

```bash
git push -u origin HEAD
```

### Step 4: Create MR/PR with Description

**For GitLab (glab):**

```bash
# Create temp file for description
desc_file=$(mktemp)
cat > "$desc_file" << 'DESCRIPTION'
## Summary

[2-3 sentences explaining what and why]

## Changes

- [Change 1]
- [Change 2]

## Review Focus Areas

> Inline comments have been added at these locations to help guide your review.

**1. [Area name]** (`file:line`)
[Why this needs attention]

**2. [Area name]** (`file:line`)
[Why this needs attention]

**3. [Area name]** (`file:line`)
[Why this needs attention]

## Testing

- [How it was tested]

## Rollback Plan

[How to rollback if needed]
DESCRIPTION

glab mr create \
  --title "[Title based on changes]" \
  --description "$(cat "$desc_file")" \
  --target-branch "$BASE_BRANCH" \
  --no-editor

rm "$desc_file"
```

**For GitHub (gh):**

```bash
desc_file=$(mktemp)
cat > "$desc_file" << 'DESCRIPTION'
[Same format as above]
DESCRIPTION

gh pr create \
  --title "[Title based on changes]" \
  --body-file "$desc_file" \
  --base "$BASE_BRANCH"

rm "$desc_file"
```

### Step 5: Post Inline Focus Comments (REQUIRED)

**THIS STEP IS MANDATORY. DO NOT SKIP THIS STEP.**

After creating the MR/PR, you MUST post inline comments at each focus area.

**For GitLab:**

```bash
# Get the commit SHAs needed for positioning
BASE_SHA=$(git merge-base origin/main HEAD)
HEAD_SHA=$(git rev-parse HEAD)

# IMPORTANT: Use JSON format with Content-Type header, NOT --field/-f flags
glab api projects/:id/merge_requests/$MR_NUMBER/discussions --method POST \
  -H "Content-Type: application/json" \
  --input - << EOF
{
  "body": "🔍 **Review Focus Area**\n\n[Explanation]\n\n**What to check:**\n- [Item 1]\n- [Item 2]",
  "position": {
    "base_sha": "$BASE_SHA",
    "head_sha": "$HEAD_SHA",
    "start_sha": "$BASE_SHA",
    "position_type": "text",
    "new_path": "path/to/file.go",
    "new_line": 42
  }
}
EOF
```

**CRITICAL: GitLab Inline Comment Requirements:**

- MUST use `-H "Content-Type: application/json"` header
- MUST use `--input -` with JSON (NOT `-f` or `--field` flags)
- DO NOT use `-f` or `--field` flags - they cannot properly format nested objects

**For GitHub:**

```bash
gh api repos/:owner/:repo/pulls/$PR_NUMBER/comments --method POST \
  -f "body=**Review Focus Area**: [Explanation]" \
  -f "commit_id=$HEAD_SHA" \
  -f "path=path/to/file.go" \
  -f "line=42" \
  -f "side=RIGHT"
```

### Step 6: Output Summary

Output MR/PR URL, branch info, title, list of posted inline comments, and next steps.

## Important Guidelines

- **ALWAYS post inline comments** - this is the primary value of this command
- Never mention AI, Claude, or automation in descriptions or comments
- If push fails, report the error and don't proceed to MR creation
- Post 3-5 inline focus comments (minimum 3, maximum 5)
- **The command is NOT complete until inline comments are posted**

## Platform Detection Notes

The command auto-detects GitLab vs GitHub from the remote URL:

- `git.nav.com` or `gitlab` → GitLab (glab)
- `github.com` → GitHub (gh)

## Example Usage

- `/create-mr` - Create MR/PR targeting main
- `/create-mr develop` - Create MR/PR targeting develop branch
