# MR Review Preparation

Generate a comprehensive review guide for a GitLab Merge Request to help reviewers focus on the most important changes.

Arguments: $ARGUMENTS (MR number or URL, e.g., "1234" or "https://gitlab.com/nav/git.nav.com/-/merge_requests/1234")

## Instructions

### Step 1: Extract MR Number and Fetch Information

Extract the MR number from $ARGUMENTS:
- If $ARGUMENTS is a URL like `https://git.nav.com/.../merge_requests/1234`, extract `1234`
- If $ARGUMENTS is just a number like `1234`, use it directly
- Store as `MR_NUMBER` for use throughout

Fetch the MR details:

```bash
# Get MR metadata (replace $MR_NUMBER with the extracted number)
glab mr view $MR_NUMBER --output json

# Get the full diff
glab mr diff $MR_NUMBER
```

### Step 2: Analyze the Changes

Review the diff and categorize each file into one of these risk levels:

**HIGH RISK** - Requires careful review:
- Business logic changes (calculations, workflows, decision trees)
- Security-related code (auth, permissions, input validation, secrets handling)
- Database migrations or schema changes
- API contract changes (protobuf, OpenAPI, GraphQL schemas)
- Concurrency/async code (goroutines, channels, promises, transactions)
- Error handling changes in critical paths
- Configuration changes that affect production behavior

**MEDIUM RISK** - Standard review:
- New features with clear scope
- Refactoring existing code
- Adding new dependencies
- Test changes that might mask issues
- Changes to shared utilities or libraries

**LOW RISK** - Quick scan sufficient:
- Adding or updating tests (that don't change behavior)
- Documentation updates
- Code formatting/style changes
- Renaming without behavior change
- Adding logging/observability
- Generated code (protobuf stubs, mocks, etc.)

### Step 3: Identify Focus Areas

For HIGH and MEDIUM risk files, identify the specific code sections that need attention:
- Functions/methods with complex logic
- Areas where bugs are likely to hide
- Code that interacts with external systems
- State mutations and side effects

### Step 4: Generate Questions for Author

Based on gaps in the MR, generate questions the reviewer should ask:
- Missing test coverage for edge cases
- Unclear intent or design decisions
- Potential breaking changes not documented
- Missing error handling scenarios
- Performance implications not addressed

### Step 5: Check for Common Issues

Scan for these common problems:
- Hardcoded values that should be configurable
- Missing error handling
- N+1 query patterns
- Race conditions or deadlock potential
- Security issues (SQL injection, XSS, secrets in code)
- Breaking changes to public APIs
- Missing database indexes for new queries

## Output Format

Structure your output exactly like this:

---

## MR Review Guide: [MR Title]

**MR:** !$MR_NUMBER | **Author:** @username | **Target:** branch-name
**Changed Files:** X files | **Lines:** +added / -removed

### Executive Summary
[2-3 sentences explaining what this MR does and why. Focus on the "what" and "why", not the "how".]

### Review Priority

#### HIGH PRIORITY - Review Carefully
| File | Risk | What to Look For |
|------|------|------------------|
| path/to/file.go | Security | Auth token validation changes |
| path/to/migration.sql | Data | Schema change affects existing data |

#### MEDIUM PRIORITY - Standard Review
| File | Risk | What to Look For |
|------|------|------------------|
| path/to/service.go | Logic | New business workflow |

#### LOW PRIORITY - Quick Scan
- `path/to/file_test.go` - New tests
- `docs/README.md` - Documentation update

### Focus Areas

**1. [Most Critical Area]** (`file:line-range`)
> [Brief explanation of what's happening and what could go wrong]

```language
[Relevant code snippet - keep to 5-15 lines, showing the critical part]
```

**2. [Second Critical Area]** (`file:line-range`)
> [Brief explanation]

```language
[Relevant code snippet]
```

**3. [Third Critical Area]** (`file:line-range`)
> [Brief explanation]

```language
[Relevant code snippet]
```

### Questions for Author
- [ ] [Question about unclear design decision]
- [ ] [Question about missing test case]
- [ ] [Question about potential edge case]

### Potential Issues Detected
- **[Issue Type]**: [Description and location]

### What Looks Good
- [Positive observation about the code]
- [Good pattern being followed]

---

## Important Guidelines

- Be specific about file paths and line numbers when possible
- Don't overwhelm with minor issues - focus on what matters
- If the MR is well-structured with good descriptions, acknowledge that
- If the MR is too large (>500 lines of non-test code), suggest how it could be split
- Reference Nav engineering standards from CLAUDE.md when relevant
- Keep the output concise - reviewers are busy
- For code snippets in Focus Areas:
  - Include only the most relevant 5-15 lines
  - Show enough context to understand what's happening
  - Use the appropriate language for syntax highlighting (go, java, typescript, sql, etc.)
  - Prefer showing the "new" code from the diff, not the removed code

## Example Usage

- `/review-prep 1234` - Review MR #1234
- `/review-prep https://gitlab.com/nav/git.nav.com/-/merge_requests/1234` - Review from URL
