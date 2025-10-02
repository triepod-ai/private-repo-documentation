# Shared Component Library Pattern

## Overview

Centralized component library shared across multiple applications within a monorepo, eliminating duplication and ensuring consistency. Based on shadcn/ui with 70+ production components.

## Architecture Design

### Component Organization

```
components/
├── ui/                         # Core UI components (70+)
│   ├── button.tsx
│   ├── card.tsx
│   ├── input.tsx
│   ├── dialog.tsx
│   └── ... (66 more)
├── payments/                   # Payment integration
│   ├── payment-form.tsx
│   ├── stripe-checkout.tsx
│   └── paypal-button.tsx
├── advertising/                # Ad integration
│   ├── adsense-slot.tsx
│   └── ad-container.tsx
├── forms/                      # Form components
│   ├── contact-form.tsx
│   └── subscription-form.tsx
└── mode-provider.tsx           # Cross-cutting concerns
```

### Import Pattern

```
┌────────────────────────────────────────────────────┐
│           Application 1 (Next.js)                  │
│                                                    │
│  import { Button } from '../../../components/ui'  │
└─────────────────────┬──────────────────────────────┘
                      │
┌─────────────────────▼──────────────────────────────┐
│         Shared Components (/components)            │
│                                                    │
│  export { Button } from './ui/button'             │
│  export { Card } from './ui/card'                 │
└─────────────────────┬──────────────────────────────┘
                      │
┌─────────────────────▼──────────────────────────────┐
│           Application 2 (Express API)              │
│                                                    │
│  import { Button } from '../../../components/ui'  │
└────────────────────────────────────────────────────┘
```

## Core Components

### 1. Base UI Components (shadcn/ui)

```typescript
// components/ui/button.tsx

import * as React from 'react'
import { Slot } from '@radix-ui/react-slot'
import { cva, type VariantProps } from 'class-variance-authority'
import { cn } from '@/lib/utils'

const buttonVariants = cva(
  'inline-flex items-center justify-center rounded-md text-sm font-medium transition-colors focus-visible:outline-none disabled:pointer-events-none disabled:opacity-50',
  {
    variants: {
      variant: {
        default: 'bg-primary text-primary-foreground hover:bg-primary/90',
        destructive: 'bg-destructive text-destructive-foreground hover:bg-destructive/90',
        outline: 'border border-input bg-background hover:bg-accent',
        ghost: 'hover:bg-accent hover:text-accent-foreground',
        link: 'text-primary underline-offset-4 hover:underline'
      },
      size: {
        default: 'h-10 px-4 py-2',
        sm: 'h-9 rounded-md px-3',
        lg: 'h-11 rounded-md px-8',
        icon: 'h-10 w-10'
      }
    },
    defaultVariants: {
      variant: 'default',
      size: 'default'
    }
  }
)

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  asChild?: boolean
}

const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, asChild = false, ...props }, ref) => {
    const Comp = asChild ? Slot : 'button'
    return (
      <Comp
        className={cn(buttonVariants({ variant, size, className }))}
        ref={ref}
        {...props}
      />
    )
  }
)
Button.displayName = 'Button'

export { Button, buttonVariants }
```

### 2. Domain-Specific Components

```typescript
// components/payments/payment-form.tsx

import { Button } from '../ui/button'
import { Card } from '../ui/card'
import { Input } from '../ui/input'

interface PaymentFormProps {
  amount: number
  onSubmit: (data: PaymentData) => Promise<void>
  provider?: 'stripe' | 'paypal'
}

export function PaymentForm({ amount, onSubmit, provider = 'stripe' }: PaymentFormProps) {
  const [isProcessing, setIsProcessing] = useState(false)

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    setIsProcessing(true)

    try {
      await onSubmit(formData)
    } finally {
      setIsProcessing(false)
    }
  }

  return (
    <Card>
      <form onSubmit={handleSubmit}>
        <Input
          name="cardNumber"
          placeholder="Card Number"
          required
        />
        <Input
          name="expiry"
          placeholder="MM/YY"
          required
        />
        <Input
          name="cvc"
          placeholder="CVC"
          required
        />
        <Button type="submit" disabled={isProcessing}>
          {isProcessing ? 'Processing...' : `Pay $${amount}`}
        </Button>
      </form>
    </Card>
  )
}
```

### 3. Compound Components

