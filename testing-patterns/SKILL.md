---
name: testing-patterns
description: |
  Testing patterns with Vitest, React Testing Library, and Playwright.
  Use when writing unit tests, component tests, integration tests,
  or end-to-end tests for React/Next.js applications.
---

# Testing Patterns

Comprehensive testing patterns for modern React applications.

## Setup

### Vitest + React Testing Library

```bash
npm install -D vitest @vitejs/plugin-react jsdom
npm install -D @testing-library/react @testing-library/jest-dom @testing-library/user-event
```

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./vitest.setup.ts'],
    include: ['**/*.{test,spec}.{ts,tsx}'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html'],
    },
  },
})
```

```typescript
// vitest.setup.ts
import '@testing-library/jest-dom/vitest'
import { cleanup } from '@testing-library/react'
import { afterEach } from 'vitest'

afterEach(() => {
  cleanup()
})
```

## Unit Tests

### Testing Utilities

```typescript
// lib/utils.ts
export function formatPrice(cents: number): string {
  return `$${(cents / 100).toFixed(2)}`
}

export function slugify(text: string): string {
  return text
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, '-')
    .replace(/(^-|-$)/g, '')
}

// lib/utils.test.ts
import { describe, it, expect } from 'vitest'
import { formatPrice, slugify } from './utils'

describe('formatPrice', () => {
  it('formats cents to dollars', () => {
    expect(formatPrice(1000)).toBe('$10.00')
    expect(formatPrice(99)).toBe('$0.99')
    expect(formatPrice(0)).toBe('$0.00')
  })
})

describe('slugify', () => {
  it('converts text to slug', () => {
    expect(slugify('Hello World')).toBe('hello-world')
    expect(slugify('  Multiple   Spaces  ')).toBe('multiple-spaces')
    expect(slugify('Special!@#Characters')).toBe('special-characters')
  })
})
```

### Testing Hooks

```typescript
import { renderHook, act } from '@testing-library/react'
import { useCounter } from './useCounter'

describe('useCounter', () => {
  it('increments count', () => {
    const { result } = renderHook(() => useCounter(0))
    
    act(() => {
      result.current.increment()
    })
    
    expect(result.current.count).toBe(1)
  })
  
  it('accepts initial value', () => {
    const { result } = renderHook(() => useCounter(10))
    expect(result.current.count).toBe(10)
  })
})
```

## Component Tests

### Basic Component

```typescript
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { Button } from './Button'

describe('Button', () => {
  it('renders with text', () => {
    render(<Button>Click me</Button>)
    expect(screen.getByRole('button', { name: /click me/i })).toBeInTheDocument()
  })
  
  it('calls onClick when clicked', async () => {
    const user = userEvent.setup()
    const handleClick = vi.fn()
    
    render(<Button onClick={handleClick}>Click me</Button>)
    
    await user.click(screen.getByRole('button'))
    
    expect(handleClick).toHaveBeenCalledTimes(1)
  })
  
  it('is disabled when loading', () => {
    render(<Button loading>Submit</Button>)
    expect(screen.getByRole('button')).toBeDisabled()
  })
})
```

### Form Component

```typescript
import { render, screen, waitFor } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { LoginForm } from './LoginForm'

describe('LoginForm', () => {
  it('submits with valid data', async () => {
    const user = userEvent.setup()
    const onSubmit = vi.fn()
    
    render(<LoginForm onSubmit={onSubmit} />)
    
    await user.type(screen.getByLabelText(/email/i), 'test@example.com')
    await user.type(screen.getByLabelText(/password/i), 'password123')
    await user.click(screen.getByRole('button', { name: /sign in/i }))
    
    await waitFor(() => {
      expect(onSubmit).toHaveBeenCalledWith({
        email: 'test@example.com',
        password: 'password123',
      })
    })
  })
  
  it('shows validation errors', async () => {
    const user = userEvent.setup()
    
    render(<LoginForm onSubmit={vi.fn()} />)
    
    await user.click(screen.getByRole('button', { name: /sign in/i }))
    
    expect(screen.getByText(/email is required/i)).toBeInTheDocument()
    expect(screen.getByText(/password is required/i)).toBeInTheDocument()
  })
})
```

### Async Component

```typescript
import { render, screen, waitFor } from '@testing-library/react'
import { UserProfile } from './UserProfile'
import { server } from '../mocks/server'
import { http, HttpResponse } from 'msw'

describe('UserProfile', () => {
  it('displays user data after loading', async () => {
    render(<UserProfile userId="123" />)
    
    // Shows loading state
    expect(screen.getByText(/loading/i)).toBeInTheDocument()
    
    // Wait for data
    await waitFor(() => {
      expect(screen.getByText('John Doe')).toBeInTheDocument()
    })
  })
  
  it('handles error state', async () => {
    // Override handler for this test
    server.use(
      http.get('/api/users/:id', () => {
        return HttpResponse.json({ error: 'Not found' }, { status: 404 })
      })
    )
    
    render(<UserProfile userId="999" />)
    
    await waitFor(() => {
      expect(screen.getByText(/user not found/i)).toBeInTheDocument()
    })
  })
})
```

## Mocking

### MSW (API Mocking)

```typescript
// mocks/handlers.ts
import { http, HttpResponse } from 'msw'

