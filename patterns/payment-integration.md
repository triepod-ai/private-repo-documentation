# Payment Integration Pattern

## Overview

Multi-provider payment system supporting Stripe and PayPal with unified interface, webhook processing, subscription management, and comprehensive error handling.

## Architecture Design

### Payment System Overview

```
┌──────────────────────────────────────────────────────────────┐
│                 Client Application                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Checkout   │  │ Subscription │  │   Account    │      │
│  │    Page      │  │  Management  │  │   Billing    │      │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘      │
└─────────┼──────────────────┼──────────────────┼──────────────┘
          │                  │                  │
┌─────────▼──────────────────▼──────────────────▼──────────────┐
│           Payment Provider (React Context)                    │
│  • Provider selection                                        │
│  • Payment method management                                 │
│  • Transaction state                                         │
└─────────┬─────────────────────────────────┬──────────────────┘
          │                                 │
┌─────────▼─────────────────┐  ┌────────────▼─────────────────┐
│    Stripe Integration     │  │   PayPal Integration         │
│  • Elements SDK           │  │  • Smart Buttons             │
│  • Payment Intents        │  │  • Orders API                │
│  • Subscriptions          │  │  • Subscriptions             │
│  • Customer Portal        │  │  • Billing Agreements        │
└─────────┬─────────────────┘  └────────────┬─────────────────┘
          │                                 │
┌─────────▼─────────────────────────────────▼──────────────────┐
│                  Webhook Processing                           │
│  • Signature verification                                    │
│  • Event routing                                             │
│  • Idempotency handling                                      │
│  • Failure recovery                                          │
└─────────┬────────────────────────────────────────────────────┘
          │
┌─────────▼────────────────────────────────────────────────────┐
│                   Database Layer                              │
│  ┌──────────┐  ┌──────────────┐  ┌────────────────────────┐ │
│  │ Payments │  │ Subscriptions│  │  Webhook Events        │ │
│  └──────────┘  └──────────────┘  └────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. Payment Provider (Unified Interface)

```typescript
// providers/payment-provider.tsx

type PaymentProvider = 'stripe' | 'paypal'

interface PaymentContextType {
  provider: PaymentProvider
  setProvider: (provider: PaymentProvider) => void
  createPayment: (amount: number, currency: string) => Promise<Payment>
  createSubscription: (planId: string) => Promise<Subscription>
  cancelSubscription: (subscriptionId: string) => Promise<void>
  isProcessing: boolean
}

export function PaymentProvider({ children }: PropsWithChildren) {
  const [provider, setProvider] = useState<PaymentProvider>('stripe')
  const [isProcessing, setIsProcessing] = useState(false)

  const createPayment = async (amount: number, currency: string) => {
    setIsProcessing(true)
    try {
      if (provider === 'stripe') {
        return await createStripePayment(amount, currency)
      } else {
        return await createPayPalPayment(amount, currency)
      }
    } finally {
      setIsProcessing(false)
    }
  }

  return (
    <PaymentContext.Provider
      value={{ provider, setProvider, createPayment, isProcessing }}
    >
      {children}
    </PaymentContext.Provider>
  )
}
```

### 2. Stripe Integration

```typescript
// components/payments/stripe-payment-form.tsx

import { Elements, PaymentElement, useStripe, useElements } from '@stripe/react-stripe-js'
import { loadStripe } from '@stripe/stripe-js'

const stripePromise = loadStripe(process.env.NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY!)

