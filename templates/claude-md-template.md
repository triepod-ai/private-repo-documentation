# CLAUDE.md Template

## Purpose

This template provides structured guidance for AI development assistants (like Claude Code) when working with your codebase. It serves as a comprehensive reference for architecture, patterns, commands, and best practices.

## Template Structure

```markdown
# CLAUDE.md

Brief description of the project and its purpose.

## Project Overview

- **Status**: 🎉 [Current status - e.g., Production Ready, In Development]
- **Architecture**: [Brief architecture description]
- **Tech Stack**: [Key technologies]

## Quick Reference Documentation

### Core System Documentation
- **[Link to main docs]** - Description
- **[Link to API docs]** - Description
- **[Link to guides]** - Description

## Quick Start Commands

### Development
\```bash
# Start development
npm run dev

# Run tests
npm run test

# Build for production
npm run build
\```

### Database Operations
\```bash
# Generate Prisma client
npx prisma generate

# Push schema changes
npx prisma db push

# Open Prisma Studio
npx prisma studio
\```

## Architecture Overview

### High-Level Structure
[Include ASCII diagram of system architecture]

### Key Components
- **Component 1**: Description and purpose
- **Component 2**: Description and purpose

### Import Patterns
\```typescript
// ✅ CORRECT: Use shared components
import { Button } from '@/components/ui/button'

// ❌ INCORRECT: Local duplication
import { Button } from './components/button'
\```

## Development Workflow

### Testing Strategy
\```bash
# Run all tests
npm run test:all

# Run specific test suite
npm run test:unit
\```

### Quality Gates
- **Build Validation**: `npm run build`
- **Type Safety**: `npm run type-check`
- **Linting**: `npm run lint`

## Authentication System

[Document authentication approach]

## Payment Systems

[Document payment integration if applicable]

## File Structure

\```
project/
├── app/                 # Main application
├── components/          # Shared components
├── lib/                 # Utilities
└── prisma/             # Database schema
\```

## Development Best Practices

### Component Usage
\```typescript
// Example of best practices
import { Button } from '@/components/ui'

export function MyComponent() {
  return <Button>Click me</Button>
}
\```

### Database Operations
- Always run schema changes from root
- Use Prisma Client for type-safe queries
- Test with isolated database

## Common Issues & Solutions

### Issue 1: [Description]
**Solution**: [Step-by-step resolution]

### Issue 2: [Description]
**Solution**: [Step-by-step resolution]

## Application-Specific Guidance

[Link to additional app-specific CLAUDE.md files if in monorepo]

## Current Development Focus

**Current Sprint**: [What's being worked on]

**Next Priorities**:
1. Priority 1 (estimated time)
2. Priority 2 (estimated time)
3. Priority 3 (estimated time)

---

**Repository**: [repo name]
**Primary Application**: [main app]
**Status**: [current status]
```

## Best Practices

### 1. Keep It Updated

Update CLAUDE.md whenever:
- Major architectural changes occur
- New patterns are established
- Common issues are discovered
- Dependencies are upgraded

### 2. Be Specific

```markdown
# ✅ GOOD: Specific and actionable
## Running Tests
\```bash
# Run all tests with coverage
npm run test:coverage

# Known issue: Test database must be running
docker run --name test-db -e POSTGRES_PASSWORD=password -p 5433:5432 -d postgres:15
\```

# ❌ BAD: Vague and unhelpful
## Running Tests
Run the tests using npm.
```

### 3. Include Context

Provide "why" along with "what":

```markdown
# ✅ GOOD: Explains reasoning
## Import Pattern
Always use absolute imports from the shared components directory:
\```typescript
import { Button } from '@/components/ui/button'
\```
This ensures components are shared across all applications in the monorepo
and prevents duplication.

# ❌ BAD: Just states the rule
## Import Pattern
Use `@/components/ui/button`
```

### 4. Use Visual Aids

Include ASCII diagrams for architecture:

```
┌─────────────────────────────────────┐
│         Client Application          │
└────────────────┬────────────────────┘
                 │
┌────────────────▼────────────────────┐
│           API Gateway               │
└────────────────┬────────────────────┘
                 │
┌────────────────▼────────────────────┐
│           Database                  │
└─────────────────────────────────────┘
```

### 5. Document Gotchas

```markdown
## Critical Lessons Learned

### Static File Placement
**Issue**: Files in root `/public/` are NOT served by Next.js

**Solution**: Always use `/app-directory/public/` for static files

**Documentation**: See `STATIC_FILE_DEPLOYMENT.md` for full details
```

## Real-World Example Sections

### Installation & Setup
```markdown
## Installation

\```bash
# Clone repository
git clone <repo-url>
cd project

# Install dependencies
npm install

# Set up environment
cp .env.example .env.local
# Edit .env.local with your credentials

# Initialize database
npx prisma generate
npx prisma db push
npm run db:seed

# Start development
npm run dev
\```
```

### Environment Variables
```markdown
## Environment Configuration

### Required Variables
\```env
DATABASE_URL="postgresql://user:pass@localhost:5432/db"
JWT_SECRET="your-secret-here"
NEXT_PUBLIC_API_URL="http://localhost:4000"
\```

### Optional Variables
\```env
STRIPE_SECRET_KEY="sk_test_..." # For payment processing
REDIS_URL="redis://localhost:6379" # For caching
\```

### Generating Secrets
\```bash
# Generate secure JWT secrets
npm run generate:secrets
\```
```

### Troubleshooting
```markdown
## Troubleshooting

### Build Failures

**Symptom**: TypeScript errors during build

**Causes**:
1. Stale Prisma client
2. Missing environment variables
3. Dependency version conflicts

**Solutions**:
\```bash
# 1. Regenerate Prisma client
npx prisma generate

# 2. Verify environment variables
npm run validate:env

# 3. Clean install dependencies
rm -rf node_modules package-lock.json
npm install
\```
```

## Integration with Development Tools

### VS Code Settings
```markdown
## VS Code Configuration

Recommended extensions:
- Prisma
- ESLint
- Tailwind CSS IntelliSense

\`.vscode/settings.json\`:
\```json
{
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true
  }
}
\```
```

## Maintenance Schedule

### Regular Updates

- **Weekly**: Update current development focus
- **Monthly**: Review and update dependencies
- **Quarterly**: Full documentation audit
- **Per Major Release**: Comprehensive update

## Related Templates

- [Implementation Guide Template](implementation-guide.md)
- [Testing Framework Template](testing-framework.md)

---

**Purpose**: Provide comprehensive, actionable guidance for AI assistants and developers working with the codebase.