export const handlers = [
  http.get('/api/users/:id', ({ params }) => {
    return HttpResponse.json({
      id: params.id,
      name: 'John Doe',
      email: 'john@example.com',
    })
  }),
  
  http.post('/api/posts', async ({ request }) => {
    const body = await request.json()
    return HttpResponse.json({ id: '1', ...body }, { status: 201 })
  }),
]

// mocks/server.ts
import { setupServer } from 'msw/node'
import { handlers } from './handlers'

export const server = setupServer(...handlers)

// vitest.setup.ts
import { server } from './mocks/server'

beforeAll(() => server.listen())
afterEach(() => server.resetHandlers())
afterAll(() => server.close())
```

### Module Mocking

```typescript
import { vi } from 'vitest'

// Mock module
vi.mock('@/lib/analytics', () => ({
  track: vi.fn(),
  identify: vi.fn(),
}))

// Mock next/navigation
vi.mock('next/navigation', () => ({
  useRouter: () => ({
    push: vi.fn(),
    back: vi.fn(),
  }),
  usePathname: () => '/test',
  useSearchParams: () => new URLSearchParams(),
}))

// Spy on existing function
const consoleSpy = vi.spyOn(console, 'error').mockImplementation(() => {})
```

## Playwright E2E Tests

```bash
npm install -D @playwright/test
npx playwright install
```

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test'

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
    { name: 'webkit', use: { ...devices['Desktop Safari'] } },
  ],
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
})
```

### E2E Test Examples

```typescript
// e2e/auth.spec.ts
import { test, expect } from '@playwright/test'

test.describe('Authentication', () => {
  test('user can sign in', async ({ page }) => {
    await page.goto('/login')
    
    await page.fill('[name="email"]', 'test@example.com')
    await page.fill('[name="password"]', 'password123')
    await page.click('button[type="submit"]')
    
    await expect(page).toHaveURL('/dashboard')
    await expect(page.locator('h1')).toContainText('Dashboard')
  })
  
  test('shows error for invalid credentials', async ({ page }) => {
    await page.goto('/login')
    
    await page.fill('[name="email"]', 'wrong@example.com')
    await page.fill('[name="password"]', 'wrongpassword')
    await page.click('button[type="submit"]')
    
    await expect(page.locator('[role="alert"]')).toContainText('Invalid credentials')
  })
})
```

### Page Object Model

```typescript
// e2e/pages/LoginPage.ts
import { Page, Locator, expect } from '@playwright/test'

export class LoginPage {
  readonly page: Page
  readonly emailInput: Locator
  readonly passwordInput: Locator
  readonly submitButton: Locator
  readonly errorMessage: Locator
  
  constructor(page: Page) {
    this.page = page
    this.emailInput = page.locator('[name="email"]')
    this.passwordInput = page.locator('[name="password"]')
    this.submitButton = page.locator('button[type="submit"]')
    this.errorMessage = page.locator('[role="alert"]')
  }
  
  async goto() {
    await this.page.goto('/login')
  }
  
  async login(email: string, password: string) {
    await this.emailInput.fill(email)
    await this.passwordInput.fill(password)
    await this.submitButton.click()
  }
  
  async expectError(message: string) {
    await expect(this.errorMessage).toContainText(message)
  }
}

// Usage in test
test('login flow', async ({ page }) => {
  const loginPage = new LoginPage(page)
  await loginPage.goto()
  await loginPage.login('test@example.com', 'password123')
  await expect(page).toHaveURL('/dashboard')
})
```

## Testing Server Components

```typescript
// For Server Components, test at the integration level
// or use the actual component output

import { render } from '@testing-library/react'

// Test client component wrapper
describe('UserProfileClient', () => {
  it('displays user data passed from server', () => {
    const user = { id: '1', name: 'John', email: 'john@test.com' }
    
    render(<UserProfileClient user={user} />)
    
    expect(screen.getByText('John')).toBeInTheDocument()
  })
})
```

## Best Practices

1. **Test behavior, not implementation** - Focus on what users see
2. **Use accessible queries** - `getByRole`, `getByLabelText`
3. **Mock at the network level** - Use MSW over mocking fetch
4. **Keep tests isolated** - Each test should be independent
5. **Use userEvent over fireEvent** - More realistic interactions
6. **Test error states** - Don't just test happy paths
7. **Use Page Object Model** - For maintainable E2E tests

## Query Priority

```typescript
// ✅ Preferred (accessible)
screen.getByRole('button', { name: /submit/i })
screen.getByLabelText(/email/i)
screen.getByText(/welcome/i)
screen.getByPlaceholderText(/search/i)

// ⚠️ Use when necessary
screen.getByAltText(/logo/i)
screen.getByTitle(/close/i)

// ❌ Last resort
screen.getByTestId('submit-button')
```
