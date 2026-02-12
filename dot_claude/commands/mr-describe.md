# Generate MR Description

Generate a comprehensive merge request description that helps reviewers understand and focus on what matters.

Arguments: $ARGUMENTS (optional - "update" to update current branch's MR, or MR number to update a specific MR)

## Instructions

### Step 1: Gather Context

```bash
# Get the current branch name
git branch --show-current

# Find the base branch (usually main)
git merge-base HEAD main

# Get all commits since diverging from main
git log main..HEAD --oneline

# Get the full diff against main
git diff main...HEAD

# Get list of changed files with stats
git diff main...HEAD --stat
```

### Step 2: Analyze Each Commit

For each commit since branching from main:
- Understand its purpose (feature, fix, refactor, test, etc.)
- Note which files it touches
- Identify if it's a logical unit or part of a larger change

### Step 3: Categorize Changes by Risk

Review all changed files and categorize:

**HIGH RISK** (reviewers must focus here):
- Business logic changes
- Security-related code (auth, permissions, validation)
- Database migrations or schema changes
- API contract changes (protobuf, REST endpoints)
- Concurrency/async code
- Error handling in critical paths

**MEDIUM RISK** (standard review needed):
- New features with clear scope
- Refactoring
- New dependencies
- Shared utility changes

**LOW RISK** (quick scan):
- Tests (that don't change behavior)
- Documentation
- Formatting/style
- Generated code

### Step 4: Identify Review Focus Areas

For HIGH and MEDIUM risk files, identify the top 3-5 specific areas reviewers should focus on:
- Complex logic that could have edge cases
- Changes that interact with external systems
- State mutations and side effects
- Areas where the "why" isn't obvious from the code

### Step 5: Generate the Description

Based on your analysis, generate a description with these sections:

**Summary**: 2-3 sentences explaining what this MR does and why. Focus on business/technical value, not implementation details.

**Changes**: Bullet points grouped logically. Be specific but concise.

**Review Focus Areas**: Top 3-5 areas with file:line references and 1-2 sentence explanations of what to look for.

**Testing**: How this was tested - checkboxes for test cases covered.

**Rollback Plan**: How to rollback if issues are found (feature flag, revert, migration rollback, etc.)

**Related**: Links to tickets, related MRs, or documentation if applicable.

Format the description like this:

```markdown
## Summary

[Your 2-3 sentence summary based on the actual changes]

## Changes

- [Specific change from the diff]
- [Another change]
- [Group related changes together]

## Review Focus Areas

> These are the areas that need careful review. Other changes are lower risk.

**1. [Descriptive area name]** (`path/to/file.go:42-55`)
[1-2 sentences explaining what to look for and why it matters]

**2. [Descriptive area name]** (`path/to/other.go:100`)
[1-2 sentences]

**3. [Descriptive area name]** (`path/to/third.go:25-40`)
[1-2 sentences]

## Testing

- [ ] [Specific test case]
- [ ] [Another test case]
- [ ] [Manual testing performed]

## Rollback Plan

[Specific rollback approach based on the changes]

## Related

- [Link to ticket if applicable]
- [Link to related MRs if applicable]
```

### Step 6: Apply or Output the Description

Based on $ARGUMENTS:

**If "update" is provided:**
```bash
# Get the MR number for the current branch
MR_NUMBER=$(glab mr list --source-branch "$(git branch --show-current)" --state opened --json iid -q '.[0].iid')

# Update the MR description
glab mr update $MR_NUMBER --description "YOUR_GENERATED_DESCRIPTION"
```

**If a specific MR number is provided (e.g., "1759"):**
```bash
# Update that specific MR
glab mr update $ARGUMENTS --description "YOUR_GENERATED_DESCRIPTION"
```

**If no arguments provided:**
Output the generated description for the user to:
- Copy/paste into an MR manually
- Use with `/create-mr` command
- Review before applying

## Important Guidelines

- Never mention AI, Claude, or automation in the description
- Focus on the "what" and "why", not the "how"
- Be specific about file paths and line numbers in focus areas
- Keep the summary concise - reviewers are busy
- Include rollback plan for any production-impacting changes
- If there are breaking changes, call them out prominently
- If the MR is large (>500 lines of non-test code), suggest how it could be reviewed in parts

## Example Usage

- `/mr-describe` - Generate description and output it
- `/mr-describe update` - Generate and update the current branch's open MR
- `/mr-describe 1759` - Generate and update MR #1759