export function StripePaymentForm({ amount, onSuccess }: Props) {
  const stripe = useStripe()
  const elements = useElements()
  const [error, setError] = useState<string>()

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()

    if (!stripe || !elements) return

    setError(undefined)

    // Create payment intent on server
    const { clientSecret } = await fetch('/api/payments/stripe/intent', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ amount })
    }).then(r => r.json())

    // Confirm payment
    const { error: paymentError, paymentIntent } = await stripe.confirmPayment({
      elements,
      clientSecret,
      confirmParams: {
        return_url: `${window.location.origin}/payment/success`
      }
    })

    if (paymentError) {
      setError(paymentError.message)
    } else if (paymentIntent.status === 'succeeded') {
      onSuccess(paymentIntent)
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <PaymentElement />
      <button type="submit" disabled={!stripe}>
        Pay ${(amount / 100).toFixed(2)}
      </button>
      {error && <div className="error">{error}</div>}
    </form>
  )
}

// Wrapper with Elements provider
export function StripeCheckout(props: Props) {
  return (
    <Elements stripe={stripePromise}>
      <StripePaymentForm {...props} />
    </Elements>
  )
}
```

### 3. PayPal Integration

```typescript
// components/payments/paypal-button.tsx

import { PayPalButtons, PayPalScriptProvider } from '@paypal/react-paypal-js'

export function PayPalCheckout({ amount, onSuccess }: Props) {
  return (
    <PayPalScriptProvider options={{
      clientId: process.env.NEXT_PUBLIC_PAYPAL_CLIENT_ID!,
      currency: 'USD'
    }}>
      <PayPalButtons
        createOrder={async () => {
          const response = await fetch('/api/payments/paypal/create-order', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ amount: amount / 100 })
          })
          const { orderId } = await response.json()
          return orderId
        }}
        onApprove={async (data) => {
          const response = await fetch('/api/payments/paypal/capture-order', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ orderId: data.orderID })
          })
          const order = await response.json()
          onSuccess(order)
        }}
        onError={(err) => {
          console.error('PayPal error:', err)
        }}
      />
    </PayPalScriptProvider>
  )
}
```

### 4. Server-Side Payment Processing

```typescript
// app/api/payments/stripe/intent/route.ts

import Stripe from 'stripe'

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2023-10-16'
})

export async function POST(request: Request) {
  const { amount, currency = 'usd', userId } = await request.json()

  try {
    // Create payment intent
    const paymentIntent = await stripe.paymentIntents.create({
      amount,
      currency,
      metadata: { userId },
      automatic_payment_methods: { enabled: true }
    })

    // Store in database
    await prisma.payment.create({
      data: {
        userId,
        provider: 'stripe',
        amount,
        currency,
        status: 'pending',
        stripePaymentIntentId: paymentIntent.id
      }
    })

    return Response.json({ clientSecret: paymentIntent.client_secret })
  } catch (error) {
    return Response.json({ error: error.message }, { status: 500 })
  }
}
```

## Webhook Processing

### Stripe Webhooks

```typescript
// app/api/webhooks/stripe/route.ts

import { headers } from 'next/headers'
import Stripe from 'stripe'

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!)

export async function POST(request: Request) {
  const body = await request.text()
  const signature = headers().get('stripe-signature')!

  let event: Stripe.Event

  try {
    // Verify webhook signature
    event = stripe.webhooks.constructEvent(
      body,
      signature,
      process.env.STRIPE_WEBHOOK_SECRET!
    )
  } catch (err) {
    return Response.json({ error: 'Invalid signature' }, { status: 400 })
  }

  // Handle event
  switch (event.type) {
    case 'payment_intent.succeeded':
      await handlePaymentSuccess(event.data.object as Stripe.PaymentIntent)
      break

    case 'payment_intent.payment_failed':
      await handlePaymentFailure(event.data.object as Stripe.PaymentIntent)
      break

    case 'customer.subscription.created':
      await handleSubscriptionCreated(event.data.object as Stripe.Subscription)
      break

    case 'customer.subscription.deleted':
      await handleSubscriptionCanceled(event.data.object as Stripe.Subscription)
      break

    case 'invoice.payment_succeeded':
      await handleInvoicePaymentSucceeded(event.data.object as Stripe.Invoice)
      break
  }

  return Response.json({ received: true })
}

