# Monorepo Architecture Pattern

## Overview

A production-proven monorepo architecture managing multiple full-stack applications with shared infrastructure, unified dependency management, and consistent development workflows.

## Architecture Design

### High-Level Structure

```
unified-monorepo/
├── app-1/                      # Next.js main application
├── app-2/                      # Express.js API gateway
├── app-3/                      # Content management system
├── components/                 # Shared component library
│   ├── ui/                    # 70+ shared UI components
│   ├── payments/              # Payment provider components
│   └── advertising/           # Ad integration components
├── prisma/                    # Unified database schema
│   └── schema.prisma          # Single source of truth
├── scripts/                   # Cross-application tooling
├── package.json               # Root package management
└── CLAUDE.md                  # AI agent instructions
```

### Application Responsibilities

```
┌─────────────────────────────────────────────────────────────┐
│                    Unified Monorepo                         │
├─────────────────┬─────────────────┬─────────────────────────┤
│   Main App      │   API Gateway   │  Content Manager       │
│   (Port 3000)   │   (Port 4000)   │  (Port 6001/6002)      │
├─────────────────┼─────────────────┼─────────────────────────┤
│ • User-facing   │ • Auth service  │ • Content processing   │
│ • Static pages  │ • Data APIs     │ • Blog management      │
│ • SSR/SSG       │ • WebSocket     │ • Media handling       │
│ • Client auth   │ • Proxy layer   │ • JSX validation       │
└─────────────────┴─────────────────┴─────────────────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
┌───────▼────────┬────────▼────────┬────────▼────────┐
│ Shared         │ Unified         │  Common         │
│ Components     │ Database        │  Scripts        │
│ (React/UI)     │ (PostgreSQL)    │  (Node.js)      │
└────────────────┴─────────────────┴─────────────────┘
```

## Key Benefits

### 1. Dependency Management
- **Single npm install** for entire ecosystem
- **Unified version control** for all dependencies
- **Consistent tooling** across applications (ESLint, TypeScript, Jest)
- **Reduced duplication** through shared node_modules

### 2. Code Sharing
- **Component Library**: 70+ shared React components (Button, Card, Form, etc.)
- **Type Definitions**: Shared TypeScript interfaces and types
- **Utility Functions**: Common helpers and business logic
- **Configuration**: Unified Tailwind, ESLint, TypeScript configs

### 3. Development Workflow
- **Single repository clone** for all applications
- **Unified scripts** for dev, build, test, deploy
- **Consistent tooling** across team members
- **Simplified onboarding** with comprehensive documentation

### 4. Testing Strategy
- **Integrated test suites** across applications
- **Shared test utilities** and mocks
- **Cross-application testing** for integration scenarios
- **Unified CI/CD** pipeline

## Implementation Patterns

### Shared Component Import Pattern

```typescript
// ✅ CORRECT: Import from shared components directory
import { Button } from '../../../components/ui/button'
import { PaymentForm } from '../../../components/payments/payment-form'
import { AdSenseSlot } from '../../../components/advertising/adsense-slot'

// ❌ INCORRECT: Local component duplication
import { Button } from './components/ui/button'
```

### Database Schema Sharing

```prisma
// prisma/schema.prisma - Single source of truth

model User {
  id            String   @id @default(cuid())
  email         String   @unique
  // Shared across all applications
  payments      Payment[]
  subscriptions Subscription[]
  content       Content[]
}

model Payment {
  id        String @id @default(cuid())
  userId    String
  user      User   @relation(fields: [userId], references: [id])
  // Unified payment model for all apps
}
```

### Unified Scripts Configuration

```json
// package.json - Root level scripts
{
  "scripts": {
    "dev": "npm run dev:all",
    "dev:all": "concurrently \"npm run dev:app1\" \"npm run dev:app2\"",
    "dev:app1": "cd app-1 && npm run dev",
    "dev:app2": "cd app-2 && npm run dev",

    "build": "npm run build:all",
    "build:all": "npm run build:app1 && npm run build:app2",

    "test": "npm run test:all",
    "test:all": "concurrently \"npm run test:app1\" \"npm run test:app2\"",

    "lint": "node scripts/unified-lint.js",
    "db:generate": "npx prisma generate",
    "db:push": "npx prisma db push"
  }
}
```

## Development Workflow

### Starting the Ecosystem

```bash
# Clone repository
git clone <repo-url>
cd unified-monorepo

# Install all dependencies (single command)
npm install

# Generate Prisma client
npm run db:generate

# Start development servers for all apps
npm run dev:all

# Or start individual applications
npm run dev:app1  # Main application on port 3000
npm run dev:app2  # API Gateway on port 4000
npm run dev:app3  # Content Manager on ports 6001/6002
```

