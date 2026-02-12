# Migrate Branch to GitHub

Migrate the current branch from GitLab to GitHub by pushing it and rebasing onto GitHub main. Automatically handles most rebase conflicts.

Use the `/create-mr` command afterward to create a Pull Request.

Arguments: $ARGUMENTS (optional - can override base branch, default is 'main')

## Instructions:

### 1. Validate Current State

First, check that we're not on the main branch and that the git state is clean:

```bash
CURRENT_BRANCH=$(git branch --show-current)

if [ -z "$CURRENT_BRANCH" ]; then
  echo "Error: Not on a branch (detached HEAD?)"
  exit 1
fi

if [ "$CURRENT_BRANCH" = "main" ]; then
  echo "Error: Cannot migrate main branch"
  exit 1
fi

echo "Migrating branch: $CURRENT_BRANCH"
```

### 2. Push Branch to GitHub as Backup

Push the current branch to GitHub before any destructive operations:

```bash
git push -u origin "$CURRENT_BRANCH"
echo "Branch pushed to GitHub as backup"
```

### 3. Fetch Latest from GitHub Main

```bash
BASE_BRANCH="${ARGUMENTS:-main}"
git fetch origin "$BASE_BRANCH"
echo "Fetched latest from origin/$BASE_BRANCH"
```

### 4. Rebase with Automatic Conflict Resolution

Enable git rerere (reuse recorded resolution) and attempt rebase with intelligent conflict handling:

```bash
git config rerere.enabled true

# Start rebase with patience algorithm for better conflict detection
if git rebase "origin/$BASE_BRANCH" --strategy-option=patience; then
  echo "Rebase completed successfully without conflicts"
else
  echo "Rebase has conflicts - attempting automatic resolution..."
fi
```

**Auto-resolution strategies:**
1. **Generated files** (*.pb.go, *.gen.go, *_moq_test.go) → Accept theirs
2. **BUILD.bazel files** → Accept theirs, will regenerate with gazelle
3. **go.mod/go.sum** → Accept theirs, regenerate with `go mod tidy`
4. **Import conflicts** → Merge both sides and sort alphabetically
5. **Whitespace conflicts** → Reformat with `go fmt`

Only stops for actual business logic conflicts that need human review.

### 5. Force Push Rebased Branch

```bash
git push -f origin "$CURRENT_BRANCH"
echo "Rebased branch force-pushed to GitHub"
```

### 6. Summary

Output migration status, branch info, remote URL, and prompt to use `/create-mr`.

## Important Notes

- Always creates a backup by pushing the branch before rebasing
- Uses git rerere to remember conflict resolutions
- Uses patience diff algorithm for better conflict detection
- Only aborts if conflicts can't be safely auto-resolved
- Never mentions AI or automation tools in commits

## Example Usage

- `/migrate-to-github` - Migrate current branch and rebase on main
- `/migrate-to-github develop` - Migrate current branch and rebase on develop
