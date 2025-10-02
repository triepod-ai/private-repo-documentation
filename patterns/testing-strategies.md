# Testing Strategies Pattern

## Overview

Comprehensive testing framework covering unit, integration, regression, and E2E testing across multiple applications in a monorepo. Features 185+ test suites with CI/CD integration.

## Testing Architecture

### Test Pyramid

```
                     ▲
                    ╱ ╲
                   ╱ E2E╲                ~10 tests
                  ╱───────╲               (Critical paths)
                 ╱         ╲
                ╱Integration╲            ~40 tests
               ╱─────────────╲           (API + DB + UI)
              ╱               ╲
             ╱  Unit + Comp.   ╲        ~135 tests
            ╱───────────────────╲       (Functions + Components)
           ╱─────────────────────╲
                  Fast → Slow
                 Many → Few
```

### Test Organization

```
project/
├── __tests__/
│   ├── unit/                   # Pure function tests
│   ├── components/             # React component tests
│   ├── integration/            # Multi-layer integration
│   ├── regression/             # Prevent known issues
│   └── e2e/                    # End-to-end flows
├── app-1/
│   ├── __tests__/             # App-specific tests
│   └── jest.config.ts
├── app-2/
│   ├── __tests__/
│   └── jest.config.ts
└── jest.config.ts             # Root configuration
```

## Unit Testing

### Pure Function Tests

```typescript
// lib/__tests__/utils.test.ts

import { formatCurrency, calculateTotal, validateEmail } from '../utils'

describe('Utils', () => {
  describe('formatCurrency', () => {
    test('formats USD correctly', () => {
      expect(formatCurrency(1234.56, 'USD')).toBe('$1,234.56')
    })

    test('handles zero', () => {
      expect(formatCurrency(0, 'USD')).toBe('$0.00')
    })

    test('handles negative values', () => {
      expect(formatCurrency(-100, 'USD')).toBe('-$100.00')
    })
  })

  describe('validateEmail', () => {
    test('accepts valid emails', () => {
      expect(validateEmail('test@example.com')).toBe(true)
      expect(validateEmail('user+tag@domain.co.uk')).toBe(true)
    })

    test('rejects invalid emails', () => {
      expect(validateEmail('invalid')).toBe(false)
      expect(validateEmail('@example.com')).toBe(false)
      expect(validateEmail('test@')).toBe(false)
    })
  })
})
```

### Business Logic Tests

```typescript
// lib/__tests__/subscription.test.ts

import { calculateSubscriptionPrice, isSubscriptionActive } from '../subscription'

describe('Subscription Logic', () => {
  test('applies discount for annual billing', () => {
    const monthly = calculateSubscriptionPrice({ plan: 'pro', billing: 'monthly' })
    const annual = calculateSubscriptionPrice({ plan: 'pro', billing: 'annual' })

    expect(annual).toBe(monthly * 12 * 0.8) // 20% discount
  })

  test('detects active subscriptions', () => {
    const activeSubscription = {
      status: 'active',
      currentPeriodEnd: new Date('2026-01-01')
    }

    const expiredSubscription = {
      status: 'active',
      currentPeriodEnd: new Date('2024-01-01')
    }

    expect(isSubscriptionActive(activeSubscription)).toBe(true)
    expect(isSubscriptionActive(expiredSubscription)).toBe(false)
  })
})
```

## Component Testing

### React Component Tests

```typescript
// components/__tests__/button.test.tsx

import { render, screen, fireEvent } from '@testing-library/react'
import { Button } from '../ui/button'

describe('Button Component', () => {
  test('renders children correctly', () => {
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
    const button = container.querySelector('button')
    expect(button).toHaveClass('bg-destructive')
  })

  test('disables interaction when disabled', () => {
    const handleClick = jest.fn()
    render(<Button onClick={handleClick} disabled>Disabled</Button>)

    const button = screen.getByRole('button')
    expect(button).toBeDisabled()

    fireEvent.click(button)
    expect(handleClick).not.toHaveBeenCalled()
  })
})
```

### Form Component Tests