### Database Operations

```bash
# Update schema
vim prisma/schema.prisma

# Generate Prisma client (affects all apps)
npm run db:generate

# Push to development database
npm run db:push

# Open Prisma Studio
npx prisma studio
```

### Testing Workflow

```bash
# Run all tests across all applications
npm run test:all

# Run application-specific tests
npm run test:app1
npm run test:app2

# Run regression tests
npm run test:regression

# Run with coverage
npm run test:coverage
```

## Best Practices

### 1. Component Library Management

```typescript
// components/ui/index.ts - Central export point
export { Button } from './button'
export { Card } from './card'
export { Input } from './input'
// ... 67 more components

// Application usage
import { Button, Card, Input } from '../../../components/ui'
```

### 2. Environment Configuration

```
# Root .env for shared configuration
DATABASE_URL="postgresql://..."
REDIS_URL="redis://..."

# Application-specific .env.local files
app-1/.env.local     # NEXT_PUBLIC_* variables
app-2/.env.local     # API_* variables
app-3/.env.local     # CONTENT_* variables
```

### 3. TypeScript Configuration

```json
// tsconfig.json - Root configuration
{
  "compilerOptions": {
    "paths": {
      "@/components/*": ["components/*"],
      "@/lib/*": ["lib/*"],
      "@/types/*": ["types/*"]
    }
  }
}

// app-1/tsconfig.json - Extends root
{
  "extends": "../tsconfig.json",
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./*"]
    }
  }
}
```

### 4. Shared Types and Interfaces

```typescript
// types/payment.ts - Shared across all apps
export interface Payment {
  id: string
  userId: string
  amount: number
  currency: string
  provider: 'stripe' | 'paypal'
  status: 'pending' | 'completed' | 'failed'
}

// types/user.ts
export interface User {
  id: string
  email: string
  role: 'USER' | 'ADMIN' | 'SUPER_ADMIN'
}
```

## Common Challenges & Solutions

### Challenge 1: Dependency Version Conflicts

**Problem**: Different apps need different versions of the same package.

**Solution**: Use workspace features or evaluate if both versions are truly needed.

```json
// Option 1: Unified version (preferred)
{
  "dependencies": {
    "react": "^19.0.0"
  }
}

// Option 2: Application-specific when necessary
{
  "workspaces": ["app-1", "app-2", "app-3"]
}
```

### Challenge 2: Build Order Dependencies

**Problem**: App B depends on App A being built first.

**Solution**: Explicit build ordering in scripts.

```json
{
  "scripts": {
    "build:all": "npm run build:shared && npm run build:apps",
    "build:shared": "cd lib && npm run build",
    "build:apps": "npm run build:app1 && npm run build:app2"
  }
}
```

### Challenge 3: Database Migration Coordination

**Problem**: Schema changes affect multiple applications simultaneously.

**Solution**: Always run Prisma commands from root, comprehensive testing.

```bash
# ✅ CORRECT: Run from root
npm run db:generate  # Regenerates client for all apps

# ✅ CORRECT: Test all apps after schema changes
npm run test:all

# ❌ INCORRECT: Running from individual app
cd app-1 && npx prisma generate  # Only updates this app
```

## Deployment Strategy

### Production Build Process

```bash
# 1. Install dependencies
npm ci

# 2. Generate Prisma client
npm run db:generate

# 3. Build all applications
npm run build:all

# 4. Run production tests
npm run test:production

# 5. Deploy individual apps
npm run deploy:app1
npm run deploy:app2
npm run deploy:app3
```

### CI/CD Pipeline

```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci
      - run: npm run db:generate
      - run: npm run lint
      - run: npm run test:all
      - run: npm run build:all
```

## Performance Considerations

### Build Time Optimization

- **Parallel Builds**: Use `concurrently` for independent apps
- **Incremental Builds**: Only rebuild changed applications
- **Caching**: Leverage npm cache and build artifacts

### Development Performance

- **Selective Dev Servers**: Start only needed applications
- **Hot Module Replacement**: Per-application HMR configuration
- **Shared Dependencies**: Single node_modules reduces memory usage

## Metrics & Success Indicators

### Real-World Performance

- **127+ page production build**: ~2-3 minutes
- **185+ test suites**: Complete in <5 minutes
- **3-4 concurrent applications**: Smooth development experience
- **70+ shared components**: Zero duplication across apps

## Related Patterns

- [Shared Component Library](shared-component-library.md)
- [Unified Database Schema](../architectures/unified-ecosystem.md)
- [Testing Strategies](testing-strategies.md)

---

**Used in Production**: Multi-application ecosystem with Next.js, Express.js, and content processing systems serving production traffic with 99.9% uptime.
