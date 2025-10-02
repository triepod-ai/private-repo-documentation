# Multi-Tenant SaaS Architecture

## Overview

Scalable multi-tenant platform serving multiple business directories with shared infrastructure, isolated data, and per-tenant customization.

## Tenancy Model

```
┌──────────────────────────────────────────────────────────┐
│              Application Load Balancer                   │
└────────────────────────┬─────────────────────────────────┘
                         │
            ┌────────────┴────────────┐
            │                         │
┌───────────▼──────────┐  ┌──────────▼──────────┐
│  bestofbrandon.com   │  │  bestofgluckstadt   │
│  (Tenant 1)          │  │  .com (Tenant 2)    │
└───────────┬──────────┘  └──────────┬──────────┘
            │                         │
            └────────────┬────────────┘
                         │
┌────────────────────────▼─────────────────────────────────┐
│              Shared Application Layer                     │
│  • Route handlers  • Business logic  • API               │
└────────────────────────┬─────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────────┐
│            Tenant Isolation Layer                         │
│  • Tenant identification  • Data scoping                 │
└────────────────────────┬─────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────────┐
│              Shared Database with Tenant ID              │
│  • All tenants in one database                           │
│  • Tenant ID on every row                                │
│  • Row-level security                                    │
└──────────────────────────────────────────────────────────┘
```

## Tenant Identification

### Domain-Based Resolution

```typescript
// middleware/tenant-resolver.ts

export async function resolveTenant(req: Request) {
  const hostname = req.headers.get('host')

  // Look up tenant by domain
  const tenant = await prisma.tenant.findUnique({
    where: { domain: hostname },
    include: { settings: true }
  })

  if (!tenant) {
    throw new Error('Tenant not found')
  }

  // Attach to request
  req.tenant = tenant

  return tenant
}

// Usage in middleware
app.use(async (req, res, next) => {
  try {
    await resolveTenant(req)
    next()
  } catch (error) {
    res.status(404).send('Site not found')
  }
})
```

## Data Isolation

### Database Schema

```prisma
// prisma/schema.prisma

model Tenant {
  id        String   @id @default(cuid())
  name      String
  domain    String   @unique
  subdomain String?  @unique

  // All tenant data scoped by this ID
  businesses Business[]
  users      User[]
  settings   TenantSettings?

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([domain])
}

model Business {
  id          String  @id @default(cuid())
  tenantId    String
  tenant      Tenant  @relation(fields: [tenantId], references: [id])

  name        String
  description String
  phone       String?
  address     String
  category    String

  // Tenant-scoped data
  reviews     Review[]

  @@index([tenantId])
  @@index([tenantId, category])
}

model Review {
  id          String   @id @default(cuid())
  businessId  String
  business    Business @relation(fields: [businessId], references: [id])
  tenantId    String   // Denormalized for query performance

  rating      Int
  comment     String
  authorName  String

  @@index([businessId])
  @@index([tenantId])
}
```

### Query Scoping

```typescript
// lib/tenant-db.ts

export class TenantDatabase {
  constructor(private tenantId: string) {}

  // All queries automatically scoped to tenant
  async getBusinesses(filters?: BusinessFilters) {
    return prisma.business.findMany({
      where: {
        tenantId: this.tenantId,
        ...filters
      }
    })
  }

  async getBusiness(id: string) {
    return prisma.business.findFirst({
      where: {
        id,
        tenantId: this.tenantId // Ensure tenant owns this business
      }
    })
  }

  async createBusiness(data: CreateBusinessInput) {
    return prisma.business.create({
      data: {
        ...data,
        tenantId: this.tenantId
      }
    })
  }
}

// Usage
const tenantDb = new TenantDatabase(req.tenant.id)
const businesses = await tenantDb.getBusinesses({ category: 'restaurants' })
```

## Per-Tenant Customization

### Theme Configuration

```typescript
// lib/tenant-theme.ts

export interface TenantTheme {
  primaryColor: string
  secondaryColor: string
  logo: string
  favicon: string
  fonts: {
    heading: string
    body: string
  }
}

export async function getTenantTheme(tenantId: string): Promise<TenantTheme> {
  const settings = await prisma.tenantSettings.findUnique({
    where: { tenantId }
  })

  return {
    primaryColor: settings?.primaryColor || '#3B82F6',
    secondaryColor: settings?.secondaryColor || '#10B981',
    logo: settings?.logo || '/default-logo.png',
    favicon: settings?.favicon || '/default-favicon.ico',
    fonts: {
      heading: settings?.headingFont || 'Inter',
      body: settings?.bodyFont || 'Inter'
    }
  }
}

// Apply theme in layout
export default async function RootLayout({ children }) {
  const tenant = await getCurrentTenant()
  const theme = await getTenantTheme(tenant.id)

  return (
    <html>
      <head>
        <link rel="icon" href={theme.favicon} />
        <style>{`
          :root {
            --color-primary: ${theme.primaryColor};
            --color-secondary: ${theme.secondaryColor};
            --font-heading: ${theme.fonts.heading};
            --font-body: ${theme.fonts.body};
          }
        `}</style>
      </head>
      <body>{children}</body>
    </html>
  )
}
```