async function handlePaymentSuccess(paymentIntent: Stripe.PaymentIntent) {
  // Update payment record
  await prisma.payment.update({
    where: { stripePaymentIntentId: paymentIntent.id },
    data: {
      status: 'completed',
      completedAt: new Date()
    }
  })

  // Send confirmation email
  await sendPaymentConfirmation(paymentIntent.metadata.userId)

  // Update user's subscription or credits
  // ... business logic
}
```

### PayPal Webhooks

```typescript
// app/api/webhooks/paypal/route.ts

export async function POST(request: Request) {
  const body = await request.json()
  const headers = Object.fromEntries(request.headers)

  // Verify webhook signature
  const isValid = await verifyPayPalWebhook(body, headers)
  if (!isValid) {
    return Response.json({ error: 'Invalid signature' }, { status: 400 })
  }

  // Handle event
  switch (body.event_type) {
    case 'PAYMENT.CAPTURE.COMPLETED':
      await handlePayPalPaymentSuccess(body.resource)
      break

    case 'PAYMENT.CAPTURE.DENIED':
      await handlePayPalPaymentFailure(body.resource)
      break

    case 'BILLING.SUBSCRIPTION.CREATED':
      await handlePayPalSubscriptionCreated(body.resource)
      break

    case 'BILLING.SUBSCRIPTION.CANCELLED':
      await handlePayPalSubscriptionCanceled(body.resource)
      break
  }

  return Response.json({ received: true })
}
```

## Subscription Management

### Creating Subscriptions

```typescript
// lib/subscriptions.ts

export async function createStripeSubscription(
  userId: string,
  priceId: string
): Promise<Subscription> {
  // Get or create Stripe customer
  let customer = await prisma.user.findUnique({
    where: { id: userId },
    select: { stripeCustomerId: true }
  })

  if (!customer?.stripeCustomerId) {
    const stripeCustomer = await stripe.customers.create({
      email: user.email,
      metadata: { userId }
    })

    await prisma.user.update({
      where: { id: userId },
      data: { stripeCustomerId: stripeCustomer.id }
    })

    customer = { stripeCustomerId: stripeCustomer.id }
  }

  // Create subscription
  const subscription = await stripe.subscriptions.create({
    customer: customer.stripeCustomerId,
    items: [{ price: priceId }],
    payment_behavior: 'default_incomplete',
    payment_settings: { save_default_payment_method: 'on_subscription' },
    expand: ['latest_invoice.payment_intent']
  })

  // Store in database
  await prisma.subscription.create({
    data: {
      userId,
      provider: 'stripe',
      stripeSubscriptionId: subscription.id,
      status: 'active',
      currentPeriodEnd: new Date(subscription.current_period_end * 1000)
    }
  })

  return subscription
}
```

### Subscription Status Management

```typescript
// lib/subscription-status.ts

type SubscriptionStatus =
  | 'active'
  | 'past_due'
  | 'canceled'
  | 'incomplete'
  | 'trialing'

export async function checkSubscriptionAccess(
  userId: string
): Promise<boolean> {
  const subscription = await prisma.subscription.findFirst({
    where: {
      userId,
      status: { in: ['active', 'trialing'] },
      currentPeriodEnd: { gt: new Date() }
    }
  })

  return !!subscription
}

export async function cancelSubscription(
  subscriptionId: string
): Promise<void> {
  const subscription = await prisma.subscription.findUnique({
    where: { id: subscriptionId }
  })

  if (subscription.provider === 'stripe') {
    await stripe.subscriptions.cancel(subscription.stripeSubscriptionId)
  } else if (subscription.provider === 'paypal') {
    await cancelPayPalSubscription(subscription.paypalSubscriptionId)
  }

  await prisma.subscription.update({
    where: { id: subscriptionId },
    data: { status: 'canceled' }
  })
}
```

## Database Schema

```prisma
// prisma/schema.prisma

