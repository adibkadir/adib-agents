---
name: docker-nextjs
description: Multi-stage Dockerfiles for Next.js standalone apps — Alpine and Debian Slim variants, pnpm workspace support, outputFileTracingRoot for monorepos, non-root users, layer caching, and .dockerignore. Use when containerizing a Next.js application for production deployment.
---

# Docker Multi-Stage Builds for Next.js

## Standard Dockerfile (Alpine, Single-App)

```dockerfile
# Stage 1: Install dependencies
FROM node:22-alpine AS deps
RUN corepack enable && corepack prepare pnpm@latest --activate
WORKDIR /app

COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile

# Stage 2: Build the application
FROM node:22-alpine AS builder
RUN corepack enable && corepack prepare pnpm@latest --activate
WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Build-time env vars (baked into JS bundle)
ARG NEXT_PUBLIC_APP_URL
ARG NEXT_PUBLIC_R2_PUBLIC_URL
ENV NEXT_PUBLIC_APP_URL=$NEXT_PUBLIC_APP_URL
ENV NEXT_PUBLIC_R2_PUBLIC_URL=$NEXT_PUBLIC_R2_PUBLIC_URL

# Dummy DATABASE_URL for build (no actual connection needed)
ENV DATABASE_URL="postgresql://dummy:dummy@localhost:5432/dummy"

RUN pnpm build

# Stage 3: Production runner
FROM node:22-alpine AS runner
WORKDIR /app

ENV NODE_ENV=production

# Non-root user
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nextjs

# Copy standalone output
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

# Copy worker files (if applicable)
COPY --from=builder --chown=nextjs:nodejs /app/worker ./worker
COPY --from=builder --chown=nextjs:nodejs /app/db ./db

# Copy entrypoint
COPY --chown=nextjs:nodejs docker-entrypoint.sh ./
RUN chmod +x docker-entrypoint.sh

USER nextjs
EXPOSE 3000
ENV PORT=3000 HOSTNAME="0.0.0.0"

ENTRYPOINT ["./docker-entrypoint.sh"]
```

## Next.js Configuration for Standalone

```typescript
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  output: 'standalone',
  serverExternalPackages: ['postgres', 'pg-boss'],  // Keep out of bundle
  cacheComponents: true,
}

export default nextConfig
```

### Why `serverExternalPackages`?

Packages like `postgres` and `pg-boss` use native Node.js features that don't bundle well with webpack. Listing them here tells Next.js to keep them in `node_modules` instead of bundling.

## Monorepo Dockerfile (pnpm Workspaces)

```dockerfile
# Stage 1: Install all workspace deps
FROM node:22-alpine AS deps
RUN corepack enable && corepack prepare pnpm@10.4.1 --activate
WORKDIR /app

# Copy workspace structure (package.json files only for caching)
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc ./
COPY packages/shared/package.json packages/shared/
COPY frontend/package.json frontend/
RUN pnpm install --frozen-lockfile

# Stage 2: Build shared packages first, then app
FROM node:22-alpine AS builder
RUN corepack enable && corepack prepare pnpm@10.4.1 --activate
WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules
COPY --from=deps /app/packages/shared/node_modules ./packages/shared/node_modules
COPY --from=deps /app/frontend/node_modules ./frontend/node_modules
COPY . .

# Build shared first, then frontend
RUN pnpm --filter @myapp/shared build && \
    pnpm --filter @myapp/frontend build

# Stage 3: Production runner
FROM node:22-alpine AS runner
WORKDIR /app

ENV NODE_ENV=production
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nextjs

# Standalone preserves monorepo structure:
# .next/standalone/frontend/server.js
# .next/standalone/packages/shared/dist/
# .next/standalone/node_modules/
COPY --from=builder --chown=nextjs:nodejs /app/frontend/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/frontend/.next/static ./frontend/.next/static
COPY --from=builder --chown=nextjs:nodejs /app/frontend/public ./frontend/public

USER nextjs
EXPOSE 3001
ENV PORT=3001 HOSTNAME="0.0.0.0"

# Note: server.js is at frontend/server.js (not root)
CMD ["node", "frontend/server.js"]
```

### Monorepo next.config.ts

```typescript
// frontend/next.config.ts
import path from 'node:path'
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  output: 'standalone',
  outputFileTracingRoot: path.join(__dirname, '..'),  // repo root
  turbopack: {
    root: path.resolve(__dirname, '..'),
  },
}

export default nextConfig
```

Without `outputFileTracingRoot`, the standalone output won't include `packages/shared` and other workspace dependencies.

## Debian Slim Variant (Chromium/Playwright)

When you need headless browser support:

```dockerfile
# Stage 3: Production runner (Debian for system deps)
FROM node:22-slim AS runner
WORKDIR /app

ENV NODE_ENV=production
ENV DEBIAN_FRONTEND=noninteractive

# System deps for headless Chromium
RUN apt-get update && apt-get install -y --no-install-recommends \
    xvfb xauth \
    libnss3 libnspr4 libatk1.0-0 libatk-bridge2.0-0 \
    libcups2 libdrm2 libxkbcommon0 libxcomposite1 \
    libxdamage1 libxrandr2 libgbm1 libpango-1.0-0 \
    libcairo2 libasound2 libxshmfence1 \
    && rm -rf /var/lib/apt/lists/*

# Install Playwright Chromium
RUN npx playwright install chromium

RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nextjs

# ... (same COPY steps as Alpine variant)
```

## .dockerignore

Always include alongside the Dockerfile:

```
# Dependencies (installed in container)
node_modules
**/node_modules

# Build outputs (rebuilt in container)
.next
out
dist
**/.next
**/dist

# Env (injected at runtime)
.env*
!.env.example

# Git & IDE
.git
.gitignore
.DS_Store
.claude
.cursor
.vscode
*.pem
*.tsbuildinfo

# Tests
coverage
**/coverage
**/*.test.ts
**/*.spec.ts
**/__tests__
**/vitest.config.*

# Docs (non-essential)
*.md
!README.md
```

## Docker Compose Build Args

```yaml
# docker-compose.yml
services:
  frontend:
    build:
      context: .
      dockerfile: frontend/Dockerfile
      args:
        NEXT_PUBLIC_ROOT_DOMAIN: ${NEXT_PUBLIC_ROOT_DOMAIN:-example.com}
        NEXT_PUBLIC_R2_PUBLIC_URL: ${NEXT_PUBLIC_R2_PUBLIC_URL:-}
```

## Docker Smoke Test

Verify the build produces a runnable image:

```json
{
  "scripts": {
    "docker:build": "docker build -t myapp-test .",
    "docker:smoke": "pnpm docker:build && docker run --rm myapp-test sh -c 'node -e \"console.log(\\\"OK\\\")\"'"
  }
}
```
