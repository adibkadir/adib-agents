---
name: pg-boss-jobs
description: "pg-boss v12 job queue integration for scrapers — dual-client setup, typed job handlers, cron scheduling, staggered dispatch, retry logic, job chaining, health checks, and Docker worker deployment. Assumes PostgreSQL with DATABASE_URL. If no DATABASE_URL found, ask the user how to manage background jobs."
---

# pg-boss Job Queue Integration

When scrapers need to run on a schedule or be dispatched via API, wrap them in pg-boss job handlers. Assumes the project uses PostgreSQL — look for `DATABASE_URL` in `.env` or environment. If not found, **ask the user** how they want to manage workers and background jobs.

## Pre-Check

Before setting up pg-boss:

```bash
# Check if DATABASE_URL exists
grep -r 'DATABASE_URL' .env .env.local .env.* 2>/dev/null

# Check if pg-boss is already installed
grep 'pg-boss' package.json
```

If `DATABASE_URL` is found → proceed with pg-boss setup.
If not found → ask: "I don't see a PostgreSQL DATABASE_URL. How would you like to manage background jobs? Options: (1) Set up PostgreSQL + pg-boss, (2) Use a simpler approach like node-cron, (3) Skip background jobs for now."

## Dual-Client Pattern

The web process (Next.js) uses a send-only client. The worker process uses a full client with scheduling.

### Send-Only Client (API Routes)

```typescript
// worker/boss.ts — singleton for Next.js API routes
import PgBoss from 'pg-boss'

const globalForBoss = globalThis as unknown as {
  __pgBoss?: PgBoss
  __pgBossReady?: Promise<PgBoss>
}

export function getBoss(): Promise<PgBoss> {
  if (!globalForBoss.__pgBossReady) {
    const boss = new PgBoss({
      connectionString: process.env.DATABASE_URL!,
      schema: 'pgboss',
      schedule: false,    // no cron in web process
      supervise: false,   // no maintenance in web process
    })
    globalForBoss.__pgBoss = boss
    globalForBoss.__pgBossReady = boss.start().then(async () => {
      // Create all queues used by scrapers
      await Promise.all([
        boss.createQueue('scraper-execute'),
        boss.createQueue('scraper-report'),
        boss.createQueue('health-check'),
      ])
      return boss
    })
  }
  return globalForBoss.__pgBossReady
}
```

### Full Worker Client

```typescript
// worker/index.ts — standalone worker process
import PgBoss from 'pg-boss'
import { handleScraperExecute } from './jobs/scraper-execute'
import { handleScraperReport } from './jobs/scraper-report'

const boss = new PgBoss({
  connectionString: process.env.DATABASE_URL!,
  schema: 'pgboss',
})

await boss.start()

await Promise.all([
  boss.createQueue('scraper-execute'),
  boss.createQueue('scraper-report'),
  boss.createQueue('health-check'),
])

// Register job handlers
await boss.work<ScraperExecuteInput>('scraper-execute', async (jobs) => {
  for (const job of jobs) {
    await handleScraperExecute(job.data, boss)
  }
})

await boss.work<ScraperReportInput>('scraper-report', async (jobs) => {
  for (const job of jobs) {
    await handleScraperReport(job.data)
  }
})

await boss.work('health-check', async () => {
  console.log('Health check OK')
})

console.log('Worker running. Waiting for jobs...')
```

## Typed Job Inputs

```typescript
// worker/jobs/scraper-execute.ts
export interface ScraperExecuteInput {
  scraperId: string
  params: Record<string, string>
  runId: string
  totalExpected: number
}

export async function handleScraperExecute(data: ScraperExecuteInput, boss: PgBoss) {
  const { scraperId, params, runId, totalExpected } = data

  try {
    // Dynamic import the scraper from registry
    const { SCRAPER_REGISTRY } = await import('@/app/admin/scrapers/scraper-registry')
    const scraper = SCRAPER_REGISTRY.find(s => s.id === scraperId)
    if (!scraper) throw new Error(`Scraper '${scraperId}' not found`)

    const result = await scraper.execute(params)

    // Chain: report success
    await boss.send('scraper-report', {
      runId,
      scraperId,
      totalExpected,
      success: result.success,
      recordCount: result.meta?.count ?? 0,
      error: result.error,
    })
  } catch (error) {
    // Chain: report failure
    await boss.send('scraper-report', {
      runId,
      scraperId,
      totalExpected,
      success: false,
      recordCount: 0,
      error: String(error),
    })
    throw error
  }
}
```

