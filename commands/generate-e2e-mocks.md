---
description: Generate mocked E2E tests with Playwright route interception using faker for test data
allowed-tools: Read, Grep, Glob, Bash(git:*), Bash(gh:*), Write, Edit, Task
argument-hint: [--pr | --page <path> | --describe "<test description>"]
---

# Generate Mocked E2E Tests

Generate comprehensive E2E tests using Playwright with mocked API responses. Uses faker-js for realistic test data and properly intercepts all API calls including third-party services.

## Arguments

- `--pr`: Generate tests based on files changed in the current PR/branch
- `--page <path>`: Generate tests for a specific page (e.g., `--page client/src/pages/onboarding`)
- `--component <path>`: Generate tests for a specific component
- `--full`: Analyze entire project and generate test suite
- `--describe "<description>"`: Generate specific tests based on a description (e.g., `--describe "test that users can complete checkout flow"`)

## Examples

```bash
# Generate tests for current PR changes
/generate-e2e-mocks --pr

# Generate tests for a specific page
/generate-e2e-mocks --page client/src/pages/checkout

# Generate tests based on a description
/generate-e2e-mocks --describe "test error handling when payment fails"

# Combine PR mode with description
/generate-e2e-mocks --pr --describe "test the new validation added to the form"
```

## Instructions

### Step 0: Parse Arguments and Determine Scope

First, parse `$ARGUMENTS` to determine the mode:

**Check for flags:**
- `--pr`: Set `PR_MODE=true`
- `--page <path>`: Set `TARGET_PAGE=<path>`
- `--component <path>`: Set `TARGET_COMPONENT=<path>`
- `--full`: Set `FULL_MODE=true`
- `--describe "<text>"`: Set `TEST_DESCRIPTION=<text>`

**If `--pr` mode:**

1. Get the list of changed files:
```bash
git diff main...HEAD --name-only
```

2. Get PR information for context:
```bash
gh pr view --json title,body,number 2>/dev/null || echo "No PR found, using branch diff"
```

3. Filter to only relevant test files (pages, components, services):
```bash
git diff main...HEAD --name-only | grep -E '\.(tsx?|jsx?)$' | grep -E '(pages|components|services)/'
```

4. Read the changed files to understand what functionality was modified

5. Check the git diff for specific changes:
```bash
git diff main...HEAD -- <file>
```

**If `--describe` is provided:**

Use the description to focus test generation on specific scenarios. The description should guide:
- Which user flows to test
- What error conditions to verify
- What edge cases to cover

Example: `--describe "test validation errors for the new zipcode field"` should focus on:
- Invalid zipcode formats
- Empty zipcode submission
- Valid zipcode acceptance
- Error message display

### Step 1: Run API Audit

Based on the mode, run the api-audit command to identify all API endpoints that need mocking:

**For full project:**
```
/api-audit --full
```

**For PR changes only:**
```
/api-audit --pr
```

**For specific page/component:**
Manually analyze the target file and its imports.

Review the generated `docs/api_calls_per_page_component.yaml` file (if using api-audit).

### Step 2: Identify Third-Party Client APIs

Search the codebase for third-party API integrations that need mocking:

**Common Third-Party APIs:**

1. **Stripe** - Payment processing
   - Look for: `@stripe/stripe-js`, `@stripe/react-stripe-js`, `loadStripe`
   - Mock: `stripe.elements()`, `stripe.confirmPayment()`, `stripe.createToken()`
   - Intercept: `https://api.stripe.com/*`, `https://js.stripe.com/*`

2. **RevenueCat** - Subscription management
   - Look for: `@revenuecat/purchases-js`, `Purchases.configure`
   - Mock: `Purchases.getOfferings()`, `Purchases.purchasePackage()`, `Purchases.getCustomerInfo()`

3. **PubNub** - Real-time messaging
   - Look for: `pubnub`, `PubNub`
   - Mock: `pubnub.subscribe()`, `pubnub.publish()`, `pubnub.addListener()`

4. **Firebase** - Auth, Firestore, Analytics
   - Look for: `firebase/*`, `@firebase/*`
   - Mock: Auth methods, Firestore queries

5. **Google Maps** - Location services
   - Look for: `@react-google-maps/api`, `google.maps`
   - Intercept: `https://maps.googleapis.com/*`

6. **Sentry** - Error tracking
   - Look for: `@sentry/react`, `@sentry/browser`
   - Mock: `Sentry.init()`, `Sentry.captureException()`

