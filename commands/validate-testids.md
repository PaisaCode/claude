---
description: Validate that components have data-testid attributes for testing
allowed-tools: Read, Grep, Glob, Bash(git diff:*), Bash(git log:*), Edit, Write
argument-hint: [--path <path> | --pr | --fix]
---

# Validate Test IDs Command

Search for React components to validate they have `data-testid` attributes. This helps ensure components are properly instrumented for E2E testing.

## Arguments

- `--path <path>`: Validate components in a specific directory or file (e.g., `--path client/src/components`)
- `--pr`: Validate only components changed in the current PR/branch
- `--fix`: Automatically add missing `data-testid` attributes to components
- `--interactive`: (Use with --fix) Ask before each fix

## Examples

```bash
# Validate all components in a directory
/validate-testids --path client/src/components

# Validate components changed in current PR
/validate-testids --pr

# Validate and fix missing test IDs
/validate-testids --path client/src/pages --fix

# Validate a specific file
/validate-testids --path client/src/components/Button.tsx
```

## Instructions

### Step 0: Parse Arguments

Parse `$ARGUMENTS` to determine the mode:

**Check for flags:**
- `--path <path>`: Set `TARGET_PATH=<path>`
- `--pr`: Set `PR_MODE=true`
- `--fix`: Set `FIX_MODE=true`
- `--interactive`: Set `INTERACTIVE_MODE=true`

**Default behavior:**
- If no path or --pr is specified, default to `client/src/components` and `client/src/pages`

### Step 1: Identify Files to Validate

**If `--pr` mode:**

```bash
git diff main...HEAD --name-only | grep -E '\.(tsx|jsx)$'
```

**If `--path` mode:**

Use Glob to find all `.tsx` and `.jsx` files in the specified path.

**Default:**

Search in common component directories:
- `client/src/components/**/*.tsx`
- `client/src/pages/**/*.tsx`

### Step 2: Define What Needs data-testid

The following elements SHOULD have `data-testid` attributes:

**High Priority (Required):**
1. **Interactive elements:**
   - `<button>` - buttons for actions
   - `<a>` - links for navigation
   - `<input>` - form inputs
   - `<select>` - dropdowns
   - `<textarea>` - text areas
   - `<form>` - forms

2. **UI Library Components (common ones):**
   - `<Button>` - MUI, Chakra, custom buttons
   - `<TextField>` - MUI text fields
   - `<Select>` - MUI select
   - `<Dialog>` - MUI dialogs/modals
   - `<Modal>` - modals
   - `<Drawer>` - drawers/sidebars
   - `<Menu>` - menus
   - `<MenuItem>` - menu items
   - `<Tab>` - tabs
   - `<Card>` - cards (if clickable)
   - `<IconButton>` - icon buttons
   - `<Checkbox>` - checkboxes
   - `<Radio>` - radio buttons
   - `<Switch>` - toggle switches
   - `<Autocomplete>` - autocomplete inputs

3. **Content containers that represent testable units:**
   - `<Table>` - tables
   - `<TableRow>` - table rows (especially with actions)
   - `<List>` - lists
   - `<ListItem>` - list items (if interactive)

**Medium Priority (Recommended):**
4. **Display elements with dynamic content:**
   - Elements showing loading states
   - Elements showing error messages
   - Elements showing success messages
   - Toast/Snackbar notifications

**Low Priority (Optional):**
5. **Static content containers** - only if needed for specific tests

### Step 3: Analyze Each File

For each file, analyze the JSX/TSX and identify:

1. **Elements that need data-testid but don't have one:**
   - Search for the element patterns listed above
   - Check if they already have `data-testid` attribute
   - Note the line number and element type

2. **Elements that already have data-testid:**
   - Validate the naming convention follows the pattern: `kebab-case-descriptive-name`
   - Good: `data-testid="submit-button"`, `data-testid="user-email-input"`
   - Bad: `data-testid="btn1"`, `data-testid="SubmitButton"`

### Step 4: Generate Report

Create a report showing:

```markdown
# data-testid Validation Report
Generated: [timestamp]
Mode: [path | pr]
Fix Mode: [enabled | disabled]

## Summary
- Files Analyzed: X
- Components Missing data-testid: Y
- Components with data-testid: Z
- Coverage: XX%

## Missing data-testid

### client/src/components/UserForm.tsx
| Line | Element | Suggested data-testid |
|------|---------|----------------------|
| 24 | `<Button>Submit</Button>` | `user-form-submit-button` |
| 31 | `<TextField label="Email" />` | `user-form-email-input` |
| 45 | `<Select>` | `user-form-role-select` |

### client/src/pages/Dashboard.tsx
| Line | Element | Suggested data-testid |
|------|---------|----------------------|
| 12 | `<IconButton onClick={handleRefresh}>` | `dashboard-refresh-button` |

## Existing data-testid (Valid)
- client/src/components/Header.tsx: 3 elements with valid test IDs
- client/src/components/Sidebar.tsx: 5 elements with valid test IDs

## Naming Issues
| File | Line | Current | Suggested |
|------|------|---------|-----------|
| Header.tsx | 15 | `data-testid="btn1"` | `data-testid="header-menu-button"` |
```

### Step 5: Fix Mode (if --fix is enabled)

When `--fix` is enabled, automatically add `data-testid` attributes:

**Naming Convention:**
Generate `data-testid` values following this pattern:
```
[component-context]-[element-purpose]-[element-type]
```

Examples:
- Button in LoginForm that submits: `login-form-submit-button`
- Email input in UserProfile: `user-profile-email-input`
- Delete button in a table row: `[table-name]-row-delete-button`
- Modal for creating a project: `create-project-modal`

**How to add data-testid:**

For a component like:
```tsx
<Button onClick={handleSubmit}>Submit</Button>
```

Transform to:
```tsx
<Button data-testid="form-submit-button" onClick={handleSubmit}>Submit</Button>
```

For self-closing elements:
```tsx
<TextField label="Email" />
```

Transform to:
```tsx
<TextField data-testid="form-email-input" label="Email" />
```

**Interactive Mode (--interactive):**
If `--interactive` is enabled with `--fix`, ask the user before each fix:
- Show the element and suggested data-testid
- Allow user to approve, modify, or skip

### Step 6: Output

**Without --fix:**
Write the report to `docs/testid-validation-report.md`

**With --fix:**
1. Apply the changes to the files using Edit tool
2. Write a summary report showing what was changed
3. Output the list of modified files

## Tips

1. **Context Matters**: Use the component name and parent context to generate meaningful test IDs
   - In `UserSettings.tsx` with a save button: `user-settings-save-button`
   - In `ProjectList.tsx` with a create button: `project-list-create-button`

2. **Avoid Generic Names**: Instead of `button-1` or `input`, use descriptive names that indicate purpose

3. **Consider Dynamic Elements**: For elements in loops/maps, include an identifier:
   ```tsx
   {items.map(item => (
     <Button data-testid={`item-${item.id}-delete-button`} />
   ))}
   ```

4. **Forms**: Group related form elements:
   - `login-form-email-input`
   - `login-form-password-input`
   - `login-form-submit-button`

5. **Modals/Dialogs**: Include the modal context:
   - `confirm-delete-modal`
   - `confirm-delete-modal-confirm-button`
   - `confirm-delete-modal-cancel-button`

## Exclusions

Skip adding data-testid to:
- Pure presentational components with no interactivity
- Components inside `__tests__` or `*.test.tsx` files
- Third-party component wrappers that pass through props
- SVG elements (use wrapper instead)
- Fragment elements `<>...</>`
