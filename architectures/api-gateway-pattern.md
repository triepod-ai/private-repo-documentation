# API Gateway Pattern

## Overview

Centralized backend service acting as a single entry point for multiple microservices, handling authentication, routing, rate limiting, and service orchestration.

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                   Client Applications                    │
│  (Web, Mobile, Desktop)                                  │
└────────────────────────┬─────────────────────────────────┘
                         │
                         │ HTTP/WebSocket
                         │
┌────────────────────────▼─────────────────────────────────┐
│                    API Gateway                           │
│  ┌────────────┐  ┌─────────────┐  ┌────────────────┐   │
│  │   Auth     │  │    Rate     │  │    Request     │   │
│  │ Middleware │  │   Limiting  │  │    Logging     │   │
│  └────────────┘  └─────────────┘  └────────────────┘   │
│                                                          │
│  ┌────────────────────────────────────────────────┐    │
│  │            Route Handler                        │    │
│  │  • /api/users/*                                │    │
│  │  • /api/payments/*                             │    │
│  │  • /api/content/*                              │    │
│  │  • /api/admin/*                                │    │
│  └────────────────────────────────────────────────┘    │
└─────────────┬──────────────┬──────────────┬─────────────┘
              │              │              │
    ┌─────────▼──────┐ ┌────▼─────┐ ┌──────▼──────┐
    │ User Service   │ │ Payment  │ │   Content   │
    │                │ │ Service  │ │   Service   │
    └────────────────┘ └──────────┘ └─────────────┘
```

## Core Implementation

```typescript
// server.ts

import express from 'express'
import cors from 'cors'
import helmet from 'helmet'
import { authMiddleware } from './middleware/auth'
import { rateLimiter } from './middleware/rate-limit'
import { logger } from './middleware/logger'

export function createServer() {
  const app = express()

  // Security middleware
  app.use(helmet())
  app.use(cors({
    origin: process.env.ALLOWED_ORIGINS?.split(','),
    credentials: true
  }))

  // Body parsing
  app.use(express.json())
  app.use(express.urlencoded({ extended: true }))

  // Request logging
  app.use(logger)

  // Global rate limiting
  app.use(rateLimiter)

  // Health check
  app.get('/health', (req, res) => {
    res.json({ status: 'ok', timestamp: new Date().toISOString() })
  })

  // Authentication routes (public)
  app.use('/api/auth', authRoutes)

  // Protected routes
  app.use('/api/users', authMiddleware, userRoutes)
  app.use('/api/payments', authMiddleware, paymentRoutes)
  app.use('/api/admin', authMiddleware, requireRole('ADMIN'), adminRoutes)

  // WebSocket support
  const server = http.createServer(app)
  const io = new Server(server, {
    cors: { origin: process.env.ALLOWED_ORIGINS?.split(',') }
  })

  io.use(socketAuthMiddleware)
  io.on('connection', handleSocketConnection)

  return server
}
```

## Key Features

### 1. Request Routing

```typescript
// routes/index.ts

import { Router } from 'express'

const router = Router()

// User management
router.use('/users', userRoutes)
router.get('/users/:id', getUserController)
router.patch('/users/:id', updateUserController)

// Payments
router.use('/payments', paymentRoutes)
router.post('/payments/intent', createPaymentIntentController)
router.post('/payments/webhook', handleWebhookController)

// Content
router.use('/content', contentRoutes)
router.get('/content/posts', listPostsController)
router.post('/content/posts', createPostController)

export default router
```

### 2. Authentication Middleware

```typescript
// middleware/auth.ts

export async function authMiddleware(
  req: Request,
  res: Response,
  next: NextFunction
) {
  const token = req.headers.authorization?.replace('Bearer ', '')

  if (!token) {
    return res.status(401).json({ error: 'No token provided' })
  }

  try {
    const payload = await verifyAccessToken(token)
    req.user = await prisma.user.findUnique({
      where: { id: payload.userId }
    })

    if (!req.user) {
      return res.status(401).json({ error: 'User not found' })
    }

    next()
  } catch (error) {
    return res.status(401).json({ error: 'Invalid token' })
  }
}

export function requireRole(...roles: UserRole[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user || !roles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Insufficient permissions' })
    }
    next()
  }
}
```

### 3. Rate Limiting

```typescript
// middleware/rate-limit.ts

import rateLimit from 'express-rate-limit'
import RedisStore from 'rate-limit-redis'
import Redis from 'ioredis'

const redis = new Redis(process.env.REDIS_URL)

export const rateLimiter = rateLimit({
  store: new RedisStore({
    client: redis,
    prefix: 'rate_limit:'
  }),
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // 100 requests per window
  message: 'Too many requests, please try again later'
})

// Stricter limits for authentication endpoints
export const authRateLimiter = rateLimit({
  store: new RedisStore({ client: redis, prefix: 'rate_limit:auth:' }),
  windowMs: 15 * 60 * 1000,
  max: 5, // Only 5 login attempts per 15 minutes
  message: 'Too many login attempts'
})
```

### 4. Service Proxy

```typescript
// lib/service-proxy.ts

export async function proxyToService(
  serviceName: string,
  endpoint: string,
  options: RequestInit
): Promise<Response> {
  const serviceUrls = {
    content: process.env.CONTENT_SERVICE_URL,
    payment: process.env.PAYMENT_SERVICE_URL,
    notification: process.env.NOTIFICATION_SERVICE_URL
  }

  const baseUrl = serviceUrls[serviceName]
  const url = `${baseUrl}${endpoint}`

  return fetch(url, {
    ...options,
    headers: {
      ...options.headers,
      'X-Service-Token': process.env.SERVICE_TOKEN
    }
  })
}

// Usage
app.post('/api/content/process', authMiddleware, async (req, res) => {
  const response = await proxyToService(
    'content',
    '/process',
    {
      method: 'POST',
      body: JSON.stringify(req.body)
    }
  )

  const data = await response.json()
  res.json(data)
})
```

## OpenAPI Documentation

```typescript
// swagger.ts

import swaggerJSDoc from 'swagger-jsdoc'
import swaggerUi from 'swagger-ui-express'

const options = {
  definition: {
    openapi: '3.0.0',
    info: {
      title: 'API Gateway',
      version: '1.0.0',
      description: 'Centralized API gateway for all services'
    },
    servers: [
      { url: 'http://localhost:4000', description: 'Development' },
      { url: 'https://api.production.com', description: 'Production' }
    ],
    components: {
      securitySchemes: {
        bearerAuth: {
          type: 'http',
          scheme: 'bearer',
          bearerFormat: 'JWT'
        }
      }
    }
  },
  apis: ['./routes/*.ts']
}

const swaggerSpec = swaggerJSDoc(options)

app.use('/docs', swaggerUi.serve, swaggerUi.setup(swaggerSpec))
```

## WebSocket Support

```typescript
// websocket/handler.ts

import { Server, Socket } from 'socket.io'

export function handleSocketConnection(socket: Socket) {
  console.log('Client connected:', socket.id)

  // Join user-specific room
  if (socket.data.user) {
    socket.join(`user:${socket.data.user.id}`)
  }

  socket.on('subscribe:notifications', () => {
    socket.join('notifications')
  })

  socket.on('disconnect', () => {
    console.log('Client disconnected:', socket.id)
  })
}

// Emit to specific user
export function notifyUser(userId: string, event: string, data: any) {
  io.to(`user:${userId}`).emit(event, data)
}
```

## Error Handling

```typescript
// middleware/error-handler.ts

export function errorHandler(
  err: Error,
  req: Request,
  res: Response,
  next: NextFunction
) {
  console.error(err)

  if (err instanceof ValidationError) {
    return res.status(400).json({
      error: 'Validation failed',
      details: err.details
    })
  }

  if (err instanceof UnauthorizedError) {
    return res.status(401).json({ error: 'Unauthorized' })
  }

  res.status(500).json({
    error: 'Internal server error',
    message: process.env.NODE_ENV === 'development' ? err.message : undefined
  })
}

// Use as final middleware
app.use(errorHandler)
```

## Related Patterns

- [Unified Ecosystem](unified-ecosystem.md)
- [Authentication Systems](../patterns/authentication-systems.md)

---

**Production Impact**: Single entry point handling 1000+ req/s with comprehensive authentication, rate limiting, and service orchestration.