Search for these integrations:
```bash
grep -r "from '@stripe\|from 'stripe\|loadStripe" client/src/
grep -r "from '@revenuecat\|from 'revenuecat\|Purchases" client/src/
grep -r "from 'pubnub\|from '@pubnub\|PubNub" client/src/
grep -r "from 'firebase\|from '@firebase" client/src/
grep -r "googleapis\|google.maps" client/src/
```

### Step 3: Create Mock Utilities File

Create or update `client/tests/e2e/specs/utils/mock-helpers.ts`:

```typescript
import { Page } from '@playwright/test'
// Use require for CommonJS runtime compatibility
const { faker } = require('@faker-js/faker')

// Generic API mock helper
export async function mockApiRoute(
  page: Page,
  pattern: string,
  response: object,
  options?: {
    status?: number
    delay?: number
    method?: 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE'
  }
) {
  await page.route(pattern, async (route) => {
    if (options?.method && route.request().method() !== options.method) {
      return route.continue()
    }

    if (options?.delay) {
      await new Promise(resolve => setTimeout(resolve, options.delay))
    }

    await route.fulfill({
      status: options?.status ?? 200,
      contentType: 'application/json',
      body: JSON.stringify(response),
    })
  })
}

// Mock paginated list response
export function createPaginatedResponse<T>(
  results: T[],
  options?: { count?: number; next?: string | null; previous?: string | null }
) {
  return {
    count: options?.count ?? results.length,
    next: options?.next ?? null,
    previous: options?.previous ?? null,
    results,
  }
}

// Stripe mocks
export async function mockStripe(page: Page) {
  await page.route('**/js.stripe.com/**', route => route.fulfill({
    status: 200,
    contentType: 'application/javascript',
    body: 'window.Stripe = function() { return { elements: () => ({}) } }',
  }))

  await page.route('**/api.stripe.com/**', route => route.fulfill({
    status: 200,
    contentType: 'application/json',
    body: JSON.stringify({ id: faker.string.uuid() }),
  }))
}

// PubNub mocks
export async function mockPubNub(page: Page) {
  await page.addInitScript(() => {
    window.PubNub = class {
      subscribe() {}
      publish(_, callback) { callback({ timetoken: Date.now() }) }
      addListener() {}
      removeListener() {}
      unsubscribe() {}
    }
  })
}

// RevenueCat mocks
export async function mockRevenueCat(page: Page) {
  await page.addInitScript(() => {
    window.Purchases = {
      configure: () => Promise.resolve(),
      getOfferings: () => Promise.resolve({ current: null, all: {} }),
      getCustomerInfo: () => Promise.resolve({ entitlements: { active: {} } }),
      purchasePackage: () => Promise.resolve({ customerInfo: {} }),
    }
  })
}

// Google Maps mocks
export async function mockGoogleMaps(page: Page) {
  await page.route('**/maps.googleapis.com/**', route => route.fulfill({
    status: 200,
    contentType: 'application/javascript',
    body: 'window.google = { maps: { Map: class {}, Marker: class {} } }',
  }))
}

// Block analytics/monitoring in tests
export async function blockAnalytics(page: Page) {
  const analyticsPatterns = [
    '**/sentry.io/**',
    '**/google-analytics.com/**',
    '**/googletagmanager.com/**',
    '**/newrelic.com/**',
    '**/nr-data.net/**',
    '**/segment.io/**',
    '**/amplitude.com/**',
    '**/mixpanel.com/**',
    '**/hotjar.com/**',
  ]

  for (const pattern of analyticsPatterns) {
    await page.route(pattern, route => route.abort())
  }
}
```

### Step 4: Create Fake Data Generators

Create or update `client/tests/e2e/specs/utils/fake-data.ts`:

> **IMPORTANT: snake_case for API Mocks**
>
> All mock data for endpoints that use the `tn-models` library **must use snake_case** keys. The `tn-models` library automatically converts snake_case responses from the API to camelCase for use in the frontend. When mocking API responses, you're simulating the raw API response, so use snake_case to match what the real API returns.

