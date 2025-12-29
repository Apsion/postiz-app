# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Postiz is a social media scheduling platform with AI features. It's a monorepo managed by NX, supporting multiple social platforms (X, Instagram, LinkedIn, Facebook, YouTube, TikTok, Reddit, Pinterest, Threads, Bluesky, Mastodon, Discord, Slack, and more).

## Tech Stack

- **Monorepo**: NX workspace with pnpm
- **Frontend**: Next.js 14 (React 18, Tailwind CSS v4, Mantine UI v5)
- **Backend**: NestJS (REST API)
- **Database**: PostgreSQL with Prisma ORM (v6.5.0)
- **Queue/Cache**: Redis with BullMQ (microservices architecture)
- **Testing**: Jest with NX
- **Email**: Resend
- **Monitoring**: Sentry (with structured logging)
- **Linting**: ESLint + Prettier (singleQuote: true)
- **Node**: >=22.12.0 <23.0.0
- **Package Manager**: pnpm@10.6.1

## Architecture

### Monorepo Structure

```
apps/
  backend/       - NestJS REST API (port 3000)
  frontend/      - Next.js UI (port 4200)
  workers/       - BullMQ microservice workers
  cron/          - NestJS scheduled tasks (@nestjs/schedule)
  commands/      - CLI commands (nestjs-command)
  extension/     - Browser extension (Vite + React + CRXJS)
  sdk/           - NodeJS SDK (published as @postiz/node)

libraries/
  nestjs-libraries/     - Shared NestJS modules
    database/           - Prisma schema and client
    integrations/       - Social platform integrations
    agent/              - AI agent features
    chat/               - Chat functionality
    3rdparties/         - Third-party integrations
    videos/             - Video processing
    sentry/             - Sentry integration
    services/           - Shared services
    dtos/               - Data transfer objects
  react-shared-libraries/ - Shared React components
  helpers/              - Utility functions
```

### TypeScript Path Aliases

All apps use consistent path aliases (defined in tsconfig.base.json):

- `@gitroom/backend/*` → `apps/backend/src/*`
- `@gitroom/frontend/*` → `apps/frontend/src/*`
- `@gitroom/workers/*` → `apps/workers/src/*`
- `@gitroom/cron/*` → `apps/cron/src/*`
- `@gitroom/extension/*` → `apps/extension/src/*`
- `@gitroom/nestjs-libraries/*` → `libraries/nestjs-libraries/src/*`
- `@gitroom/react/*` → `libraries/react-shared-libraries/src/*`
- `@gitroom/helpers/*` → `libraries/helpers/src/*`

### Key Backend Modules (NestJS)

The backend uses NestJS with a modular architecture:

- **DatabaseModule**: Prisma ORM integration
- **BullMqModule**: Queue/job processing with custom microservice transport
- **ApiModule**: Main REST API routes (protected, authenticated)
- **PublicApiModule**: Public API endpoints (API key auth)
- **AgentModule**: AI agent functionality (Mastra, LangChain)
- **ThirdPartyModule**: External integrations (Make.com, N8N, etc.)
- **VideoModule**: Video processing
- **ChatModule**: Chat functionality (CopilotKit)

Guards are globally registered in app.module.ts:
- `ThrottlerBehindProxyGuard`: Rate limiting (default: 30 req/hour, configurable via API_LIMIT env var)
- `PoliciesGuard`: CASL-based permissions system

### Workers Architecture

Workers are NestJS microservices using a custom BullMQ transport strategy (`BullMqServer`). They:
- Run as separate processes from the main API
- Consume jobs from Redis queues
- Handle async tasks: post scheduling, media processing, integrations
- Initialize Sentry independently for error tracking

### Database Schema

Prisma schema location: `libraries/nestjs-libraries/src/database/prisma/schema.prisma`

Key models:
- **Organization**: Multi-tenant organizations
- **User**: Users with organization relationships
- **Post**: Social media posts
- **Integration**: Social platform connections
- **Media**: Uploaded media files
- **Subscription**: Payment/subscription management

## Quick Start

1. Install dependencies: `pnpm install` (auto-runs `prisma-generate` via postinstall)
2. Copy `.env.example` to `.env` and configure required variables
3. Start Docker services: `pnpm run dev:docker`
4. Push database schema: `pnpm run prisma-db-push`
5. Start all services: `pnpm run dev`

## Common Commands

### Development

```bash
# Install dependencies (automatically runs prisma-generate)
pnpm install

# Run all services in dev mode (backend, frontend, workers, cron, extension)
pnpm run dev

# Run individual services
pnpm run dev:backend    # NestJS API on port 3000
pnpm run dev:frontend   # Next.js on port 4200
pnpm run dev:workers    # BullMQ workers
pnpm run dev:cron       # Cron jobs

# Start local Docker services (Postgres + Redis)
pnpm run dev:docker
# or
docker compose -f docker-compose.dev.yaml up -d
```

### Linting & Formatting

```bash
# No root-level lint/format scripts defined
# Each app has its own linting configuration
# Prettier config: { "singleQuote": true }
```

### Database (Prisma)

**Note**: `pnpm install` automatically runs `prisma-generate` via postinstall hook.

```bash
# Generate Prisma client after schema changes
pnpm run prisma-generate

# Push schema to database (dev only - no migrations)
pnpm run prisma-db-push

# Pull schema from existing database
pnpm run prisma-db-pull

# Reset database (DESTRUCTIVE - drops all data)
pnpm run prisma-reset
```

Schema location: `libraries/nestjs-libraries/src/database/prisma/schema.prisma`

### Building

