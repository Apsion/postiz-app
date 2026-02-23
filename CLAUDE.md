# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Postiz is a social media scheduling platform with AI features. It's a monorepo using pnpm workspaces, supporting 14+ social platforms (X, Instagram, LinkedIn, Facebook, YouTube, TikTok, Reddit, Pinterest, Threads, Bluesky, Mastodon, Discord, Slack, Dribbble).

## Tech Stack

- **Frontend**: Next.js 14 (React 18, Tailwind CSS v4, Mantine UI v5)
- **Backend**: NestJS (REST API)
- **Orchestrator**: NestJS + Temporal.io (workflow orchestration, replaces old BullMQ workers)
- **Database**: PostgreSQL with Prisma ORM (v6.5.0)
- **Cache**: Redis
- **Email**: Resend
- **Monitoring**: Sentry (structured logging)
- **Linting**: ESLint + Prettier (`singleQuote: true`)
- **Node**: >=22.12.0 <23.0.0
- **Package Manager**: pnpm@10.6.1

## Architecture

### Monorepo Structure

```
apps/
  backend/       - NestJS REST API (port 3000)
  frontend/      - Next.js UI (port 4200)
  orchestrator/  - Temporal.io workflow orchestrator
  commands/      - CLI commands (nestjs-command)
  extension/     - Browser extension (Vite + React + CRXJS)
  cli/           - Standalone CLI (published as `postiz` on npm)
  sdk/           - NodeJS SDK (published as @postiz/node)

libraries/
  nestjs-libraries/     - Shared NestJS modules
    database/           - Prisma schema and client
    integrations/       - Social platform integrations
    temporal/           - Temporal workflow client
    agent/              - AI agent features (Mastra, LangChain)
    chat/               - Chat functionality (CopilotKit, MCP)
    3rdparties/         - Third-party integrations
    videos/             - Video processing
    sentry/             - Sentry integration
    services/           - Shared services
    dtos/               - Data transfer objects
  react-shared-libraries/ - Shared React components
  helpers/              - Utility functions
```

### TypeScript Path Aliases (tsconfig.base.json)

Always use `@gitroom/*` imports, never relative paths across apps/libraries:

- `@gitroom/backend/*` → `apps/backend/src/*`
- `@gitroom/frontend/*` → `apps/frontend/src/*`
- `@gitroom/orchestrator/*` → `apps/orchestrator/src/*`
- `@gitroom/extension/*` → `apps/extension/src/*`
- `@gitroom/nestjs-libraries/*` → `libraries/nestjs-libraries/src/*`
- `@gitroom/react/*` → `libraries/react-shared-libraries/src/*`
- `@gitroom/helpers/*` → `libraries/helpers/src/*`

### Backend Architecture (NestJS)

Key modules in `apps/backend/src/app.module.ts`:

- **ApiModule**: Main REST API routes (protected, authenticated)
- **PublicApiModule**: Public API endpoints (API key auth)
- **AgentModule**: AI agent functionality (Mastra, LangChain)
- **ThirdPartyModule**: External integrations (Make.com, N8N)
- **TemporalModule**: Temporal workflow client for orchestrator communication
- **VideoModule**: Video processing
- **ChatModule**: Chat + MCP (Model Context Protocol)

Global guards:
- `ThrottlerBehindProxyGuard`: Rate limiting (default: 30 req/hour for POST only, configurable via `API_LIMIT`)
- `PoliciesGuard`: CASL-based permissions

Backend layering: Controller → Service → Repository (or Controller → Manager → Service → Repository).

### Orchestrator Architecture (Temporal.io)

The orchestrator (`apps/orchestrator/`) replaces the old BullMQ workers with Temporal.io workflows:

- **Activities** (`src/activities/`): Reusable work units — autopost, email, integrations, post processing
- **Workflows** (`src/workflows/`): Orchestration logic — post scheduling, email sending, token refresh, streaks
- **Signals** (`src/signals/`): External inputs to running workflows

### Frontend Architecture (Next.js)

- **App Router** (not Pages Router) with route groups:
  - `src/app/(app)/` — authenticated application
  - `src/app/(extension)/` — browser extension specific
- Shared components in `libraries/react-shared-libraries/`
- Forms: react-hook-form + @hookform/resolvers
- UI: Mantine v5 components
- Styling: Tailwind CSS v4 + Sass (`global.scss`, `colors.scss`)
- i18n: react-i18next
- Data fetching: SWR with `useFetch` hook from `@gitroom/helpers/utils/custom.fetch`
- Each SWR call must be in a separate hook (comply with react-hooks/rules-of-hooks)

### Social Platform Integrations

