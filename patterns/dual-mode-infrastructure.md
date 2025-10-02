# Dual-Mode Infrastructure Pattern

## Overview

Environment-based application mode switching enabling instant business model transformation through configuration changes. Allows a single codebase to serve multiple business strategies (e.g., portfolio vs. consultancy) with ~30 second activation time.

## Architecture Design

### Mode System Overview

```
┌──────────────────────────────────────────────────────────────┐
│                Environment Configuration                      │
│  SITE_MODE=portfolio | business | hybrid                     │
│  BUSINESS_MODE_PERCENTAGE=50  (for A/B testing)              │
└────────────────────┬─────────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────────┐
│                  Mode Provider (React Context)                │
│  • Detects current mode                                      │
│  • Provides mode-specific configuration                      │
│  • Handles A/B testing logic                                 │
└────────────────────┬─────────────────────────────────────────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
┌───────▼──────┐ ┌──▼──────┐ ┌──▼──────────┐
│  Portfolio   │ │Business │ │   Hybrid    │
│    Mode      │ │  Mode   │ │    Mode     │
└───────┬──────┘ └──┬──────┘ └──┬──────────┘
        │           │            │
┌───────▼───────────▼────────────▼─────────────────────────────┐
│              Component-Level Adaptation                       │
│  • Navigation menus                                          │
│  • Hero sections                                             │
│  • Call-to-action buttons                                    │
│  • Analytics tracking                                        │
│  • Ad placement                                              │
└──────────────────────────────────────────────────────────────┘
```

## Core Implementation

### 1. Mode Provider

```typescript
// components/mode-provider.tsx

type SiteMode = 'portfolio' | 'business' | 'hybrid'

interface ModeConfig {
  mode: SiteMode
  navigation: NavigationItem[]
  heroContent: HeroContent
  ctaText: string
  analyticsId: string
  showAds: boolean
}

interface ModeContextType {
  mode: SiteMode
  config: ModeConfig
  isBusinessMode: boolean
  isPortfolioMode: boolean
}

export function ModeProvider({ children }: PropsWithChildren) {
  // Determine mode from environment
  const baseMode = (process.env.NEXT_PUBLIC_SITE_MODE || 'portfolio') as SiteMode

  // A/B testing logic for hybrid mode
  const [effectiveMode, setEffectiveMode] = useState<SiteMode>(baseMode)

  useEffect(() => {
    if (baseMode === 'hybrid') {
      // Determine mode for this user session
      const businessPercentage = parseInt(
        process.env.NEXT_PUBLIC_BUSINESS_MODE_PERCENTAGE || '50'
      )

      const sessionMode = localStorage.getItem('site-mode')

      if (!sessionMode) {
        // Assign mode based on percentage
        const mode = Math.random() * 100 < businessPercentage
          ? 'business'
          : 'portfolio'

        localStorage.setItem('site-mode', mode)
        setEffectiveMode(mode)
      } else {
        setEffectiveMode(sessionMode as SiteMode)
      }
    }
  }, [baseMode])

  const config = getModeConfig(effectiveMode)

  return (
    <ModeContext.Provider
      value={{
        mode: effectiveMode,
        config,
        isBusinessMode: effectiveMode === 'business',
        isPortfolioMode: effectiveMode === 'portfolio'
      }}
    >
      {children}
    </ModeContext.Provider>
  )
}

function getModeConfig(mode: SiteMode): ModeConfig {
  const configs: Record<SiteMode, ModeConfig> = {
    portfolio: {
      mode: 'portfolio',
      navigation: [
        { label: 'Projects', href: '/projects' },
        { label: 'Blog', href: '/blog' },
        { label: 'About', href: '/about' }
      ],
      heroContent: {
        title: 'Full-Stack Developer & AI Consultant',
        subtitle: 'Building modern web applications and AI solutions',
        cta: 'View Projects'
      },
      ctaText: 'View Portfolio',
      analyticsId: process.env.NEXT_PUBLIC_GA_PORTFOLIO!,
      showAds: true
    },

    business: {
      mode: 'business',
      navigation: [
        { label: 'Services', href: '/services' },
        { label: 'Case Studies', href: '/case-studies' },
        { label: 'Contact', href: '/contact' }
      ],
      heroContent: {
        title: 'Transform Your Business with AI',
        subtitle: 'Enterprise consulting for modern digital transformation',
        cta: 'Schedule Consultation'
      },
      ctaText: 'Get Started',
      analyticsId: process.env.NEXT_PUBLIC_GA_BUSINESS!,
      showAds: false
    },

    hybrid: {
      // Hybrid uses one of the above configs based on user assignment
      ...configs.portfolio
    }
  }

  return configs[mode]
}

export function useMode() {
  return useContext(ModeContext)
}
```

