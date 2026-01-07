---
description: Analyze pages and components to list all API calls for e2e testing interception
allowed-tools: Read, Grep, Glob, Bash(git diff:*), Bash(git show:*), Bash(git log:*), Write
argument-hint: [--full | --pr]
---

# API Calls Audit Command

Analyze the codebase to identify all API calls made by pages and their subcomponents. This helps with e2e testing by providing a comprehensive list of endpoints to intercept.

## Arguments

- `--full` or no argument: Analyze the entire project
- `--pr`: Analyze only files changed in the current PR/branch

## Instructions

1. **Determine Scope**: Check if `$ARGUMENTS` contains `--pr`. If so, only analyze files changed in the current branch compared to main.

2. **Find Pages**: Look for page components in:
   - `client/src/pages/**/*.tsx`
   - `client/src/routes/**/*.tsx`
   - Any files that define routes

3. **For Each Page**:
   - Read the page file
   - Identify all imported components (both local and from the project)
   - Recursively analyze each imported component

4. **Identify API Calls**: In each file, look for:
   - `useQuery(` calls - extract the query key and endpoint
   - `useMutation(` calls - extract the mutation function
   - `api.` or `*Api.` calls (e.g., `sessionApi.list`, `participantApi.create`)
   - `*Queries.` calls (e.g., `sessionQueries.list`, `quotaQueries.list`)
   - Direct `axios` or `fetch` calls
   - Custom hooks that wrap API calls (hooks starting with `use*` that return query/mutation)

5. **Extract Endpoint Information**:
   - For queries: Find the `baseUri` in the corresponding API file
   - For mutations: Identify the HTTP method and endpoint
   - Note any dynamic parameters (`:id`, etc.)

6. **Output Format**: Write results to `docs/api_calls_per_page_component.yaml` in this format:

```yaml
# API Calls by Page/Component
# Generated for e2e testing interception
# Scope: [full | pr]
# Generated: [timestamp]

pages:
  /projects:
    file: client/src/pages/projects/index.tsx
    api_calls:
      - endpoint: GET /api/projects/
        hook: useQuery(projectQueries.list)
        component: ProjectsPage
      - endpoint: POST /api/projects/
        hook: useMutation(useCreateProject)
        component: CreateProjectModal
    subcomponents:
      - name: ProjectTable
        file: client/src/components/project-table.tsx
        api_calls:
          - endpoint: GET /api/projects/
            hook: useQuery

  /projects/:id:
    file: client/src/pages/projects/project-detail.tsx
    api_calls:
      - endpoint: GET /api/projects/:id/
        hook: useQuery(projectQueries.retrieve)
        component: ProjectDetailPage
    subcomponents:
      - name: SessionsTab
        file: client/src/components/sessions-tab.tsx
        api_calls:
          - endpoint: GET /api/sessions/
            hook: useQuery(sessionQueries.list)
          - endpoint: POST /api/sessions/
            hook: useMutation(useCreateSession)
```

7. **PR Mode**: When `--pr` is specified:
   - Use `git diff main...HEAD --name-only` to get changed files
   - Only analyze pages/components that were modified
   - Add a `changed_in_pr: true` flag to affected entries

## Tips for Finding API Calls

- Check the `client/src/services/` directory for API definitions
- Look for `createApi` calls to understand the base URIs
- `*Queries` objects define query configurations
- `use*` hooks in services often wrap mutations

## Example Analysis

For a file like:
```tsx
import { useQuery } from '@tanstack/react-query'
import { sessionQueries } from '../services/sessions'

const { data } = useQuery(sessionQueries.list({ filters: {...} }))
```

Extract:
- Hook: `useQuery(sessionQueries.list)`
- Look up `sessionQueries.list` -> finds `baseUri: 'sessions/'`
- Result: `GET /api/sessions/`