```typescript
// components/ui/dialog.tsx

export function Dialog({ children, ...props }: DialogProps) {
  return (
    <DialogPrimitive.Root {...props}>
      {children}
    </DialogPrimitive.Root>
  )
}

export function DialogTrigger({ children, ...props }: DialogTriggerProps) {
  return (
    <DialogPrimitive.Trigger asChild {...props}>
      {children}
    </DialogPrimitive.Trigger>
  )
}

export function DialogContent({ children, ...props }: DialogContentProps) {
  return (
    <DialogPrimitive.Portal>
      <DialogPrimitive.Overlay className="dialog-overlay" />
      <DialogPrimitive.Content className="dialog-content" {...props}>
        {children}
      </DialogPrimitive.Content>
    </DialogPrimitive.Portal>
  )
}

// Usage across applications
import { Dialog, DialogTrigger, DialogContent } from '@/components/ui/dialog'

function MyComponent() {
  return (
    <Dialog>
      <DialogTrigger>
        <Button>Open Dialog</Button>
      </DialogTrigger>
      <DialogContent>
        <h2>Dialog Content</h2>
      </DialogContent>
    </Dialog>
  )
}
```

## Central Export Pattern

```typescript
// components/ui/index.ts

export { Button, buttonVariants } from './button'
export { Card, CardHeader, CardContent, CardFooter } from './card'
export { Input } from './input'
export { Label } from './label'
export { Dialog, DialogTrigger, DialogContent } from './dialog'
export { Tabs, TabsList, TabsTrigger, TabsContent } from './tabs'
// ... 64 more exports

// Application usage
import { Button, Card, Input } from '../../../components/ui'
// Single import line for multiple components
```

## Styling Strategy

### Tailwind Configuration

```typescript
// tailwind.config.ts (shared config)

export default {
  content: [
    './app/**/*.{ts,tsx}',
    './components/**/*.{ts,tsx}',
    './website/**/*.{ts,tsx}',
    './content-manager/**/*.{ts,tsx}'
  ],
  theme: {
    extend: {
      colors: {
        border: 'hsl(var(--border))',
        input: 'hsl(var(--input))',
        ring: 'hsl(var(--ring))',
        background: 'hsl(var(--background))',
        foreground: 'hsl(var(--foreground))',
        primary: {
          DEFAULT: 'hsl(var(--primary))',
          foreground: 'hsl(var(--primary-foreground))'
        }
        // ... more colors
      }
    }
  },
  plugins: [require('tailwindcss-animate')]
}
```

### CSS Variables Approach

```css
/* app/globals.css */

@layer base {
  :root {
    --background: 0 0% 100%;
    --foreground: 222.2 84% 4.9%;
    --primary: 222.2 47.4% 11.2%;
    --primary-foreground: 210 40% 98%;
    /* ... more variables */
  }

  .dark {
    --background: 222.2 84% 4.9%;
    --foreground: 210 40% 98%;
    --primary: 210 40% 98%;
    --primary-foreground: 222.2 47.4% 11.2%;
    /* ... more variables */
  }
}
```

## Component Composition

### Composable Form Example

```typescript
// components/forms/contact-form.tsx

import { Button } from '../ui/button'
import { Input } from '../ui/input'
import { Label } from '../ui/label'
import { Textarea } from '../ui/textarea'
import { Card, CardHeader, CardContent, CardFooter } from '../ui/card'

export function ContactForm({ onSubmit }: ContactFormProps) {
  return (
    <Card className="max-w-md">
      <CardHeader>
        <h2>Get in Touch</h2>
      </CardHeader>
      <CardContent>
        <form onSubmit={onSubmit}>
          <div className="space-y-4">
            <div>
              <Label htmlFor="name">Name</Label>
              <Input id="name" name="name" required />
            </div>
            <div>
              <Label htmlFor="email">Email</Label>
              <Input id="email" name="email" type="email" required />
            </div>
            <div>
              <Label htmlFor="message">Message</Label>
              <Textarea id="message" name="message" required />
            </div>
          </div>
        </form>
      </CardContent>
      <CardFooter>
        <Button type="submit">Send Message</Button>
      </CardFooter>
    </Card>
  )
}
```

## Type Safety

### Shared Type Definitions

```typescript
// types/components.ts

export interface BaseComponentProps {
  className?: string
  children?: React.ReactNode
}

export interface FormFieldProps extends BaseComponentProps {
  name: string
  label?: string
  error?: string
  required?: boolean
}

export interface ButtonProps extends BaseComponentProps {
  variant?: 'default' | 'destructive' | 'outline' | 'ghost' | 'link'
  size?: 'default' | 'sm' | 'lg' | 'icon'
  disabled?: boolean
  onClick?: () => void
}

// Usage in components
import type { ButtonProps } from '@/types/components'

export function MyButton({ variant = 'default', ...props }: ButtonProps) {
  return <Button variant={variant} {...props} />
}
```

## Testing Shared Components

### Component Tests

