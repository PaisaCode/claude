---
description: Generate commit message and push changes to git
allowed-tools: Bash(git:*), Bash(gh:*), Read, Grep, Glob
argument-hint: [--only-selected]
---

# Git Commit with Generated Message

Automatically generate a meaningful commit message based on your changes and push to the remote.

## Arguments

- No argument: Stage all changes (`git add -A`) then commit and push
- `--only-selected`: Only commit already-staged files (skip `git add -A`)

## Instructions

### Step 1: Determine Mode

Check if `$ARGUMENTS` contains `--only-selected`:
- If YES: Skip staging, only commit what's already staged
- If NO: Run `git add -A` to stage all changes

### Step 2: Check for Changes

```bash
git status --porcelain
```

If no changes to commit, inform the user and exit.

### Step 3: Gather Context for Commit Message

Run these commands to understand the changes:

```bash
# See what files changed
git diff --cached --name-status

# See the actual diff (for context)
git diff --cached --stat

# Get detailed diff for message generation
git diff --cached
```

### Step 4: Analyze Changes and Generate Message

Based on the diff, determine:

1. **Type of change** (use conventional commits):
   - `feat`: New feature
   - `fix`: Bug fix
   - `refactor`: Code refactoring
   - `style`: Formatting, whitespace
   - `docs`: Documentation changes
   - `test`: Adding/updating tests
   - `chore`: Maintenance, dependencies, config

2. **Scope** (optional): The area of the codebase affected
   - Examples: `auth`, `api`, `ui`, `db`, `config`

3. **Subject**: Concise description of what changed (imperative mood)
   - Good: "add user authentication flow"
   - Bad: "added user authentication flow"

4. **Body** (if needed): More detailed explanation for complex changes

### Step 5: Stage Changes (if not --only-selected)

```bash
git add -A
```

### Step 6: Create the Commit

Use this format:

```bash
git commit -m "$(cat <<'EOF'
<type>(<scope>): <subject>

<body if needed>

EOF
)"
```

Examples:
- `feat(auth): add password reset functionality`
- `fix(api): handle null response in user endpoint`
- `refactor: simplify date formatting utilities`
- `chore: update dependencies to latest versions`

### Step 7: Push to Remote

```bash
git push
```

If the branch has no upstream, use:
```bash
git push -u origin HEAD
```

### Step 8: Report Results

Output:
1. The generated commit message
2. Files included in the commit
3. Push status (success/failure)
4. Link to the commit if available

## Commit Message Guidelines

- Keep subject line under 72 characters
- Use imperative mood ("add" not "added")
- Don't end subject with a period
- Separate subject from body with blank line
- Body should explain "what" and "why", not "how"
- Reference issues/PRs if relevant (e.g., "Fixes #123")

## Examples

**Single file change:**
```
fix(auth): correct token expiration check

The JWT expiration was being compared incorrectly, causing
premature logouts for users.
```

**Multiple related changes:**
```
feat(dashboard): add analytics widgets

- Add daily active users chart
- Add revenue metrics display
- Add export to CSV functionality
```

**Simple change:**
```
style: format code with prettier
```