```typescript
// components/__tests__/contact-form.test.tsx

import { render, screen, waitFor } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { ContactForm } from '../forms/contact-form'

describe('ContactForm', () => {
  test('validates required fields', async () => {
    render(<ContactForm onSubmit={jest.fn()} />)

    const submitButton = screen.getByText('Send Message')
    await userEvent.click(submitButton)

    expect(await screen.findByText('Name is required')).toBeInTheDocument()
    expect(await screen.findByText('Email is required')).toBeInTheDocument()
  })

  test('submits form with valid data', async () => {
    const handleSubmit = jest.fn()
    render(<ContactForm onSubmit={handleSubmit} />)

    await userEvent.type(screen.getByLabelText('Name'), 'John Doe')
    await userEvent.type(screen.getByLabelText('Email'), 'john@example.com')
    await userEvent.type(screen.getByLabelText('Message'), 'Hello!')

    await userEvent.click(screen.getByText('Send Message'))

    await waitFor(() => {
      expect(handleSubmit).toHaveBeenCalledWith({
        name: 'John Doe',
        email: 'john@example.com',
        message: 'Hello!'
      })
    })
  })
})
```

## Integration Testing

### API Integration Tests

```typescript
// __tests__/integration/auth-api.test.ts

import { createServer } from '../../server'
import request from 'supertest'

describe('Authentication API', () => {
  let app: Express
  let authToken: string

  beforeAll(() => {
    app = createServer()
  })

  test('POST /api/auth/signup creates new user', async () => {
    const response = await request(app)
      .post('/api/auth/signup')
      .send({
        email: 'test@example.com',
        password: 'SecurePass123!',
        name: 'Test User'
      })

    expect(response.status).toBe(201)
    expect(response.body.user.email).toBe('test@example.com')
    expect(response.body.accessToken).toBeDefined()

    authToken = response.body.accessToken
  })

  test('POST /api/auth/login authenticates user', async () => {
    const response = await request(app)
      .post('/api/auth/login')
      .send({
        email: 'test@example.com',
        password: 'SecurePass123!'
      })

    expect(response.status).toBe(200)
    expect(response.body.accessToken).toBeDefined()
  })

  test('GET /api/auth/me returns current user', async () => {
    const response = await request(app)
      .get('/api/auth/me')
      .set('Authorization', `Bearer ${authToken}`)

    expect(response.status).toBe(200)
    expect(response.body.email).toBe('test@example.com')
  })

  test('Protected route rejects unauthenticated requests', async () => {
    const response = await request(app)
      .get('/api/admin/dashboard')

    expect(response.status).toBe(401)
  })
})
```

### Database Integration Tests

```typescript
// __tests__/integration/database.test.ts

import { PrismaClient } from '@prisma/client'

const prisma = new PrismaClient()

describe('Database Integration', () => {
  beforeEach(async () => {
    // Clean database before each test
    await prisma.$executeRaw`TRUNCATE TABLE "User" CASCADE`
  })

  afterAll(async () => {
    await prisma.$disconnect()
  })

  test('creates user with relationships', async () => {
    const user = await prisma.user.create({
      data: {
        email: 'test@example.com',
        name: 'Test User',
        profile: {
          create: {
            bio: 'Test bio',
            location: 'Test location'
          }
        }
      },
      include: {
        profile: true
      }
    })

    expect(user.profile).toBeDefined()
    expect(user.profile?.bio).toBe('Test bio')
  })

  test('enforces unique email constraint', async () => {
    await prisma.user.create({
      data: { email: 'test@example.com', name: 'User 1' }
    })

    await expect(
      prisma.user.create({
        data: { email: 'test@example.com', name: 'User 2' }
      })
    ).rejects.toThrow()
  })
})
```

## Regression Testing

### Authentication Regression Tests

