---
name: docker-nextjs
description: Docker multi-stage build for Next.js standalone output with Alpine Node 20. Use when containerizing a Next.js application for production deployment.
---

# Docker Multi-Stage Build for Next.js

## Standard Dockerfile

```dockerfile
# Stage 1: Dependencies
FROM node:20-alpine AS deps
RUN corepack enable
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile --prod=false

# Stage 2: Build
FROM node:20-alpine AS builder
RUN corepack enable
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Build arguments for environment
ARG DATABASE_URL
ARG NEXT_PUBLIC_APP_URL
ENV DATABASE_URL=$DATABASE_URL
ENV NEXT_PUBLIC_APP_URL=$NEXT_PUBLIC_APP_URL

RUN pnpm build

# Stage 3: Production runtime
FROM node:20-slim AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV HOSTNAME=0.0.0.0
ENV PORT=3000

# Security: non-root user
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nextjs

# Copy standalone output
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public

USER nextjs
EXPOSE 3000

CMD ["node", "server.js"]
```

## next.config.ts for Standalone

```typescript
const nextConfig = {
  output: 'standalone',
  // Required for Docker: trust proxy headers
  serverExternalPackages: ['pg', 'postgres'],
}
```

## With Worker Process (pg-boss)

For apps that need a separate worker alongside the web server:

```dockerfile
# ... same deps and builder stages ...

# Web server
FROM node:20-slim AS web
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public
EXPOSE 3000
CMD ["node", "server.js"]

# Worker process
FROM node:20-slim AS worker
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/worker ./worker
CMD ["node", "worker/index.js"]
```

## Docker Compose

```yaml
services:
  web:
    build:
      context: .
      target: web
    ports:
      - "3000:3000"
    env_file: .env
    depends_on:
      - db
    restart: unless-stopped

  worker:
    build:
      context: .
      target: worker
    env_file: .env
    depends_on:
      - db
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"

volumes:
  pgdata:
```