Each integration is a provider in `libraries/nestjs-libraries/src/integrations/social/`:
- Implements standard methods: `post()`, `getProfile()`, etc.
- Uses platform-specific APIs (Twitter API v2, Instagram Graph API, etc.)
- Handles OAuth flows and token management

### Database

Prisma schema: `libraries/nestjs-libraries/src/database/prisma/schema.prisma`

Key models: Organization (multi-tenant root), User, Post, Integration, Media, Subscription.

All data is scoped to Organization — always include orgId in queries.

## Commands

### Development

```bash
pnpm install                    # Install deps (auto-runs prisma-generate)
pnpm run dev                    # All services in parallel
pnpm run dev:backend            # NestJS API on port 3000
pnpm run dev:frontend           # Next.js on port 4200
pnpm run dev:orchestrator       # Temporal orchestrator
pnpm run dev:docker             # Start Postgres + Redis containers
```

### Database (Prisma)

```bash
pnpm run prisma-generate        # Generate client after schema changes
pnpm run prisma-db-push         # Push schema to database (no migrations)
pnpm run prisma-db-pull         # Pull schema from existing database
pnpm run prisma-reset           # DESTRUCTIVE - drops all data
```

### Building

```bash
pnpm run build                  # All apps (workspace-concurrency=1)
pnpm run build:backend
pnpm run build:frontend
pnpm run build:orchestrator
pnpm run build:extension
pnpm run build:cli
```

### Testing

```bash
pnpm test                       # Jest with coverage, output to ./reports/junit.xml
```

### Production

```bash
pnpm run start:prod:backend
pnpm run start:prod:frontend
pnpm run start:prod:orchestrator
pnpm run pm2                    # All services via PM2
```

### Publishing

```bash
pnpm run publish-sdk            # @postiz/node to npm
pnpm run publish-cli            # postiz CLI to npm
```

## Environment Variables

Copy `.env.example` to `.env`. Required:

- `DATABASE_URL` — PostgreSQL connection string
- `REDIS_URL` — Redis connection string
- `JWT_SECRET` — JWT signing key
- `FRONTEND_URL` — e.g., `http://localhost:4200`
- `NEXT_PUBLIC_BACKEND_URL` — Backend URL for frontend, e.g., `http://localhost:3000`
- `BACKEND_INTERNAL_URL` — Internal backend URL
- Cloudflare R2 credentials for media storage (or `STORAGE_PROVIDER=local`)

Always update `.env.example` when adding new environment variables.

## Code Patterns

### Sentry Logging

Initialize first (before other imports in main.ts):
```typescript
import { initializeSentry } from '@gitroom/nestjs-libraries/sentry/initialize.sentry';
initializeSentry('backend'); // or 'orchestrator'
```

Structured logging:
```typescript
import * as Sentry from "@sentry/nestjs"; // or "@sentry/nextjs" for frontend
const { logger } = Sentry;

logger.debug(logger.fmt`Cache miss for user: ${userId}`);
logger.info("Updated profile", { profileId: 345 });
logger.error("Failed to process payment", { orderId: "123", amount: 99.99 });
```

### Frontend SWR Pattern

Each SWR hook must be standalone (no nesting multiple useSWR in one hook):
```typescript
// Correct
const useCommunity = () => useSWR(...)

// Wrong - violates react-hooks/rules-of-hooks
const useCommunity = () => ({
  communities: () => useSWR(...),
  providers: () => useSWR(...),
})
```

## Conventions

- **Commits**: Conventional commits (`feat:`, `fix:`, `refactor:`, `test:`, `chore:`)
- **PRs**: Clear description, issue links, UI screenshots/GIFs for visual changes
- **TypeScript**: Strict mode with `strictNullChecks: false` and `noImplicitAny: true`
- **Dependencies**: Use pnpm only. Write native frontend components instead of installing from npm.
- **Styling**: Check `colors.scss`, `global.scss`, and `tailwind.config.js` before writing new components. All `--color-custom*` CSS vars are deprecated.

## Important Gotchas

- **Prisma regenerates on install**: postinstall hook runs `prisma-generate`
- **Sentry must initialize first**: Call `initializeSentry()` before any other imports in main.ts
- **Rate limiting**: 30 requests/hour per IP for POST requests, configurable via `API_LIMIT`
- **Multi-tenant**: All data scoped to Organization — always include orgId
- **Path aliases**: Always `@gitroom/*`, never relative paths across apps/libraries
- **`strictNullChecks: false`**: Be careful with null/undefined — TypeScript won't catch these

## Documentation

- Main docs: https://docs.postiz.com/
- Developer guide: https://docs.postiz.com/developer-guide
- Public API: https://docs.postiz.com/public-api