```typescript
// __tests__/regression/auth-regression.test.ts

describe('Authentication Regression Tests', () => {
  test('prevents authentication bypass (CVE-2024-001)', async () => {
    // Test for specific vulnerability
    const response = await request(app)
      .get('/api/admin/dashboard')
      .set('Authorization', 'Bearer null')

    expect(response.status).toBe(401)
  })

  test('enforces password requirements', async () => {
    const weakPasswords = ['12345678', 'password', 'abcdefgh']

    for (const password of weakPasswords) {
      const response = await request(app)
        .post('/api/auth/signup')
        .send({
          email: 'test@example.com',
          password
        })

      expect(response.status).toBe(400)
      expect(response.body.error).toContain('password')
    }
  })

  test('prevents token reuse after logout', async () => {
    // Login
    const loginResponse = await request(app)
      .post('/api/auth/login')
      .send({ email: 'test@example.com', password: 'SecurePass123!' })

    const token = loginResponse.body.accessToken

    // Logout
    await request(app)
      .post('/api/auth/logout')
      .set('Authorization', `Bearer ${token}`)

    // Attempt to use token after logout
    const response = await request(app)
      .get('/api/auth/me')
      .set('Authorization', `Bearer ${token}`)

    expect(response.status).toBe(401)
  })
})
```

### TypeScript Regression Tests

```typescript
// __tests__/regression/typescript.test.ts

import { execSync } from 'child_process'

describe('TypeScript Regression', () => {
  test('entire codebase type-checks without errors', () => {
    expect(() => {
      execSync('npm run type-check', {
        stdio: 'pipe',
        cwd: process.cwd()
      })
    }).not.toThrow()
  })

  test('strict mode enabled for all apps', () => {
    const tsConfigs = [
      'tsconfig.json',
      'app-1/tsconfig.json',
      'app-2/tsconfig.json'
    ]

    tsConfigs.forEach(config => {
      const content = require(`../../${config}`)
      expect(content.compilerOptions.strict).toBe(true)
    })
  })
})
```

## End-to-End Testing

### Critical User Flows

```typescript
// __tests__/e2e/checkout-flow.test.ts

import { test, expect } from '@playwright/test'

test.describe('Checkout Flow', () => {
  test('completes full purchase journey', async ({ page }) => {
    // Navigate to product
    await page.goto('http://localhost:3000/products/pro-plan')

    // Add to cart
    await page.click('[data-testid="add-to-cart"]')
    await expect(page.locator('[data-testid="cart-count"]')).toHaveText('1')

    // Proceed to checkout
    await page.click('[data-testid="checkout-button"]')
    await expect(page).toHaveURL(/.*checkout/)

    // Fill billing details
    await page.fill('[name="email"]', 'test@example.com')
    await page.fill('[name="cardNumber"]', '4242424242424242')
    await page.fill('[name="expiry"]', '12/26')
    await page.fill('[name="cvc"]', '123')

    // Submit payment
    await page.click('[data-testid="submit-payment"]')

    // Verify success
    await expect(page).toHaveURL(/.*success/)
    await expect(page.locator('h1')).toContainText('Thank you')
  })

  test('handles payment failure gracefully', async ({ page }) => {
    await page.goto('http://localhost:3000/checkout')

    // Use test card that always fails
    await page.fill('[name="cardNumber"]', '4000000000000002')
    await page.fill('[name="expiry"]', '12/26')
    await page.fill('[name="cvc"]', '123')

    await page.click('[data-testid="submit-payment"]')

    // Verify error message
    await expect(page.locator('[data-testid="error-message"]'))
      .toContainText('declined')

    // User can retry
    await expect(page.locator('[data-testid="submit-payment"]'))
      .toBeEnabled()
  })
})
```

## Test Configuration

### Jest Configuration

```typescript
// jest.config.ts

import type { Config } from 'jest'

const config: Config = {
  preset: 'ts-jest',
  testEnvironment: 'jsdom',
  setupFilesAfterEnv: ['<rootDir>/jest.setup.ts'],
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/$1',
    '^@/components/(.*)$': '<rootDir>/components/$1'
  },
  collectCoverageFrom: [
    'app/**/*.{ts,tsx}',
    'components/**/*.{ts,tsx}',
    'lib/**/*.{ts,tsx}',
    '!**/*.d.ts',
    '!**/node_modules/**'
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
```

### Test Setup

