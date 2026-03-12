# Commit and Create GitHub PR

Create a commit with the latest changes and open a pull request on GitHub.

Arguments: $ARGUMENTS (optional - can include target branch or PR title)

## Instructions:

### Step 1: Pre-push review agents

Before committing, kick off these two review agents IN PARALLEL using the Task tool:

1. **`pre-push-review`** subagent - Run with the instruction: "Review the changes on this branch against main"
2. **`go-lead`** subagent - Run with the instruction: "Review the Go code changes on this branch against main"

Wait for both agents to complete. If either reports BLOCKING ISSUES, stop and present the findings to the user before proceeding.

### Step 2: Commit and push

1. Check `git status` to see what files have been modified
2. If there are changes to commit:
   - Stage all modified files: `git add -A`
   - Create a descriptive commit message based on the actual changes made
     - Follow conventional commit format (feat:, fix:, docs:, style:, refactor:, test:, chore:)
     - Be clear and specific about what was changed
     - Never mention AI, automation, or any tools used
   - Example: `git commit -m "feat: add user authentication to login page"`
3. Push the current branch to origin:
   ```bash
   git push -u origin HEAD
   ```

### Step 3: Create pull request

Create a pull request using gh with proper formatting:
- Parse $ARGUMENTS for target branch (default to 'main' if not specified)
- Generate an appropriate PR title based on the branch name and recent commits
- Use the temporary file approach for the PR description:

```bash
description_file=$(mktemp)

cat > "$description_file" << 'EOF'
## Summary
[Brief summary of what was changed]

## Changes
- [Specific change 1]
- [Specific change 2]
- [Additional changes...]

## Testing
- [How the changes were tested]

## Impact
- [Any potential impacts or considerations]
EOF

gh pr create \
  --title "[Generated PR Title]" \
  --body "$(cat "$description_file")" \
  --base [target-branch]

rm "$description_file"
```

**Alternative for complex cases**: If the description is too complex or if there are issues, use:
```bash
gh pr create --web
```

After creating the PR, capture and display the PR URL.

## Important Guidelines:
- Never mention that code was generated or assisted by AI/Claude
- Focus on what the code does, not how it was created
- Make commit messages and PR descriptions professional and focused on the technical changes
- Use conventional commit format for consistency
- Branch names should follow: `<initials>/<service>/<short-description>` (e.g., `bc/account/filter-by-capabilities`)

## Common Issues to Avoid:
- Don't use --description (doesn't exist in gh, use --body or -b)
- Don't put multi-line strings directly in command line
- Don't use nested quotes without proper escaping
- Always use the temporary file approach for multi-line descriptions
- Use --web as a fallback for complex cases

## Available gh pr create flags:
- `-t, --title string` - PR title
- `-b, --body string` - PR description/body
- `-w, --web` - Open in browser (safest option)
- `-B, --base string` - Target/base branch (default: main)
- `-d, --draft` - Create as draft PR