```typescript
// components/ui/__tests__/button.test.tsx

import { render, screen, fireEvent } from '@testing-library/react'
import { Button } from '../button'

describe('Button', () => {
  test('renders with default variant', () => {
    render(<Button>Click me</Button>)
    const button = screen.getByRole('button')
    expect(button).toHaveClass('bg-primary')
  })

  test('renders with destructive variant', () => {
    render(<Button variant="destructive">Delete</Button>)
    const button = screen.getByRole('button')
    expect(button).toHaveClass('bg-destructive')
  })

  test('handles click events', () => {
    const handleClick = jest.fn()
    render(<Button onClick={handleClick}>Click</Button>)

    fireEvent.click(screen.getByRole('button'))
    expect(handleClick).toHaveBeenCalledTimes(1)
  })

  test('disables when disabled prop is true', () => {
    render(<Button disabled>Disabled</Button>)
    const button = screen.getByRole('button')
    expect(button).toBeDisabled()
  })
})
```

### Integration Tests

```typescript
// __tests__/forms/contact-form.test.tsx

describe('ContactForm', () => {
  test('submits form with correct data', async () => {
    const handleSubmit = jest.fn()
    render(<ContactForm onSubmit={handleSubmit} />)

    await userEvent.type(screen.getByLabelText('Name'), 'John Doe')
    await userEvent.type(screen.getByLabelText('Email'), 'john@example.com')
    await userEvent.type(screen.getByLabelText('Message'), 'Hello!')

    fireEvent.click(screen.getByText('Send Message'))

    expect(handleSubmit).toHaveBeenCalledWith({
      name: 'John Doe',
      email: 'john@example.com',
      message: 'Hello!'
    })
  })
})
```

## Documentation Strategy

### Component Documentation

```typescript
// components/ui/button.tsx

/**
 * Button component with multiple variants and sizes
 *
 * @example
 * ```tsx
 * <Button variant="default">Click me</Button>
 * <Button variant="destructive" size="lg">Delete</Button>
 * <Button variant="outline" disabled>Disabled</Button>
 * ```
 *
 * @param variant - Button style variant (default | destructive | outline | ghost | link)
 * @param size - Button size (default | sm | lg | icon)
 * @param disabled - Whether button is disabled
 * @param asChild - Render as child element (using Radix Slot)
 */
export function Button({ ... }) {
  // Implementation
}
```

### Storybook Integration

```typescript
// components/ui/button.stories.tsx

import type { Meta, StoryObj } from '@storybook/react'
import { Button } from './button'

const meta: Meta<typeof Button> = {
  title: 'UI/Button',
  component: Button,
  tags: ['autodocs']
}

export default meta
type Story = StoryObj<typeof Button>

export const Default: Story = {
  args: {
    children: 'Button',
    variant: 'default'
  }
}

export const Destructive: Story = {
  args: {
    children: 'Delete',
    variant: 'destructive'
  }
}

export const AllSizes: Story = {
  render: () => (
    <div className="flex gap-4">
      <Button size="sm">Small</Button>
      <Button size="default">Default</Button>
      <Button size="lg">Large</Button>
    </div>
  )
}
```

## Versioning & Updates

### Component Changelog

```markdown
# Component Library Changelog

## v2.1.0 (2025-01-15)

### Added
- `Combobox` component for searchable select inputs
- `Calendar` component with date range selection
- Dark mode support for all components

### Changed
- `Button` now supports `loading` prop
- `Input` improved accessibility labels

### Fixed
- `Dialog` close animation glitch
- `Tabs` keyboard navigation
```

### Breaking Changes

```typescript
// v1.x
<Button color="primary">Click</Button>

// v2.x (breaking change)
<Button variant="default">Click</Button>

// Migration guide provided
```

## Performance Optimization

### Tree Shaking

```typescript
// ✅ GOOD: Individual imports allow tree shaking
import { Button } from '@/components/ui/button'
import { Card } from '@/components/ui/card'

// ❌ AVOID: Barrel imports may include unused code
import * as UI from '@/components/ui'
```

### Code Splitting

```typescript
// Dynamic imports for large components
const RichTextEditor = dynamic(
  () => import('@/components/forms/rich-text-editor'),
  { loading: () => <Skeleton className="h-64" /> }
)
```

## Best Practices

1. **Single Source of Truth**: All UI components in `/components`
2. **Consistent Naming**: Use PascalCase for components
3. **Prop Interface**: Export prop types for TypeScript
4. **Accessibility**: ARIA labels, keyboard navigation
5. **Documentation**: JSDoc comments + Storybook stories
6. **Testing**: Unit tests for all shared components
7. **Versioning**: Semantic versioning for breaking changes

## Metrics

**Real-World Scale**:
- 70+ shared components
- Used across 3+ applications
- Zero component duplication
- 185+ component tests
- 100% TypeScript coverage

## Related Patterns

- [Monorepo Architecture](monorepo-architecture.md)
- [Testing Strategies](testing-strategies.md)

---

**Production Impact**: Eliminates code duplication across applications, ensures UI consistency, and accelerates development velocity.
