---
description: Add features, components, or services from the current project to the TN SPA Bootstrapper
allowed-tools: Read, Grep, Glob, Bash, Write, Edit, Task, WebFetch, AskUserQuestion
argument-hint: <file_or_description>
---

# Add to TN SPA Bootstrapper

This command helps you contribute code from your current project back to the [@thinknimble/tn-spa-bootstrapper](https://github.com/thinknimble/tn-spa-bootstrapper) repository.

## Workflow

### Step 1: Understand the Request

Parse `$ARGUMENTS` to understand what the user wants to add:
- A specific file path (e.g., `client/src/services/myService.ts`)
- A component name or description
- A feature description

If unclear, use AskUserQuestion to clarify:
- What exactly should be added?
- Is it a component, service, utility, or full feature?
- Where in the bootstrapper should it live?

### Step 2: Clone/Update the Bootstrapper Repository

Check if the bootstrapper repo exists locally:

```bash
# Check if bootstrapper directory exists at a sibling level or in a known location
ls -d ../tn-spa-bootstrapper 2>/dev/null || ls -d ~/code/tn-spa-bootstrapper 2>/dev/null || ls -d ~/projects/tn-spa-bootstrapper 2>/dev/null
```

If not found, ask the user where it is or if they want to clone it:
```bash
gh repo clone thinknimble/tn-spa-bootstrapper <target_directory>
```

### Step 3: Check for Open PRs

List open PRs in the bootstrapper repo to see if there's an existing PR to add to:

```bash
cd <bootstrapper_path> && gh pr list --state open --json number,title,headRefName,body --limit 20
```

Present the open PRs to the user with AskUserQuestion:
- Option 1: Update an existing PR (show list of relevant PRs)
- Option 2: Create a new branch and PR

### Step 4: Prepare the Branch

**If updating an existing PR:**
```bash
cd <bootstrapper_path>
git fetch origin
gh pr checkout <pr_number>
git pull
```

**If creating a new branch:**
```bash
cd <bootstrapper_path>
git fetch origin
git checkout main
git pull origin main
git checkout -b feature/<descriptive-name>
```

### Step 5: Analyze the Code to Add

Read the source files from the current project that the user wants to add:

1. Identify all files involved
2. Check for dependencies and imports
3. Understand how the code integrates

Look for:
- Import statements that need to be updated
- Project-specific configurations that need generalizing
- Environment variables or constants
- Type definitions

### Step 6: Adapt the Code for the Bootstrapper

When copying code to the bootstrapper, you may need to:

1. **Generalize project-specific code**: Remove hardcoded values, use environment variables
2. **Update import paths**: Adjust to match bootstrapper structure
3. **Remove unused dependencies**: Strip out imports not needed
4. **Add documentation**: Ensure code is well-documented for bootstrapper users
5. **Follow bootstrapper conventions**: Match existing code style

Typical bootstrapper structure:
```
tn-spa-bootstrapper/
├── {{cookiecutter.project_slug}}/
│   ├── client/
│   │   ├── src/
│   │   │   ├── components/
│   │   │   ├── services/
│   │   │   ├── hooks/
│   │   │   ├── pages/
│   │   │   └── utils/
│   └── server/
│       └── {{cookiecutter.project_slug}}/
│           ├── core/
│           └── <app_name>/
```

Note: The bootstrapper uses cookiecutter templating. You may need to:
- Replace project names with `{{cookiecutter.project_slug}}`
- Use `{{cookiecutter.project_name}}` for display names
- Check existing files for patterns to follow

### Step 7: Write the Changes

1. Copy/adapt the code to the appropriate location in the bootstrapper
2. Update any related files (imports, exports, index files)
3. If the feature requires new dependencies, update package.json

### Step 8: Commit and Push

```bash
cd <bootstrapper_path>
git add -A
git status
git commit -m "feat: add <feature_description>

<detailed description of what was added>

Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>"
git push -u origin HEAD
```

### Step 9: Create or Update PR

**If creating a new PR:**
```bash
gh pr create --title "feat: <feature_title>" --body "## Summary
- Added <feature> from <source_project>
- <bullet points of what's included>

## Changes
- <list of files added/modified>

## Testing
- [ ] Tested in source project
- [ ] Verified bootstrapper still generates correctly

---
Generated with Claude Code"
```

**If updating existing PR:**
The push in Step 8 will automatically update the PR. Optionally add a comment:
```bash
gh pr comment <pr_number> --body "Added <feature> to this PR

Changes:
- <list changes>"
```

### Step 10: Report Results

Provide the user with:
1. Summary of what was added
2. PR URL (new or updated)
3. Any manual steps needed (e.g., testing, additional configuration)
4. List of files that were created/modified

## Example Usage

```
/add-to-bootstrapper client/src/services/notifications.ts
/add-to-bootstrapper "the new pagination hook we built"
/add-to-bootstrapper server/myapp/emails.py
```

## Important Notes

- Always verify the bootstrapper path with the user if uncertain
- The bootstrapper uses cookiecutter - respect template variables
- Test that the bootstrapper still works after changes if possible
- Be conservative with changes - only add what's requested
- If the code depends on project-specific features, note this in the PR