```bash
# Build all apps
pnpm run build

# Build individual apps
pnpm run build:backend
pnpm run build:frontend
pnpm run build:workers
pnpm run build:cron
pnpm run build:extension
```

### Testing

```bash
# Run all tests with coverage
pnpm test

# Test output goes to ./reports/junit.xml
```

### Production

```bash
# Start production services
pnpm run start:prod:backend
pnpm run start:prod:frontend
pnpm run start:prod:workers
pnpm run start:prod:cron

# Or use PM2 for all services
pnpm run pm2
```

## Environment Variables

Copy `.env.example` to `.env` and configure:

**Required:**
- `DATABASE_URL`: PostgreSQL connection string
- `REDIS_URL`: Redis connection string
- `JWT_SECRET`: Long random string for JWT signing
- `FRONTEND_URL`: Frontend URL (e.g., http://localhost:4200)
- `NEXT_PUBLIC_BACKEND_URL`: Backend URL for frontend (e.g., http://localhost:3000)
- `BACKEND_INTERNAL_URL`: Internal backend URL
- Cloudflare R2 credentials (for media storage)

**Important:** Always update `.env.example` when adding new environment variables.

## Code Patterns

### Backend (NestJS)

- Use decorators for routes, guards, and interceptors
- DTOs are in `libraries/nestjs-libraries/src/dtos/`
- Shared services in `libraries/nestjs-libraries/src/services/`
- Social integrations extend base provider pattern in `libraries/nestjs-libraries/src/integrations/`
- Queue jobs are defined in workers app and consumed by workers service

### Frontend (Next.js)

- **App Router** (not Pages Router) with route groups
  - Main app routes: `app/(app)/` - authenticated application
  - Extension routes: `app/(extension)/` - browser extension specific
- Server and client components following Next.js 14 patterns
- Shared React components in `libraries/react-shared-libraries/`
- Form handling with react-hook-form + @hookform/resolvers
- UI components from Mantine v5
- Styling: Tailwind CSS v4 + Sass for custom styles
- i18n with react-i18next

### Social Platform Integrations

Each integration is a provider in `libraries/nestjs-libraries/src/integrations/social/`:
- Implements standard methods: `post()`, `getProfile()`, etc.
- Uses platform-specific APIs (Twitter API v2, Instagram Graph API, etc.)
- Handles OAuth flows and token management

## Logging with Sentry

**Initialize first** (before other imports in main.ts):
```typescript
// Backend/Workers/Cron
import { initializeSentry } from '@gitroom/nestjs-libraries/sentry/initialize.sentry';
initializeSentry('backend'); // or 'workers', 'cron'
```

Use structured logging via Sentry:

```typescript
import * as Sentry from "@sentry/nestjs"; // or "@sentry/nextjs" for frontend

const { logger } = Sentry;

// Use logger.fmt template literal for variables
logger.trace("Starting database connection", { database: "users" });
logger.debug(logger.fmt`Cache miss for user: ${userId}`);
logger.info("Updated profile", { profileId: 345 });
logger.warn("Rate limit reached", { endpoint: "/api/results/", isEnterprise: false });
logger.error("Failed to process payment", { orderId: "123", amount: 99.99 });
logger.fatal("Database connection pool exhausted", { activeConnections: 100 });
```

Initialize Sentry (frontend):
```typescript
Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  enableLogs: true,
  integrations: [
    Sentry.consoleLoggingIntegration({ levels: ["log", "error", "warn"] }),
  ],
});
```

## Conventions

- **Commits**: Use conventional commits (`feat:`, `fix:`, `refactor:`, `test:`, `chore:`)
- **PRs**: Include clear description, related issue links, and UI screenshots/GIFs if UI changes
- **Comments**: Required for complex logic
- **TypeScript**: Strict mode enabled, but `strictNullChecks: false` and `noImplicitAny: true`

## Integration APIs

Postiz integrates with:
- Social platforms: X, Instagram, LinkedIn, Facebook, YouTube, TikTok, Reddit, Pinterest, Threads, Bluesky, Mastodon, Discord, Slack
- Automation: Make.com, N8N (via SDK)
- Email: Resend
- Payments: Stripe
- Analytics: Dub Analytics
- AI: OpenAI, Mastra agents

## Important Notes & Gotchas

- **Prisma client regenerates on install**: The postinstall hook runs `prisma-generate` automatically
- **Environment variables**: Both NEXT_PUBLIC_BACKEND_URL (frontend) and BACKEND_INTERNAL_URL (backend) must be set correctly
- **Path aliases**: Always use `@gitroom/*` imports, never relative paths across apps/libraries
- **Workers as microservices**: Workers are NestJS microservices, not standalone scripts - they use BullMQ transport
- **Sentry must initialize first**: In backend/workers/cron, call `initializeSentry()` before any other imports
- **Rate limiting**: Default is 30 requests/hour per IP, configurable via API_LIMIT env var
- **TypeScript strict mode**: Enabled, but `strictNullChecks: false` - be careful with null/undefined
- **Multi-tenant**: All data is scoped to Organization model - always include orgId in queries
- **Social OAuth**: Uses official OAuth flows, never stores API keys/tokens from users

## SDK Publishing

The SDK (`apps/sdk`) is published to npm as `@postiz/node`:
```bash
pnpm run publish-sdk  # Builds with tsup and publishes to npm
```

## Documentation

- Main docs: https://docs.postiz.com/
- Developer guide: https://docs.postiz.com/developer-guide
- Public API: https://docs.postiz.com/public-api
- Quick start: https://docs.postiz.com/quickstart
- Configuration reference: https://docs.postiz.com/configuration/reference
