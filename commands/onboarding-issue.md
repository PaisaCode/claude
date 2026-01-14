---
description: Create GitHub issues for onboarding engineers with codebase references
allowed-tools: Bash(gh:*), Read, Grep, Glob, AskUserQuestion, Task
argument-hint: "<feature description>" [--dry-run]
---

# Onboarding Issue Generator

Create well-structured GitHub issues for onboarding engineers that explain tasks clearly, reference relevant libraries/patterns, and point to examples in the RAPA codebase.

## Arguments

- **feature description** (required): What the engineer should build/implement
- **--dry-run** (optional): Preview the issue without creating it

## Examples

```
/onboarding-issue "Report name editing"
/onboarding-issue "Add vendor search with autocomplete"
/onboarding-issue "Implement expense date range filtering"
/onboarding-issue "Create a new Tag entity with CRUD" --dry-run
```

## Instructions

### Step 1: Parse the Feature Description

Extract the feature description from `$ARGUMENTS`. Remove any `--dry-run` flag and note if it's present.

If no description is provided, use AskUserQuestion to ask:
"What feature or task should the onboarding engineer implement?"

### Step 2: Analyze What's Needed

Based on the feature description, determine:

1. **Stack layers involved:**
   - Frontend only (React components, forms, queries)
   - Backend only (Django models, serializers, views)
   - Full-stack (both)

2. **Libraries/patterns likely needed:**
   - `tn-models` - For API services and Zod shapes
   - `tn-forms` - For form handling and validation
   - `@tanstack/react-query` - For data fetching/caching
   - Django models, serializers, viewsets

3. **Type of work:**
   - New entity/feature
   - Modification to existing feature
   - UI component work
   - API endpoint work

### Step 3: Search for Relevant Examples

**IMPORTANT:** Do NOT write actual implementation code. Only provide file paths and brief descriptions. Short pseudo-code snippets are acceptable for illustrating patterns, but the engineer should write the real code themselves.

Use the Task tool with Explore agent to find relevant code examples in the codebase:

1. Search for similar implementations
2. Find the best reference files for each pattern needed
3. Note specific file paths and what they demonstrate

**Key directories to search:**
- Frontend services: `client/src/services/`
- Frontend components: `client/src/components/`
- Frontend pages: `client/src/pages/`
- Backend models: `server/rapa/*/models.py`
- Backend serializers: `server/rapa/*/serializers.py`
- Backend views: `server/rapa/*/views.py`

### Step 4: Generate the Issue Content

Create a GitHub issue with this structure:

```markdown
## Overview
[2-3 sentences explaining what the engineer will build and why it matters]

## What You'll Learn
- [Library/pattern 1 they'll practice]
- [Library/pattern 2 they'll practice]
- [Library/pattern 3 they'll practice]

## Task Description
[Detailed explanation of what to build, written in plain language]

### Requirements
- [Specific requirement 1]
- [Specific requirement 2]
- [Specific requirement 3]

## Reference Files
Study these files to understand the patterns:

### [Category 1 - e.g., "Frontend Service Layer"]
- `path/to/file.ts` - [What to learn from this file]
- `path/to/another.ts` - [What to learn from this file]

### [Category 2 - e.g., "Backend API"]
- `path/to/file.py` - [What to learn from this file]

## Implementation Hints
1. [First hint - where to start]
2. [Second hint - key pattern to follow]
3. [Third hint - common pitfall to avoid]

## Acceptance Criteria
- [ ] [Criterion 1]
- [ ] [Criterion 2]
- [ ] [Criterion 3]
- [ ] Code follows existing patterns in the codebase
- [ ] No linting errors (`pnpm tslint` / `ruff check`)

## Resources
- [Link to relevant docs if applicable]
```

### Step 5: Create the GitHub Issue

**If --dry-run flag is present:**
Display the full issue content and ask if it looks good.

**If NOT dry-run:**
Create the issue using:

```bash
gh issue create \
  --title "[ONBOARD] <short title based on feature>" \
  --body "$(cat <<'EOF'
<issue body content>
EOF
)" \
  --label "onboarding"
```

### Step 6: Report Results

Output:
1. Summary of the issue created
2. Link to the GitHub issue (or preview if dry-run)
3. Key files the engineer should study first

---

## Library Quick Reference

When explaining tasks, reference these libraries as appropriate:

### Frontend - tn-models (API Layer)
- **Purpose:** Type-safe API services with Zod validation
- **Key concepts:** `createApi`, `createCustomServiceCall`, shapes, filters
- **Best examples:** `client/src/services/expenses/api.ts`, `client/src/services/vendors/models.ts`

### Frontend - tn-forms (Form Handling)
- **Purpose:** Form state management with validation
- **Key concepts:** `Form` class, `FormField`, validators, `val` getter
- **Best examples:** `client/src/services/vendors/forms.ts`, `client/src/services/expenses/forms.ts`

### Frontend - TanStack Query (Data Fetching)
- **Purpose:** Server state management, caching, refetching
- **Key concepts:** `queryOptions`, query keys, pagination, mutations
- **Best examples:** `client/src/services/vendors/queries.ts`, `client/src/services/expenses/queries.ts`

### Backend - Django Models
- **Purpose:** Database schema and business logic
- **Key concepts:** `AbstractBaseModel`, custom QuerySet with `for_user()`, TextChoices
- **Best examples:** `server/rapa/expenses/models.py`, `server/rapa/core/models.py`

### Backend - DRF Serializers
- **Purpose:** API request/response transformation
- **Key concepts:** `ModelSerializer`, nested serializers, `_ref` pattern, validation
- **Best examples:** `server/rapa/expenses/serializers.py`

### Backend - DRF ViewSets
- **Purpose:** REST API endpoints
- **Key concepts:** `ModelViewSet`, `get_queryset()`, `@action` decorator, permissions
- **Best examples:** `server/rapa/expenses/views.py`