```typescript
// jest.setup.ts

import '@testing-library/jest-dom'
import { cleanup } from '@testing-library/react'
import { TextEncoder, TextDecoder } from 'util'

// Polyfills for Node environment
global.TextEncoder = TextEncoder
global.TextDecoder = TextDecoder as any

// Cleanup after each test
afterEach(() => {
  cleanup()
})

// Mock environment variables
process.env.DATABASE_URL = 'postgresql://test:test@localhost:5433/test'
process.env.JWT_SECRET = 'test-secret'
```

## CI/CD Integration

### GitHub Actions

```yaml
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
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5433:5432

    steps:
      - uses: actions/checkout@v3

      - uses: actions/setup-node@v3
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Generate Prisma Client
        run: npx prisma generate

      - name: Run unit tests
        run: npm run test:unit

      - name: Run integration tests
        run: npm run test:integration
        env:
          DATABASE_URL: postgresql://postgres:password@localhost:5433/test

      - name: Run E2E tests
        run: npm run test:e2e

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/coverage-final.json
```

## Test Scripts

```json
// package.json

{
  "scripts": {
    "test": "jest",
    "test:unit": "jest __tests__/unit",
    "test:components": "jest __tests__/components",
    "test:integration": "jest __tests__/integration",
    "test:regression": "jest __tests__/regression",
    "test:e2e": "playwright test",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "test:ci": "jest --ci --coverage --maxWorkers=2"
  }
}
```

## Best Practices

### 1. Test Independence

```typescript
// ✅ GOOD: Each test is independent
describe('User Service', () => {
  beforeEach(async () => {
    await prisma.user.deleteMany()
  })

  test('creates user', async () => {
    const user = await createUser({ email: 'test@example.com' })
    expect(user).toBeDefined()
  })

  test('finds user by email', async () => {
    await createUser({ email: 'test@example.com' })
    const user = await findUserByEmail('test@example.com')
    expect(user).toBeDefined()
  })
})

// ❌ BAD: Tests depend on each other
let userId: string

test('creates user', async () => {
  const user = await createUser({ email: 'test@example.com' })
  userId = user.id // Next test depends on this
})

test('updates user', async () => {
  await updateUser(userId, { name: 'Updated' }) // Breaks if first test fails
})
```

### 2. Meaningful Assertions

```typescript
// ✅ GOOD: Specific assertions
expect(response.status).toBe(200)
expect(response.body.user.email).toBe('test@example.com')
expect(response.body.accessToken).toMatch(/^[A-Za-z0-9-_]+\.[A-Za-z0-9-_]+\.[A-Za-z0-9-_]+$/)

// ❌ BAD: Vague assertions
expect(response).toBeTruthy()
expect(response.body).toBeDefined()
```

### 3. Test Organization

```typescript
// ✅ GOOD: Descriptive, hierarchical organization
describe('Payment Processing', () => {
  describe('Stripe Integration', () => {
    describe('Payment Intents', () => {
      test('creates payment intent with correct amount', () => {})
      test('handles insufficient funds error', () => {})
    })
  })

  describe('PayPal Integration', () => {
    describe('Order Creation', () => {
      test('creates order with line items', () => {})
    })
  })
})
```

## Performance Optimization

### Parallel Test Execution

```bash
# Run tests in parallel
jest --maxWorkers=4

# Run specific suites in parallel
npm run test:unit & npm run test:components & wait
```

### Test Isolation

```typescript
// Use test database per worker
process.env.DATABASE_URL = `postgresql://localhost:5433/test_${process.env.JEST_WORKER_ID}`
```

## Metrics & Monitoring

**Real-World Coverage**:
- 185+ test suites
- 80%+ code coverage
- <5 minute CI/CD pipeline
- Regression tests prevent recurring issues
- E2E tests cover critical user journeys

## Related Patterns

- [Monorepo Architecture](monorepo-architecture.md)
- [CI/CD Integration](../tech-stack/ci-cd-pipelines.md)

---

**Production Impact**: Comprehensive test coverage prevents regressions, enables confident refactoring, and maintains 99.9% uptime in production.
