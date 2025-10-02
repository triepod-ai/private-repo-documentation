# Unified Ecosystem Architecture

## Overview

Comprehensive monorepo architecture integrating multiple full-stack applications with shared infrastructure, unified database, and consistent development patterns.

## System Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                     Unified Ecosystem                            │
├──────────────┬──────────────────┬──────────────┬────────────────┤
│  Main App    │   API Gateway    │   Content    │  Additional    │
│  (Next.js)   │   (Express.js)   │   Manager    │  Services      │
│  Port 3000   │   Port 4000      │   Port 6001  │                │
└──────┬───────┴────────┬─────────┴──────┬───────┴────────┬────────┘
       │                │                │                │
       │                │                │                │
┌──────▼────────────────▼────────────────▼────────────────▼────────┐
│              Shared Infrastructure Layer                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  Components  │  │    Types     │  │   Utilities  │          │
│  │  (70+ UI)    │  │  (Shared)    │  │   (Common)   │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└───────────────────────────┬───────────────────────────────────────┘
                            │
┌───────────────────────────▼───────────────────────────────────────┐
│                   Data & Services Layer                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  PostgreSQL  │  │    Redis     │  │   Storage    │          │
│  │  (Prisma)    │  │   (Cache)    │  │   (S3/Local) │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└───────────────────────────────────────────────────────────────────┘
                            │
┌───────────────────────────▼───────────────────────────────────────┐
│                   External Services                               │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌──────────────┐       │
│  │ Stripe  │  │ PayPal  │  │ AdSense │  │  Auth (OAuth)│       │
│  └─────────┘  └─────────┘  └─────────┘  └──────────────┘       │
└───────────────────────────────────────────────────────────────────┘
```

## Component Applications

### Main Application (Next.js 15)

**Purpose**: User-facing website with SSR/SSG capabilities

**Key Features**:
- 127+ page production website
- App Router architecture
- Server and client components
- Dynamic routes and API routes
- Authentication integration
- Payment processing UI
- Blog system with MDX
- SEO optimization

**Tech Stack**:
- Next.js 15
- React 19
- TypeScript
- Tailwind CSS
- NextAuth.js

### API Gateway (Express.js)

**Purpose**: Centralized backend service orchestration

**Key Features**:
- RESTful API endpoints
- JWT authentication
- WebSocket support
- Service proxying
- Rate limiting
- Request logging
- OpenAPI/Swagger docs

**Tech Stack**:
- Express.js
- TypeScript
- JWT
- Socket.io
- OpenAPI

### Content Manager

**Purpose**: Content processing and blog management

**Key Features**:
- Blog post creation/editing
- MDX processing
- Image optimization
- JSX validation
- Content scheduling
- SEO metadata generation

**Tech Stack**:
- React
- TypeScript
- MDX
- Sharp (images)

## Data Architecture

### Unified Database Schema

```prisma
// prisma/schema.prisma

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

// Core user model shared across all apps
model User {
  id                String   @id @default(cuid())
  email             String   @unique
  name              String?
  role              Role     @default(USER)
  status            AccountStatus @default(ACTIVE)

  // Authentication
  password          String?
  emailVerified     DateTime?

  // Provider integration
  stripeCustomerId  String?
  paypalCustomerId  String?

  // Relationships (cross-app)
  payments          Payment[]
  subscriptions     Subscription[]
  blogPosts         BlogPost[]
  sessions          Session[]

  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt
}

// Payment model (used by main app and API gateway)
model Payment {
  id                    String   @id @default(cuid())
  userId                String
  user                  User     @relation(fields: [userId], references: [id])

  provider              PaymentProvider
  amount                Int
  currency              String
  status                PaymentStatus

  stripePaymentIntentId String?
  paypalOrderId         String?

  createdAt             DateTime @default(now())
  completedAt           DateTime?
}

// Content model (used by main app and content manager)
model BlogPost {
  id          String   @id @default(cuid())
  slug        String   @unique
  title       String
  excerpt     String?
  content     String
  published   Boolean  @default(false)

  authorId    String
  author      User     @relation(fields: [authorId], references: [id])

  tags        Tag[]

  publishedAt DateTime?
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
}
```

### Database Access Pattern

```typescript
// lib/prisma.ts - Singleton Prisma client

import { PrismaClient } from '@prisma/client'

const globalForPrisma = global as unknown as { prisma: PrismaClient }

export const prisma =
  globalForPrisma.prisma ||
  new PrismaClient({
    log: ['query', 'error', 'warn'],
  })

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma

// Usage in any application
import { prisma } from '@/lib/prisma'

const users = await prisma.user.findMany()
```

## Communication Patterns

### App-to-App Communication

```typescript
// Main App → API Gateway

// components/user-profile.tsx
async function fetchUserData() {
  const response = await fetch('http://localhost:4000/api/users/me', {
    headers: {
      'Authorization': `Bearer ${getAccessToken()}`
    }
  })

  return response.json()
}

// API Gateway → Content Manager
async function processBlogPost(postId: string) {
  const response = await fetch('http://localhost:6002/api/process', {
    method: 'POST',
    body: JSON.stringify({ postId })
  })

  return response.json()
}
```

### Shared Event Bus

```typescript
// lib/events.ts

import { EventEmitter } from 'events'

export const eventBus = new EventEmitter()

// Publisher (any app)
eventBus.emit('user.registered', {
  userId: 'user_123',
  email: 'test@example.com'
})