## Sending Jobs (from API or Admin UI)

```typescript
// Dispatch a single scraper job
const boss = await getBoss()
const runId = crypto.randomUUID()

await boss.send('scraper-execute', {
  scraperId: 'example-sailings',
  params: { shipCode: 'CE', guests: '2' },
  runId,
  totalExpected: 1,
})

// Dispatch multiple scrapers with staggered timing
const DELAY_MS = 30_000

for (let i = 0; i < scraperIds.length; i++) {
  await boss.send('scraper-execute', {
    scraperId: scraperIds[i],
    params: {},
    runId,
    totalExpected: scraperIds.length,
  }, {
    ...(i > 0 ? { startAfter: new Date(Date.now() + DELAY_MS * i) } : {}),
    expireInSeconds: 600,
    retryLimit: 1,
    retryDelay: 30,
  })
}
```

## Cron Scheduling

```typescript
// worker/index.ts — register cron after boss.start()

// Daily at 2am
await boss.schedule('daily-scrape', '0 2 * * *', {})

await boss.work('daily-scrape', async () => {
  const runId = crypto.randomUUID()
  const scraperIds = ['example-sailings', 'example-ships']

  for (let i = 0; i < scraperIds.length; i++) {
    await boss.send('scraper-execute', {
      scraperId: scraperIds[i],
      params: {},
      runId,
      totalExpected: scraperIds.length,
    }, {
      ...(i > 0 ? { startAfter: new Date(Date.now() + 30_000 * i) } : {}),
    })
  }
})
```

## Run Tracking Schema

```typescript
// db/schema.ts
import { pgTable, text, integer, timestamp } from 'drizzle-orm/pg-core'

export const scrapeRuns = pgTable('scrape_runs', {
  id: text('id').primaryKey(),
  type: text('type').notNull(),
  totalExpected: integer('total_expected').notNull().default(0),
  totalCompleted: integer('total_completed').notNull().default(0),
  totalSucceeded: integer('total_succeeded').notNull().default(0),
  totalFailed: integer('total_failed').notNull().default(0),
  status: text('status').notNull().default('running'),
  startedAt: timestamp('started_at').defaultNow(),
  completedAt: timestamp('completed_at'),
})
```

## Health Check

```typescript
// app/api/health-test/route.ts
export async function GET() {
  const boss = await getBoss()
  const jobId = await boss.send('health-check', { ts: Date.now() })
  if (!jobId) throw new Error('Failed to enqueue health-check job')

  for (let i = 0; i < 50; i++) {
    await new Promise(r => setTimeout(r, 200))
    const job = await boss.getJobById('health-check', jobId)
    if (!job) throw new Error('Job disappeared')
    if (job.state === 'completed') return Response.json({ status: 'ok' })
    if (job.state === 'failed') throw new Error('Job failed')
  }
  throw new Error('Timed out')
}
```

## Docker Worker Deployment

```bash
#!/bin/sh
# docker-entrypoint.sh
set -e

echo "Running migrations..."
npx tsx db/migrate.ts

echo "Starting worker..."
(
  while true; do
    node --import tsx worker/index.ts
    EXIT_CODE=$?
    echo "Worker exited ($EXIT_CODE), restarting in 5s..."
    sleep 5
  done
) &

echo "Starting Next.js..."
exec node server.js
```

## Job States Reference

| State | Meaning |
|-------|---------|
| `created` | Queued, waiting |
| `active` | Being processed |
| `completed` | Finished successfully |
| `failed` | Processing threw an error |
| `retry` | Failed, waiting to retry |
| `expired` | Timed out |
| `cancelled` | Manually cancelled |
