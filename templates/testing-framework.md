# Testing Framework Template

## Purpose

Template for establishing comprehensive testing frameworks covering unit, integration, regression, and E2E testing strategies.

## Template Structure

```markdown
# [Project Name] Testing Framework

**Coverage Target**: 80%+
**Test Count**: [Number]+ tests
**CI/CD Integration**: [Yes/No]

## Table of Contents

1. [Overview](#overview)
2. [Test Structure](#test-structure)
3. [Unit Testing](#unit-testing)
4. [Integration Testing](#integration-testing)
5. [Regression Testing](#regression-testing)
6. [End-to-End Testing](#end-to-end-testing)
7. [Configuration](#configuration)
8. [Running Tests](#running-tests)
9. [Writing New Tests](#writing-new-tests)
10. [Best Practices](#best-practices)

---

## Overview

### Testing Philosophy

[Describe the team's approach to testing]

### Test Pyramid

\```
              ▲
             ╱ ╲
            ╱E2E ╲            ~[X] tests
           ╱───────╲
          ╱         ╲
         ╱Integration╲       ~[Y] tests
        ╱─────────────╲
       ╱               ╲
      ╱  Unit + Comp.   ╲   ~[Z] tests
     ╱───────────────────╲
\```

### Coverage Goals

- **Unit Tests**: 85%+ coverage
- **Integration Tests**: Critical paths covered
- **E2E Tests**: User journeys covered
- **Regression Tests**: Known issues prevented

---

## Test Structure

### Directory Organization

\```
project/
├── __tests__/
│   ├── unit/              # Pure function tests
│   ├── components/        # React component tests
│   ├── integration/       # Multi-layer tests
│   ├── regression/        # Prevent known issues
│   └── e2e/              # User flows
├── __mocks__/            # Shared mocks
├── __fixtures__/         # Test data
└── jest.config.ts        # Test configuration
\```

---

## Unit Testing

### Setup

\```typescript
// jest.setup.ts

import '@testing-library/jest-dom'

// Global test setup
beforeAll(() => {
  // Setup code
})

afterEach(() => {
  // Cleanup code
})
\```

### Testing Pure Functions

\```typescript
// __tests__/unit/utils.test.ts

import { functionName } from '@/lib/utils'

describe('functionName', () => {
  test('handles normal input', () => {
    const result = functionName(input)
    expect(result).toBe(expected)
  })

  test('handles edge cases', () => {
    expect(functionName(null)).toBeNull()
    expect(functionName('')).toBe('')
  })

  test('throws on invalid input', () => {
    expect(() => functionName(invalid)).toThrow(ErrorType)
  })
})
\```

### Testing React Components

\```typescript
// __tests__/components/button.test.tsx

import { render, screen, fireEvent } from '@testing-library/react'
import { Button } from '@/components/ui/button'

describe('Button', () => {
  test('renders with text', () => {
    render(<Button>Click me</Button>)
    expect(screen.getByRole('button')).toHaveTextContent('Click me')
  })

  test('calls onClick handler', () => {
    const handleClick = jest.fn()
    render(<Button onClick={handleClick}>Click</Button>)

    fireEvent.click(screen.getByRole('button'))
    expect(handleClick).toHaveBeenCalledTimes(1)
  })

  test('applies variant classes', () => {
    const { container } = render(<Button variant="destructive">Delete</Button>)
    expect(container.querySelector('button')).toHaveClass('bg-destructive')
  })
})
\```

---

## Integration Testing

### API Integration Tests

\```typescript
// __tests__/integration/api.test.ts

import request from 'supertest'
import { createServer } from '@/server'

describe('API Integration', () => {
  let app: Express
  let token: string

  beforeAll(() => {
    app = createServer()
  })

  test('POST /api/auth/login authenticates user', async () => {
    const response = await request(app)
      .post('/api/auth/login')
      .send({ email: 'test@example.com', password: 'password' })

    expect(response.status).toBe(200)
    expect(response.body.accessToken).toBeDefined()

    token = response.body.accessToken
  })

  test('GET /api/protected requires authentication', async () => {
    const response = await request(app)
      .get('/api/protected')
      .set('Authorization', `Bearer ${token}`)

    expect(response.status).toBe(200)
  })
})
\```

### Database Integration Tests

\```typescript
// __tests__/integration/database.test.ts

import { PrismaClient } from '@prisma/client'

const prisma = new PrismaClient()

describe('Database Integration', () => {
  beforeEach(async () => {
    // Clean database
    await prisma.$executeRaw`TRUNCATE TABLE "User" CASCADE`
  })

  afterAll(async () => {
    await prisma.$disconnect()
  })

  test('creates user with relationships', async () => {
    const user = await prisma.user.create({
      data: {
        email: 'test@example.com',
        profile: {
          create: { bio: 'Test bio' }
        }
      },
      include: { profile: true }
    })

    expect(user.profile).toBeDefined()
    expect(user.profile?.bio).toBe('Test bio')
  })
})
\```

---

## Regression Testing

### Regression Test Template

\```typescript
// __tests__/regression/issue-[number].test.ts

/**
 * Regression test for: [Issue Description]
 * Issue ID: #[number]
 * Date Fixed: [date]
 * Root Cause: [brief description]
 */

describe('Regression: [Issue Description]', () => {
  test('prevents [specific issue]', async () => {
    // Test that reproduces the original issue
    // Should pass with fix, would fail without it
  })
})
\```

### Example Regression Tests

\```typescript
// __tests__/regression/auth-bypass.test.ts

/**
 * Regression test for: Authentication bypass vulnerability
 * Issue ID: #CVE-2024-001
 * Date Fixed: 2024-12-01
 * Root Cause: Insufficient token validation
 */

describe('Regression: Auth Bypass', () => {
  test('rejects null tokens', async () => {
    const response = await request(app)
      .get('/api/admin')
      .set('Authorization', 'Bearer null')

    expect(response.status).toBe(401)
  })

  test('rejects expired tokens', async () => {
    const expiredToken = generateExpiredToken()
    const response = await request(app)
      .get('/api/admin')
      .set('Authorization', `Bearer ${expiredToken}`)

    expect(response.status).toBe(401)
  })
})
\```

---

## End-to-End Testing

### Playwright Configuration

\```typescript
// playwright.config.ts

import { defineConfig } from '@playwright/test'

export default defineConfig({
  testDir: './__tests__/e2e',
  use: {
    baseURL: 'http://localhost:3000',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure'
  },
  webServer: {
    command: 'npm run dev',
    port: 3000,
    reuseExistingServer: !process.env.CI
  }
})
\```

### E2E Test Example

\```typescript
// __tests__/e2e/checkout-flow.test.ts

import { test, expect } from '@playwright/test'

test.describe('Checkout Flow', () => {
  test('completes purchase', async ({ page }) => {
    // Navigate to product
    await page.goto('/products/item-1')

    // Add to cart
    await page.click('[data-testid="add-to-cart"]')
    await expect(page.locator('[data-testid="cart-count"]')).toHaveText('1')

    // Proceed to checkout
    await page.click('[data-testid="checkout"]')
    await expect(page).toHaveURL(/.*checkout/)

    // Fill form
    await page.fill('[name="email"]', 'test@example.com')
    await page.fill('[name="card"]', '4242424242424242')

    // Submit
    await page.click('[data-testid="submit"]')

    // Verify success
    await expect(page).toHaveURL(/.*success/)
    await expect(page.locator('h1')).toContainText('Thank you')
  })
})
\```

---

## Configuration

### Jest Configuration

\```typescript
// jest.config.ts

import type { Config } from 'jest'

const config: Config = {
  preset: 'ts-jest',
  testEnvironment: 'jsdom',
  setupFilesAfterEnv: ['<rootDir>/jest.setup.ts'],
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/$1'
  },
  collectCoverageFrom: [
    'app/**/*.{ts,tsx}',
    'components/**/*.{ts,tsx}',
    'lib/**/*.{ts,tsx}',
    '!**/*.d.ts'
  ],
  coverageThresholds: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    }
  }
}

export default config
\```

### CI/CD Integration

\```yaml
# .github/workflows/test.yml

name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: password
        ports:
          - 5433:5432

    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '20'

      - name: Install
        run: npm ci

      - name: Unit Tests
        run: npm run test:unit

      - name: Integration Tests
        run: npm run test:integration

      - name: E2E Tests
        run: npm run test:e2e

      - name: Upload Coverage
        uses: codecov/codecov-action@v3
\```

---

## Running Tests

### Quick Commands

\```bash
# Run all tests
npm test

# Run specific suite
npm run test:unit
npm run test:integration
npm run test:e2e
npm run test:regression

# Watch mode
npm run test:watch

# Coverage report
npm run test:coverage

# CI mode
npm run test:ci
\```

### Filtering Tests

\```bash
# Run specific file
npm test -- user.test.ts

# Run by pattern
npm test -- --testPathPattern=auth

# Run by test name
npm test -- --testNamePattern="login"
\```

---

## Writing New Tests

### Test Naming Convention

\```typescript
// ✅ GOOD: Descriptive test names
test('creates user with valid email', () => {})
test('rejects user with invalid email', () => {})
test('updates user profile successfully', () => {})

// ❌ BAD: Vague test names
test('user test 1', () => {})
test('it works', () => {})
\```

### Test Structure (AAA Pattern)

\```typescript
test('description', () => {
  // Arrange: Set up test data
  const user = createTestUser()

  // Act: Perform the action
  const result = processUser(user)

  // Assert: Verify the outcome
  expect(result.success).toBe(true)
})
\```

### Mock Best Practices

\```typescript
// __mocks__/stripe.ts

export const mockStripe = {
  paymentIntents: {
    create: jest.fn().mockResolvedValue({
      id: 'pi_test_123',
      status: 'succeeded'
    })
  }
}

// Usage in test
jest.mock('stripe', () => mockStripe)

test('creates payment', async () => {
  await createPayment(100)

  expect(mockStripe.paymentIntents.create).toHaveBeenCalledWith({
    amount: 100,
    currency: 'usd'
  })
})
\```

---

## Best Practices

### Test Independence

\```typescript
// ✅ GOOD: Each test is independent
describe('UserService', () => {
  beforeEach(async () => {
    await cleanDatabase()
  })

  test('test 1', async () => {
    // Fully self-contained
  })

  test('test 2', async () => {
    // Doesn't depend on test 1
  })
})

// ❌ BAD: Tests depend on each other
let userId: string

test('creates user', async () => {
  userId = await createUser() // Next test depends on this
})

test('updates user', async () => {
  await updateUser(userId) // Breaks if first test fails
})
\```

### Meaningful Assertions

\```typescript
// ✅ GOOD: Specific assertions
expect(response.status).toBe(200)
expect(response.body.user.email).toBe('test@example.com')
expect(response.body.accessToken).toMatch(/^[A-Za-z0-9-_]+\.[A-Za-z0-9-_]+/)

// ❌ BAD: Vague assertions
expect(response).toBeTruthy()
expect(response.body).toBeDefined()
\```

---

## Metrics & Monitoring

### Coverage Reports

View coverage:
\```bash
npm run test:coverage
open coverage/lcov-report/index.html
\```

### Test Performance

Monitor test duration:
\```bash
npm test -- --verbose --testTimeout=10000
\```

---

## Related Resources

- [Testing Library Documentation](https://testing-library.com/)
- [Jest Documentation](https://jestjs.io/)
- [Playwright Documentation](https://playwright.dev/)

---

**Maintained by**: [Team Name]
**Last Updated**: [Date]
```

## Checklist for Complete Testing Framework

- [ ] Jest configuration with TypeScript support
- [ ] Testing Library setup for React components
- [ ] Unit test structure and examples
- [ ] Integration test setup with database
- [ ] Regression test documentation
- [ ] E2E testing with Playwright
- [ ] CI/CD integration
- [ ] Coverage reporting
- [ ] Mock utilities and fixtures
- [ ] Testing best practices documented

## Related Templates

- [CLAUDE.md Template](claude-md-template.md)
- [Implementation Guide Template](implementation-guide.md)

---

**Purpose**: Establish comprehensive, maintainable testing frameworks that ensure code quality and prevent regressions.