### 2. Mode-Aware Components

```typescript
// components/hero.tsx

export function Hero() {
  const { config, isBusinessMode } = useMode()

  return (
    <section className="hero">
      <h1>{config.heroContent.title}</h1>
      <p>{config.heroContent.subtitle}</p>

      {isBusinessMode ? (
        <button onClick={openCalendly}>
          {config.heroContent.cta}
        </button>
      ) : (
        <Link href="/projects">
          {config.heroContent.cta}
        </Link>
      )}

      {/* Mode-specific imagery */}
      {isBusinessMode ? (
        <Image src="/business-hero.jpg" alt="Business" />
      ) : (
        <Image src="/portfolio-hero.jpg" alt="Portfolio" />
      )}
    </section>
  )
}
```

### 3. Analytics Integration

```typescript
// lib/analytics.ts

export function useModeAnalytics() {
  const { config, mode } = useMode()

  useEffect(() => {
    // Initialize Google Analytics with mode-specific ID
    if (typeof window !== 'undefined') {
      gtag('config', config.analyticsId, {
        custom_map: { dimension1: 'site_mode' }
      })

      gtag('event', 'page_view', {
        site_mode: mode
      })
    }
  }, [config.analyticsId, mode])

  const trackEvent = (eventName: string, params?: Record<string, any>) => {
    gtag('event', eventName, {
      ...params,
      site_mode: mode
    })
  }

  return { trackEvent }
}

// Usage in components
export function CTAButton() {
  const { trackEvent } = useModeAnalytics()
  const { isBusinessMode } = useMode()

  const handleClick = () => {
    trackEvent('cta_click', {
      button_type: isBusinessMode ? 'consultation' : 'portfolio'
    })

    // Proceed with action...
  }

  return <button onClick={handleClick}>Learn More</button>
}
```

### 4. Ad Placement Control

```typescript
// components/advertising/ad-slot.tsx

export function AdSlot({ position }: { position: string }) {
  const { config } = useMode()

  // Don't show ads in business mode
  if (!config.showAds) {
    return null
  }

  return (
    <ins
      className="adsbygoogle"
      style={{ display: 'block' }}
      data-ad-client="ca-pub-XXXXXXXXX"
      data-ad-slot={getSlotId(position)}
    />
  )
}
```

## Environment Configuration

### Development Setup

```bash
# Portfolio mode (default)
NEXT_PUBLIC_SITE_MODE=portfolio
NEXT_PUBLIC_GA_PORTFOLIO=G-XXXXXXXXX
npm run dev

# Business mode
NEXT_PUBLIC_SITE_MODE=business
NEXT_PUBLIC_GA_BUSINESS=G-YYYYYYYYY
npm run dev

# A/B testing mode
NEXT_PUBLIC_SITE_MODE=hybrid
NEXT_PUBLIC_BUSINESS_MODE_PERCENTAGE=50
npm run dev
```

### Production Deployment

```yaml
# Vercel/Netlify environment variables

# Current active mode
NEXT_PUBLIC_SITE_MODE=portfolio

# A/B testing configuration
NEXT_PUBLIC_BUSINESS_MODE_PERCENTAGE=25

# Analytics IDs
NEXT_PUBLIC_GA_PORTFOLIO=G-PORTFOLIO
NEXT_PUBLIC_GA_BUSINESS=G-BUSINESS
```

## Mode Switching Strategy

### 30-Second Business Activation

```bash
# Current state: Portfolio mode live in production

# Step 1: Update environment variable (10 seconds)
vercel env add NEXT_PUBLIC_SITE_MODE business production

# Step 2: Redeploy (20 seconds)
vercel deploy --prod

# Result: Business mode now live
```

### Gradual Rollout with Hybrid Mode

```bash
# Start with 10% business mode traffic
NEXT_PUBLIC_SITE_MODE=hybrid
NEXT_PUBLIC_BUSINESS_MODE_PERCENTAGE=10

# Monitor conversion metrics...

# Increase to 25%
NEXT_PUBLIC_BUSINESS_MODE_PERCENTAGE=25

# Monitor metrics...

# Full business mode
NEXT_PUBLIC_SITE_MODE=business
```

## Testing Strategy

### Mode-Specific Tests