```typescript
// Use require for CommonJS runtime compatibility
const { faker } = require('@faker-js/faker')

// Set seed for reproducible tests (optional)
// faker.seed(12345)

// IMPORTANT: Use snake_case for all mock data - tn-models converts to camelCase
export const fakeUser = () => ({
  id: faker.string.uuid(),
  email: faker.internet.email(),
  first_name: faker.person.firstName(),
  last_name: faker.person.lastName(),
  created_at: faker.date.past().toISOString(),
  updated_at: faker.date.recent().toISOString(),
})

export const fakeProfile = (role: 'donor' | 'recipient' = 'donor') => ({
  id: faker.string.uuid(),
  role,
  baby_age: faker.helpers.arrayElement(['0-3', '3-6', '6-9', '9-12']),
  dietary_intake: [faker.string.uuid()],
  short_bio: faker.lorem.sentence(),
  sms_opt_in: faker.datatype.boolean(),
  latitude: faker.location.latitude(),
  longitude: faker.location.longitude(),
  zip_code: faker.location.zipCode('#####'),
  phone_number: faker.phone.number(),
  agreed_privacy_policy: true,
  agreed_guidelines: true,
  agreed_terms_of_use: true,
  profile_picture: null,
})

export const fakeAddress = () => ({
  street: faker.location.streetAddress(),
  city: faker.location.city(),
  state: faker.location.state({ abbreviated: true }),
  zip_code: faker.location.zipCode('#####'),
  country: 'US',
})

export const fakePayment = () => ({
  id: faker.string.uuid(),
  amount: faker.number.int({ min: 100, max: 10000 }),
  currency: 'usd',
  status: faker.helpers.arrayElement(['succeeded', 'pending', 'failed']),
  created_at: faker.date.recent().toISOString(),
})

// Add more generators based on your models
// Remember: always use snake_case keys to match API response format
```

### Step 5: Generate Test File Structure

For each page/component, create a test file following this pattern:

```typescript
import { test, expect } from '@playwright/test'
import {
  mockApiRoute,
  createPaginatedResponse,
  blockAnalytics,
  mockStripe, // if needed
} from './utils/mock-helpers'
import { fakeUser, fakeProfile } from './utils/fake-data'

test.describe('[Feature Name] (Mocked)', () => {
  test.beforeEach(async ({ page }) => {
    // Block analytics
    await blockAnalytics(page)

    // Mock auth/user endpoint
    await mockApiRoute(page, '**/api/auth/user/**', {
      user: fakeUser(),
    })

    // Mock any third-party services used
    // await mockStripe(page)
  })

  test('should [expected behavior]', async ({ page }) => {
    // Arrange: Set up specific mocks for this test
    const mockData = fakeProfile()
    await mockApiRoute(page, '**/api/profiles/**', mockData)

    // Act: Navigate and interact
    await page.goto('/your-page')
    await page.getByRole('button', { name: 'Submit' }).click()

    // Assert: Verify expected outcomes
    await expect(page.getByText(/success/i)).toBeVisible()
  })

  test('handles error states', async ({ page }) => {
    // Mock error response
    await mockApiRoute(page, '**/api/endpoint/**',
      { detail: 'Server error' },
      { status: 500 }
    )

    await page.goto('/your-page')

    // Verify error handling
    await expect(page.getByText(/error/i)).toBeVisible()
  })

  test('handles loading states', async ({ page }) => {
    // Mock slow response
    await mockApiRoute(page, '**/api/endpoint/**',
      { data: 'success' },
      { delay: 2000 }
    )

    await page.goto('/your-page')

    // Verify loading indicator appears
    await expect(page.getByRole('progressbar')).toBeVisible()
  })
})
```

### Step 6: Mock Patterns by API Type

**REST API Mocks:**
```typescript
// List endpoint
await mockApiRoute(page, '**/api/items/', createPaginatedResponse([
  { id: '1', name: 'Item 1' },
  { id: '2', name: 'Item 2' },
]))

// Retrieve endpoint
await mockApiRoute(page, '**/api/items/*/'), { id: '1', name: 'Item 1' })

// Create endpoint
await mockApiRoute(page, '**/api/items/',
  { id: faker.string.uuid(), name: 'New Item' },
  { method: 'POST', status: 201 }
)

// Update endpoint
await mockApiRoute(page, '**/api/items/*/',
  { id: '1', name: 'Updated Item' },
  { method: 'PATCH' }
)

// Delete endpoint
await mockApiRoute(page, '**/api/items/*/',
  {},
  { method: 'DELETE', status: 204 }
)
```

**GraphQL Mocks:**
```typescript
await page.route('**/graphql', async (route) => {
  const request = route.request()
  const postData = JSON.parse(request.postData() || '{}')

  if (postData.operationName === 'GetUser') {
    return route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify({
        data: { user: fakeUser() }
      }),
    })
  }

  return route.continue()
})
```

### Step 7: Test Naming Conventions

Use descriptive test names that explain:
- The scenario being tested
- The expected outcome

