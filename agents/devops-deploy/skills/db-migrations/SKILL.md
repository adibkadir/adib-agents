---
name: db-migrations
description: Database migration strategies for Docker deployments — Drizzle ORM migrate scripts, drizzle-kit push for init containers, schema verification, single-connection pools, and migration ordering in entrypoints and docker-compose. Use when setting up database migrations for containerized applications.
---

# Database Migrations in Docker

## Strategy Overview

| Strategy | When to Use | How |
|----------|------------|-----|
| **Entrypoint migration** | Single-container apps | `npx tsx db/migrate.ts` before starting Next.js |
| **Init container** | Multi-service docker-compose | Separate service with `service_completed_successfully` |
| **drizzle-kit push** | Quick deploys (no migration files) | `drizzle-kit push --force` in init container |
| **drizzle-kit migrate** | Versioned migration files | `drizzle-kit migrate` for sequential SQL files |

## Drizzle Migrate Script

Used in entrypoint scripts for single-container deployments:

```typescript
// db/migrate.ts
import { drizzle } from 'drizzle-orm/postgres-js'
import { migrate } from 'drizzle-orm/postgres-js/migrator'
import postgres from 'postgres'

async function main() {
  const url = process.env.DATABASE_URL
  if (!url) {
    console.error('[migrate] DATABASE_URL not set')
    process.exit(1)
  }

  // Log connection info (without password)
  const parsed = new URL(url)
  console.log(`[migrate] Connecting to ${parsed.hostname}:${parsed.port || 5432}${parsed.pathname}`)

  // Single connection — prevents race conditions during migration
  const client = postgres(url, { max: 1 })
  const db = drizzle(client)

  console.log('[migrate] Running migrations...')
  await migrate(db, { migrationsFolder: './db/drizzle' })
  console.log('[migrate] Migrations complete')

  await client.end()
}

main().catch((err) => {
  console.error('[migrate] Migration failed:', err)
  process.exit(1)
})
```

### Why `max: 1`?

Migrations must run sequentially. A connection pool with multiple connections could interleave DDL statements and cause deadlocks. Always use a single connection for migrations.

## Migration with Schema Verification

Verify all expected tables exist after migration:

```typescript
// db/migrate.ts (enhanced)
import { sql } from 'drizzle-orm'

async function main() {
  const client = postgres(process.env.DATABASE_URL!, { max: 1 })
  const db = drizzle(client)

  await migrate(db, { migrationsFolder: './drizzle' })

  // Verify tables exist
  const expected = ['user', 'session', 'account', 'posts', 'comments', 'media']

  const result = await db.execute(
    sql`SELECT table_name FROM information_schema.tables WHERE table_schema = 'public'`
  )
  const existing = new Set(
    (result.rows as { table_name: string }[]).map(r => r.table_name)
  )

  const missing = expected.filter(t => !existing.has(t))
  if (missing.length > 0) {
    throw new Error(`Migration verification failed — missing tables: ${missing.join(', ')}`)
  }

  console.log(`[migrate] Verified ${expected.length} tables exist`)
  await client.end()
}
```

## Init Container (docker-compose)

A separate lightweight container that runs migrations and exits:

```dockerfile
# Dockerfile.db-push
FROM node:22-alpine
RUN corepack enable && corepack prepare pnpm@10.4.1 --activate
WORKDIR /app

# Install only shared package deps
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc ./
COPY packages/shared/package.json packages/shared/
RUN pnpm install --frozen-lockfile --filter @myapp/shared

COPY packages/shared packages/shared
WORKDIR /app/packages/shared

CMD ["pnpm", "exec", "drizzle-kit", "push", "--force"]
```

```yaml
# docker-compose.yml
services:
  db-push:
    build:
      context: .
      dockerfile: packages/shared/Dockerfile.db-push
    env_file:
      - .env
    networks:
      - dokploy-network

  frontend:
    depends_on:
      db-push:
        condition: service_completed_successfully
    # ... rest of frontend config
```

### `push` vs `migrate`

| Command | Behavior | Use When |
|---------|----------|----------|
| `drizzle-kit push` | Compares schema → DB, applies diffs | Fast iteration, no migration files needed |
| `drizzle-kit migrate` | Runs SQL files from `drizzle/` folder | Production, versioned history, rollback support |

`push --force` skips interactive prompts — required for automated environments. Use `migrate` when you need a reviewable migration history.

## Drizzle Config

```typescript
// drizzle.config.ts
import 'dotenv/config'
import { defineConfig } from 'drizzle-kit'

export default defineConfig({
  schema: './db/schema.ts',       // or './src/db/schema.ts'
  out: './db/drizzle',            // migration output directory
  dialect: 'postgresql',
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
})
```

## Migration Workflow

### Development

```bash
# 1. Edit schema.ts
# 2. Generate migration
pnpm drizzle-kit generate

# 3. Review generated SQL in db/drizzle/XXXX_migration_name.sql
# 4. Apply to local DB
pnpm drizzle-kit migrate

# 5. Verify with Drizzle Studio
pnpm drizzle-kit studio
```

### Production (via Docker)

```
Container starts
  → docker-entrypoint.sh
    → npx tsx db/migrate.ts
      → Connects with max: 1
      → Runs all pending migrations
      → Verifies schema
      → Exits cleanly
    → Worker starts (background)
    → Next.js starts (foreground)
```

## Package.json Scripts

```json
{
  "scripts": {
    "db:generate": "drizzle-kit generate",
    "db:migrate": "tsx db/migrate.ts",
    "db:push": "drizzle-kit push",
    "db:studio": "drizzle-kit studio",
    "db:seed": "tsx db/seed.ts",
    "build": "drizzle-kit migrate && next build"
  }
}
```

Note: `build` runs migrations before Next.js build when running on platforms like Vercel that execute the build command before starting the app.

## Common Pitfalls

| Problem | Cause | Fix |
|---------|-------|-----|
| Migration hangs | Connection pool has multiple connections | Use `max: 1` |
| "relation does not exist" after deploy | Migrations didn't run | Check entrypoint logs, ensure migration runs before `node server.js` |
| Push changes wrong schema | Wrong DATABASE_URL | Log the hostname before running |
| Build fails needing DATABASE_URL | Next.js tries to connect at build | Use dummy URL: `DATABASE_URL=postgresql://dummy:dummy@localhost/dummy` |
| Init container restarts forever | `restart: unless-stopped` on db-push | Don't set restart policy on init containers |