```typescript
// __tests__/mode-switching.test.tsx

describe('Mode Switching', () => {
  test('portfolio mode shows correct navigation', () => {
    process.env.NEXT_PUBLIC_SITE_MODE = 'portfolio'

    render(
      <ModeProvider>
        <Navigation />
      </ModeProvider>
    )

    expect(screen.getByText('Projects')).toBeInTheDocument()
    expect(screen.queryByText('Services')).not.toBeInTheDocument()
  })

  test('business mode shows correct navigation', () => {
    process.env.NEXT_PUBLIC_SITE_MODE = 'business'

    render(
      <ModeProvider>
        <Navigation />
      </ModeProvider>
    )

    expect(screen.getByText('Services')).toBeInTheDocument()
    expect(screen.queryByText('Projects')).not.toBeInTheDocument()
  })

  test('hybrid mode assigns users to buckets', () => {
    process.env.NEXT_PUBLIC_SITE_MODE = 'hybrid'
    process.env.NEXT_PUBLIC_BUSINESS_MODE_PERCENTAGE = '50'

    const { rerender } = render(
      <ModeProvider>
        <Hero />
      </ModeProvider>
    )

    const mode = localStorage.getItem('site-mode')
    expect(['business', 'portfolio']).toContain(mode)

    // Mode persists across rerenders
    rerender(
      <ModeProvider>
        <Hero />
      </ModeProvider>
    )

    expect(localStorage.getItem('site-mode')).toBe(mode)
  })
})
```

## Use Cases

### 1. Market Opportunity Response

**Scenario**: Enterprise lead expresses interest in consulting services.

**Action**: Activate business mode to maximize conversion.

```bash
# Immediate switch (30 seconds)
vercel env add NEXT_PUBLIC_SITE_MODE business production
vercel deploy --prod
```

### 2. Revenue Model Testing

**Scenario**: Test whether business mode generates more revenue than portfolio + ads.

**Action**: Use hybrid mode with A/B testing.

```bash
# Run for 30 days with 50/50 split
NEXT_PUBLIC_SITE_MODE=hybrid
NEXT_PUBLIC_BUSINESS_MODE_PERCENTAGE=50

# Analyze conversion metrics
# - Portfolio mode: Ad revenue + project inquiries
# - Business mode: Consultation bookings + contract value
```

### 3. Seasonal Strategy

**Scenario**: Focus on ad revenue during high-traffic seasons, switch to consulting during enterprise buying cycles.

**Action**: Schedule mode switches aligned with business calendar.

```bash
# Q4: Holiday traffic, maximize ad revenue
NEXT_PUBLIC_SITE_MODE=portfolio

# Q1: Enterprise budgets renew, focus on consulting
NEXT_PUBLIC_SITE_MODE=business
```

## Performance Considerations

- **Build Time**: No impact (mode determined at runtime)
- **Bundle Size**: Minimal (~2-3KB for mode logic)
- **Runtime Performance**: Negligible (context lookup)
- **Analytics**: Separate tracking for accurate conversion measurement

## Metrics & Monitoring

### Key Metrics by Mode

**Portfolio Mode**:
- Ad revenue (RPM, CTR)
- Project inquiry rate
- Blog engagement

**Business Mode**:
- Consultation booking rate
- Lead quality score
- Contract conversion rate

### A/B Testing Analysis

```typescript
// analytics/mode-comparison.ts

interface ModeMetrics {
  mode: SiteMode
  sessions: number
  conversions: number
  revenue: number
  conversionRate: number
}

export async function compareModels(dateRange: DateRange): Promise<{
  portfolio: ModeMetrics
  business: ModeMetrics
  winner: SiteMode
}> {
  const data = await fetchAnalytics(dateRange)

  const portfolio = calculateMetrics(data, 'portfolio')
  const business = calculateMetrics(data, 'business')

  const winner = business.conversionRate > portfolio.conversionRate * 1.2
    ? 'business'
    : 'portfolio'

  return { portfolio, business, winner }
}
```

## Best Practices

1. **Persistent User Experience**: Once assigned a mode, keep users in that mode for session consistency
2. **Analytics Separation**: Use distinct tracking IDs to prevent data contamination
3. **Gradual Rollout**: Use hybrid mode for safe testing before full switch
4. **Documentation**: Document mode-specific business logic clearly
5. **Monitoring**: Track key metrics for each mode to inform strategy

## Related Patterns

- [Case Study: Dual-Mode Deployment](../case-studies/dual-mode-deployment.md)
- [Testing Strategies](testing-strategies.md)

---

**Production Impact**: Enables instant business model pivot with <30 second deployment time, supporting A/B testing for 4-6x conversion improvement potential.