```typescript
test('displays error toast when profile creation fails with 500 error', ...)
test('redirects to dashboard after successful login', ...)
test('shows validation error for invalid email format', ...)
test('loads more items when scrolling to bottom of list', ...)
```

### Step 8: Coverage Checklist

For each page/feature, ensure tests cover:

- [ ] Happy path - successful flow
- [ ] Error states - API errors (400, 401, 403, 404, 500)
- [ ] Loading states - spinner/skeleton visibility
- [ ] Empty states - no data scenarios
- [ ] Validation errors - form validation
- [ ] Edge cases - boundary conditions
- [ ] Accessibility - keyboard navigation, screen reader
- [ ] Responsive - different viewport sizes

### Output

Generate tests in `client/tests/e2e/specs/` following the project's existing structure:
- One test file per page/major feature
- Shared utilities in `utils/` subdirectory
- Use TypeScript for type safety

## Tips

1. **Use Network Tab**: Run the app and use browser dev tools Network tab to see actual API calls and response shapes

2. **Match Response Shapes**: Ensure mock responses match the actual API response structure (check OpenAPI schema or service models)

3. **Faker Consistency**: Use `faker.seed()` if you need reproducible test data

4. **Intercept Order**: Set up route interceptions BEFORE navigating to the page

5. **Wildcard Patterns**: Use `**` for any path segment, `*` for single segment
   - `**/api/users/**` matches `/api/users/123/profile`
   - `**/api/users/*` matches `/api/users/123` but not `/api/users/123/profile`

6. **Debug Mocks**: Add console logs in route handlers during development:
   ```typescript
   await page.route('**/api/**', route => {
     console.log('Intercepted:', route.request().url())
     route.continue()
   })
   ```

7. **snake_case for tn-models**: All mock API responses must use snake_case keys (e.g., `first_name`, `created_at`). The `tn-models` library automatically converts these to camelCase when consuming the response. If your mocks use camelCase, the data won't be transformed correctly.

## PR-Based Test Generation

When using `--pr` mode, follow this workflow:

### 1. Analyze the PR Changes

```bash
# Get changed files
git diff main...HEAD --name-only

# Get the actual diff to understand what changed
git diff main...HEAD

# Get PR title and description for context
gh pr view --json title,body
```

### 2. Categorize Changes

Group changes by type:
- **New features**: Need comprehensive happy path + error tests
- **Bug fixes**: Need regression tests for the specific bug
- **Validation changes**: Need boundary testing for new rules
- **UI changes**: Need visual/interaction tests
- **API changes**: Need mock updates + integration tests

### 3. Generate Targeted Tests

For each category, generate appropriate tests:

**New Feature Example:**
```typescript
test.describe('New Checkout Flow (PR #123)', () => {
  test('completes checkout with valid payment', async ({ page }) => { ... })
  test('shows error when payment fails', async ({ page }) => { ... })
  test('validates required fields before submission', async ({ page }) => { ... })
})
```

**Bug Fix Example:**
```typescript
test.describe('Zipcode Validation Fix (PR #173)', () => {
  test('prevents proceeding with invalid zipcode', async ({ page }) => {
    // Regression test for the specific bug that was fixed
  })
})
```

**Validation Change Example:**
```typescript
test.describe('Name Field Validation (PR #180)', () => {
  test('accepts valid names with letters only', async ({ page }) => { ... })
  test('accepts names with hyphens and apostrophes', async ({ page }) => { ... })
  test('rejects names with numbers', async ({ page }) => { ... })
  test('rejects names with special characters', async ({ page }) => { ... })
})
```

### 4. Reference PR in Test Names

Include PR reference in test describe blocks for traceability:
```typescript
test.describe('Feature Name (PR #123)', () => { ... })
```

## Description-Based Test Generation

When using `--describe`, interpret the user's intent:

| Description | Focus Areas |
|-------------|-------------|
| "test login flow" | Auth endpoints, form validation, redirects, error messages |
| "test payment errors" | Payment API failures, error toasts, retry logic |
| "test form validation" | Required fields, format validation, error display |
| "test loading states" | Slow API responses, spinners, skeleton screens |
| "test empty states" | No data scenarios, empty list messages |

### Combining --pr and --describe

When both flags are used:
1. First, scope to files changed in the PR
2. Then, focus on the specific scenarios from the description
3. Generate tests that are both relevant to the PR AND match the description

Example: `/generate-e2e-mocks --pr --describe "test error handling"`
- Look at changed files in PR
- Focus only on error handling scenarios within those changes
- Generate tests for error states, error messages, error recovery