// Subscriber (any app)
eventBus.on('user.registered', async (data) => {
  await sendWelcomeEmail(data.email)
  await createStripeCustomer(data.userId)
})
```

## Deployment Strategy

### Development Environment

```bash
# Start all services
npm run dev:all

# Services running:
# - Main App: http://localhost:3000
# - API Gateway: http://localhost:4000
# - Content Manager: http://localhost:6001
# - PostgreSQL: localhost:5432
# - Redis: localhost:6379
```

### Production Deployment

```yaml
# docker-compose.yml

version: '3.8'

services:
  main-app:
    build: ./app
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}
    depends_on:
      - postgres
      - redis

  api-gateway:
    build: ./api-gateway
    ports:
      - "4000:4000"
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      - postgres

  content-manager:
    build: ./content-manager
    ports:
      - "6001:6001"
    environment:
      - DATABASE_URL=${DATABASE_URL}

  postgres:
    image: postgres:15
    environment:
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

## Scalability Considerations

### Horizontal Scaling

```
┌─────────────────────────────────────────────┐
│           Load Balancer                     │
└─────────────┬───────────────────────────────┘
              │
    ┌─────────┼─────────┐
    │         │         │
┌───▼────┐ ┌──▼─────┐ ┌▼────────┐
│Main App│ │Main App│ │Main App │
│Instance│ │Instance│ │Instance │
│   1    │ │   2    │ │   3     │
└────────┘ └────────┘ └─────────┘
```

### Database Optimization

- Read replicas for scaling queries
- Connection pooling (PgBouncer)
- Query optimization and indexing
- Caching layer (Redis)

### Caching Strategy

```typescript
// lib/cache.ts

import Redis from 'ioredis'

const redis = new Redis(process.env.REDIS_URL)

export async function getCached<T>(
  key: string,
  fallback: () => Promise<T>,
  ttl: number = 300
): Promise<T> {
  // Try cache first
  const cached = await redis.get(key)
  if (cached) {
    return JSON.parse(cached)
  }

  // Fallback to data source
  const data = await fallback()

  // Store in cache
  await redis.setex(key, ttl, JSON.stringify(data))

  return data
}

// Usage
const user = await getCached(
  `user:${userId}`,
  () => prisma.user.findUnique({ where: { id: userId } }),
  600 // 10 minutes
)
```

## Monitoring & Observability

### Application Metrics

```typescript
// lib/metrics.ts

import { Counter, Histogram } from 'prom-client'

export const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code']
})

export const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code']
})

// Usage in middleware
app.use((req, res, next) => {
  const start = Date.now()

  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000
    httpRequestDuration.observe(
      { method: req.method, route: req.path, status_code: res.statusCode },
      duration
    )
    httpRequestsTotal.inc({
      method: req.method,
      route: req.path,
      status_code: res.statusCode
    })
  })

  next()
})
```

### Health Checks

```typescript
// app/api/health/route.ts

export async function GET() {
  const checks = {
    database: await checkDatabase(),
    redis: await checkRedis(),
    externalAPIs: await checkExternalAPIs()
  }

  const healthy = Object.values(checks).every(c => c.status === 'ok')

  return Response.json(
    {
      status: healthy ? 'healthy' : 'unhealthy',
      timestamp: new Date().toISOString(),
      checks
    },
    { status: healthy ? 200 : 503 }
  )
}

async function checkDatabase() {
  try {
    await prisma.$queryRaw`SELECT 1`
    return { status: 'ok' }
  } catch (error) {
    return { status: 'error', message: error.message }
  }
}
```

## Security Architecture

### Defense in Depth

1. **Network Layer**: Firewall rules, DDoS protection
2. **Application Layer**: Input validation, CSRF protection
3. **Authentication**: JWT tokens, OAuth integration
4. **Authorization**: Role-based access control
5. **Data Layer**: Encrypted at rest, parameterized queries

### Content Security Policy

```typescript
// middleware.ts

export function middleware(request: Request) {
  const response = NextResponse.next()

  response.headers.set(
    'Content-Security-Policy',
    [
      "default-src 'self'",
      "script-src 'self' 'unsafe-inline' 'unsafe-eval' https://js.stripe.com https://www.paypal.com",
      "style-src 'self' 'unsafe-inline'",
      "img-src 'self' data: https:",
      "connect-src 'self' https://api.stripe.com"
    ].join('; ')
  )

  return response
}
```

## Migration Strategy

### Adding New Applications

```bash
# 1. Create new app directory
mkdir new-app
cd new-app

# 2. Initialize with package.json
npm init -y

# 3. Add to root scripts
# package.json
{
  "scripts": {
    "dev:new-app": "cd new-app && npm run dev",
    "dev:all": "concurrently \"npm run dev:main\" \"npm run dev:new-app\""
  }
}

# 4. Use shared components
import { Button } from '../../../components/ui'

# 5. Use unified database
import { prisma } from '@/lib/prisma'
```

## Best Practices

1. **Unified Dependencies**: Single version across all apps
2. **Shared Types**: TypeScript interfaces in common directory
3. **Consistent Patterns**: Same auth, error handling across apps
4. **Comprehensive Testing**: Integration tests across app boundaries
5. **Documentation**: Architecture decision records (ADRs)

## Related Resources

- [Monorepo Architecture Pattern](../patterns/monorepo-architecture.md)
- [API Gateway Pattern](api-gateway-pattern.md)
- [Testing Strategies](../patterns/testing-strategies.md)

---

**Production Scale**: Supports 127+ pages, 185+ tests, multi-application ecosystem with 99.9% uptime.