model Payment {
  id        String   @id @default(cuid())
  userId    String
  user      User     @relation(fields: [userId], references: [id])

  provider  PaymentProvider
  amount    Int
  currency  String
  status    PaymentStatus

  // Provider-specific IDs
  stripePaymentIntentId String?
  paypalOrderId         String?

  createdAt   DateTime @default(now())
  completedAt DateTime?

  @@index([userId])
  @@index([status])
}

model Subscription {
  id        String   @id @default(cuid())
  userId    String
  user      User     @relation(fields: [userId], references: [id])

  provider  PaymentProvider
  status    SubscriptionStatus

  stripeSubscriptionId String?
  paypalSubscriptionId String?

  currentPeriodEnd DateTime
  cancelAt         DateTime?
  canceledAt       DateTime?

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([userId])
  @@index([status])
}

enum PaymentProvider {
  STRIPE
  PAYPAL
}

enum PaymentStatus {
  PENDING
  PROCESSING
  COMPLETED
  FAILED
  REFUNDED
}

enum SubscriptionStatus {
  ACTIVE
  PAST_DUE
  CANCELED
  INCOMPLETE
  TRIALING
}
```

## Error Handling

```typescript
// lib/payment-errors.ts

export class PaymentError extends Error {
  constructor(
    public code:
      | 'INSUFFICIENT_FUNDS'
      | 'CARD_DECLINED'
      | 'PROCESSING_ERROR'
      | 'INVALID_AMOUNT',
    message: string,
    public provider: 'stripe' | 'paypal'
  ) {
    super(message)
  }
}

export function handlePaymentError(error: unknown): PaymentError {
  if (error instanceof Stripe.errors.StripeCardError) {
    return new PaymentError(
      error.code === 'insufficient_funds'
        ? 'INSUFFICIENT_FUNDS'
        : 'CARD_DECLINED',
      error.message,
      'stripe'
    )
  }

  return new PaymentError('PROCESSING_ERROR', 'Payment failed', 'stripe')
}

// User-friendly error messages
export function getUserMessage(error: PaymentError): string {
  const messages = {
    INSUFFICIENT_FUNDS: 'Your card has insufficient funds.',
    CARD_DECLINED: 'Your card was declined. Please try another card.',
    PROCESSING_ERROR: 'An error occurred processing your payment.',
    INVALID_AMOUNT: 'The payment amount is invalid.'
  }

  return messages[error.code]
}
```

## Testing Strategy

```typescript
// __tests__/payments/stripe.test.ts

describe('Stripe Payment Flow', () => {
  test('successful payment creates database record', async () => {
    const { paymentIntent } = await createTestPaymentIntent(1000)

    const payment = await prisma.payment.findFirst({
      where: { stripePaymentIntentId: paymentIntent.id }
    })

    expect(payment).toBeDefined()
    expect(payment.amount).toBe(1000)
    expect(payment.status).toBe('pending')
  })

  test('webhook updates payment status', async () => {
    const { paymentIntent } = await createTestPaymentIntent(1000)

    // Simulate webhook
    await handlePaymentSuccess(paymentIntent)

    const payment = await prisma.payment.findFirst({
      where: { stripePaymentIntentId: paymentIntent.id }
    })

    expect(payment.status).toBe('completed')
    expect(payment.completedAt).toBeDefined()
  })
})
```

## Best Practices

1. **Idempotency**: Use idempotency keys for Stripe API calls
2. **Webhook Verification**: Always verify webhook signatures
3. **Error Recovery**: Implement retry logic for failed payments
4. **Security**: Never expose API keys or secrets to client
5. **Logging**: Comprehensive logging for debugging and auditing
6. **Testing**: Use provider test environments extensively

## Related Patterns

- [API Gateway Pattern](../architectures/api-gateway-pattern.md)
- [Case Study: Payment System Design](../case-studies/payment-system-design.md)

---

**Production Implementation**: Processes real transactions across multiple payment providers with comprehensive webhook handling and subscription management.
