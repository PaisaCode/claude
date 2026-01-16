---
description: Generate a non-technical testing plan for human testers from a PR
allowed-tools: Bash(git:*), Bash(gh:*), Read, Grep, Glob
argument-hint: [--comment] [PR_NUMBER or PR_URL]
---

# Generate QA Testing Plan

Review changes in a PR and generate a clear, non-technical testing plan that human testers can follow.

## Arguments

- `--comment`: Post the test plan as a comment on the PR (optional)
- No argument: Use the current branch's PR
- PR number (e.g., `123`): Review that specific PR
- PR URL: Extract PR number from URL

## Examples

```
/test-plan                    # Generate plan for current branch's PR
/test-plan 123                # Generate plan for PR #123
/test-plan --comment          # Generate and post as comment on current PR
/test-plan --comment 123      # Generate and post as comment on PR #123
```

## Instructions

### Step 1: Parse Arguments and Identify the PR

First, check if `$ARGUMENTS` contains `--comment` flag:
- If `--comment` is present, set `POST_COMMENT=true` and remove it from the arguments
- Otherwise, set `POST_COMMENT=false`

Then identify the PR from the remaining argument:
- If it's a number, use it directly
- If it's a URL, extract the PR number
- If no argument remains, get PR from current branch:
```bash
gh pr view --json number,title,body,baseRefName,headRefName
```

### Step 2: Get PR Information

```bash
gh pr view <PR_NUMBER> --json number,title,body,baseRefName,headRefName,files,additions,deletions
```

### Step 3: Get the Changed Files

```bash
gh pr diff <PR_NUMBER> --name-only
```

### Step 4: Analyze the Code Changes

```bash
gh pr diff <PR_NUMBER>
```

Read and understand:
1. What features/functionality changed
2. What user-facing behavior is affected
3. What screens/pages are impacted
4. What user flows are modified

Focus on:
- UI components and pages
- API endpoints that affect user data
- Form validations and error handling
- Navigation and routing changes
- Permission/access control changes

### Step 5: Generate the Testing Plan

Create a testing plan with these sections:

---

## Testing Plan for PR #<number>: <title>

### Summary
[2-3 sentences explaining what this PR does in plain language]

### What Changed
[Bullet list of user-visible changes, no technical jargon]

### Areas to Test

#### 1. [Feature/Area Name]

**Setup:**
- [Any prerequisites like test accounts, data needed]

**Test Steps:**
1. [Clear step-by-step instructions]
2. [Use specific UI element names: "Click the blue 'Save' button"]
3. [Include what to look for: "You should see a success message"]

**Expected Result:**
- [What should happen if it works correctly]

**Things to Watch For:**
- [Edge cases or potential issues]

#### 2. [Another Feature/Area]
[Repeat structure]

---

### Regression Testing

Check that these existing features still work:
- [ ] [Related feature that might be affected]
- [ ] [Another related feature]

### Edge Cases to Test

- [ ] [Unusual scenario 1]
- [ ] [Unusual scenario 2]

### Different Environments/Conditions

- [ ] Test on mobile viewport
- [ ] Test with slow network (if applicable)
- [ ] Test as different user roles (if applicable)

### Notes for Testers

[Any additional context, known limitations, or special instructions]

---

## Guidelines for Writing Test Plans

### DO:
- Use plain, simple language
- Be specific about what to click and where
- Include exact button names, menu items, field labels
- Describe what success looks like
- Mention any test data or accounts needed
- Group related tests together

### DON'T:
- Use technical terms (API, endpoint, component, state, etc.)
- Assume testers know the codebase
- Skip steps that seem "obvious"
- Write vague instructions like "test the form"

### Example Good Test Step:
```
1. Go to the Login page
2. Enter "test@example.com" in the Email field
3. Enter "wrongpassword" in the Password field
4. Click the "Sign In" button
5. You should see a red error message saying "Invalid email or password"
```

### Example Bad Test Step:
```
1. Test the auth flow with invalid credentials
```

## Output Format

Write the testing plan as a clean markdown document that can be shared directly with QA testers. The plan should be self-contained and require no additional context to follow.

### Step 6: Post as PR Comment (if --comment flag was set)

If `POST_COMMENT=true`, post the generated test plan as a comment on the PR:

```bash
gh pr comment <PR_NUMBER> --body "<TEST_PLAN_CONTENT>"
```

Use a HEREDOC for the body to preserve formatting:
```bash
gh pr comment <PR_NUMBER> --body "$(cat <<'EOF'
## 🧪 QA Testing Plan

<generated test plan content here>
EOF
)"
```

After posting, confirm to the user that the test plan was posted as a comment on the PR with a link to view it.