### Feature Flags

```typescript
// lib/tenant-features.ts

export interface TenantFeatures {
  reviews: boolean
  reservations: boolean
  onlineOrdering: boolean
  analytics: boolean
  customDomain: boolean
}

export async function getTenantFeatures(tenantId: string): Promise<TenantFeatures> {
  const tenant = await prisma.tenant.findUnique({
    where: { id: tenantId },
    include: { subscription: true }
  })

  const plan = tenant.subscription?.plan || 'free'

  // Feature availability by plan
  const features: Record<string, TenantFeatures> = {
    free: {
      reviews: true,
      reservations: false,
      onlineOrdering: false,
      analytics: false,
      customDomain: false
    },
    pro: {
      reviews: true,
      reservations: true,
      onlineOrdering: true,
      analytics: true,
      customDomain: true
    }
  }

  return features[plan]
}

// Usage in components
export async function BusinessPage({ params }) {
  const tenant = await getCurrentTenant()
  const features = await getTenantFeatures(tenant.id)

  return (
    <div>
      <BusinessInfo />
      {features.reviews && <Reviews />}
      {features.reservations && <ReservationForm />}
    </div>
  )
}
```

## Performance Optimization

### Tenant-Specific Caching

```typescript
// lib/tenant-cache.ts

export class TenantCache {
  private redis: Redis
  private tenantId: string

  constructor(tenantId: string) {
    this.tenantId = tenantId
    this.redis = new Redis(process.env.REDIS_URL)
  }

  private key(suffix: string): string {
    return `tenant:${this.tenantId}:${suffix}`
  }

  async getBusinesses(category?: string): Promise<Business[]> {
    const cacheKey = this.key(`businesses:${category || 'all'}`)
    const cached = await this.redis.get(cacheKey)

    if (cached) {
      return JSON.parse(cached)
    }

    const businesses = await prisma.business.findMany({
      where: {
        tenantId: this.tenantId,
        ...(category && { category })
      }
    })

    await this.redis.setex(cacheKey, 300, JSON.stringify(businesses))

    return businesses
  }

  async invalidate(pattern: string) {
    const keys = await this.redis.keys(this.key(pattern))
    if (keys.length) {
      await this.redis.del(...keys)
    }
  }
}
```

## Billing & Subscription Management

```typescript
// lib/tenant-billing.ts

export async function createTenantSubscription(
  tenantId: string,
  plan: 'free' | 'pro' | 'enterprise'
) {
  const prices = {
    free: 0,
    pro: 99,
    enterprise: 299
  }

  if (plan === 'free') {
    return prisma.subscription.create({
      data: {
        tenantId,
        plan,
        status: 'active'
      }
    })
  }

  // Create Stripe subscription for paid plans
  const tenant = await prisma.tenant.findUnique({
    where: { id: tenantId }
  })

  const subscription = await stripe.subscriptions.create({
    customer: tenant.stripeCustomerId,
    items: [{ price: process.env[`STRIPE_PRICE_${plan.toUpperCase()}`] }]
  })

  return prisma.subscription.create({
    data: {
      tenantId,
      plan,
      status: 'active',
      stripeSubscriptionId: subscription.id
    }
  })
}
```

## Tenant Onboarding

```typescript
// lib/tenant-onboarding.ts

export async function createTenant(data: CreateTenantInput) {
  // 1. Create tenant record
  const tenant = await prisma.tenant.create({
    data: {
      name: data.name,
      domain: data.domain,
      subdomain: data.subdomain
    }
  })

  // 2. Initialize settings with defaults
  await prisma.tenantSettings.create({
    data: {
      tenantId: tenant.id,
      primaryColor: '#3B82F6',
      secondaryColor: '#10B981'
    }
  })

  // 3. Create default subscription
  await createTenantSubscription(tenant.id, 'free')

  // 4. Create admin user
  await prisma.user.create({
    data: {
      email: data.adminEmail,
      role: 'ADMIN',
      tenantId: tenant.id
    }
  })

  // 5. Seed initial data
  await seedTenantData(tenant.id)

  return tenant
}
```

## Security Considerations

### Row-Level Security

```sql
-- PostgreSQL RLS policies
ALTER TABLE businesses ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON businesses
  USING (tenant_id = current_setting('app.current_tenant')::text);
```

### Cross-Tenant Protection

```typescript
// Middleware to prevent cross-tenant access
export async function validateTenantAccess(
  req: Request,
  resourceId: string,
  resourceType: 'business' | 'user'
) {
  const resource = await prisma[resourceType].findUnique({
    where: { id: resourceId },
    select: { tenantId: true }
  })

  if (resource?.tenantId !== req.tenant.id) {
    throw new Error('Unauthorized: Cross-tenant access denied')
  }
}
```

## Related Patterns

- [Unified Ecosystem](unified-ecosystem.md)
- [Authentication Systems](../patterns/authentication-systems.md)

---

**Production Scale**: Serves multiple tenant sites with isolated data, custom themes, and per-tenant feature flags.
