---
name: pg-boss-jobs
description: pg-boss v12 job queue patterns with PostgreSQL — dual-client setup, typed job handlers, cron scheduling, staggered dispatch, retry logic, job chaining, health checks, and Docker worker deployment. Use when building background job infrastructure, worker processes, or queue-based scrape pipelines.
---

# pg-boss Job Queue Patterns

## Dual-Client Pattern

The web process (Next.js) uses a send-only client. The worker process uses a full client with scheduling and supervision enabled. Never enable the scheduler in the web process.

### Send-Only Client (API Routes)

```typescript
// worker/boss.ts — shared singleton for Next.js API routes
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
      await Promise.all([
        boss.createQueue('sailing-scrape'),
        boss.createQueue('sailing-report'),
        boss.createQueue('health-check'),
      ])
      return boss
    })
  }
  return globalForBoss.__pgBossReady
}
```

Why `globalThis`: Survives Next.js hot reloads in dev. Without it, each reload spawns a new pg-boss connection.

### Full Worker Client

```typescript
// worker/index.ts — standalone worker process
import PgBoss from 'pg-boss'

const boss = new PgBoss({
  connectionString: process.env.DATABASE_URL!,
  schema: 'pgboss',
  // defaults: schedule + supervise enabled
})

await boss.start()

// Create all queues before registering workers
await Promise.all([
  boss.createQueue('sailing-scrape'),
  boss.createQueue('sailing-report'),
  boss.createQueue('health-check'),
])
```

## Typed Job Inputs

Every queue has a TypeScript interface. No `any` types.

```typescript
// worker/jobs/sailing-scrape.ts
export interface SailingScrapeInput {
  shipSlug: string
  shipName: string
  shipCode: string
  cruiseLineSlug: string
  runId: string
  totalExpected: number
  dryRun?: boolean
}
```

## Registering Workers

pg-boss v12 delivers jobs as arrays. Process one at a time inside the handler.

```typescript
// worker/index.ts
import { handleSailingScrape } from './jobs/sailing-scrape'

await boss.work<SailingScrapeInput>('sailing-scrape', async (jobs) => {
  for (const job of jobs) {
    await handleSailingScrape(job.data, boss, logger)
  }
})

// Simple handler
await boss.work('health-check', async () => {
  logger.info('Health check processed')
})

// With batch size control
await boss.work('port-enrich', { batchSize: 1 }, async (jobs) => {
  for (const job of jobs) {
    await handlePortEnrich(job.data)
  }
})
```

## Sending Jobs

### Basic Send

```typescript
const boss = await getBoss()
const jobId = await boss.send('sailing-scrape', {
  shipSlug: 'carnival-celebration',
  shipName: 'Carnival Celebration',
  shipCode: 'CE',
  cruiseLineSlug: 'carnival',
  runId,
  totalExpected: 1,
})
```

### With Options

```typescript
await boss.send('sailing-scrape', jobData, {
  expireInSeconds: 600,   // expires if not completed in 10 min
  retryLimit: 1,          // retry once on failure
  retryDelay: 30,         // wait 30s before retry
})
```

### Staggered Dispatch (Rate Limiting)

Use `startAfter` to space out jobs and avoid overwhelming target sites:

```typescript
const DELAY_MS = 30_000  // 30s between jobs
const DISNEY_DELAY_MS = 2 * 60 * 1000  // Disney needs more spacing

for (let i = 0; i < ships.length; i++) {
  const delay = ships[i].cruiseLineSlug === 'disney' ? DISNEY_DELAY_MS : DELAY_MS
  await boss.send('sailing-scrape', {
    shipSlug: ships[i].slug,
    shipName: ships[i].name,
    shipCode: ships[i].shipCode,
    cruiseLineSlug: ships[i].cruiseLineSlug,
    runId,
    totalExpected: ships.length,
  }, {
    ...(i > 0 ? { startAfter: new Date(Date.now() + delay * i) } : {}),
  })
}
```

## Job Chaining

Jobs can enqueue downstream jobs on completion or failure:

```typescript
// worker/jobs/sailing-scrape.ts
export async function handleSailingScrape(
  data: SailingScrapeInput,
  boss: PgBoss,
  logger: Logger
) {
  try {
    const sailings = await scrapeSailings(data)

    // Persist results
    await db.insert(cruiseSailings).values(sailings)

    // Chain: send report job
    await boss.send('sailing-report', {
      runId: data.runId,
      totalExpected: data.totalExpected,
      shipSlug: data.shipSlug,
      cruiseLineSlug: data.cruiseLineSlug,
      success: true,
      sailingCount: sailings.length,
    })
  } catch (error) {
    // Chain even on failure — coordinator needs to know
    await boss.send('sailing-report', {
      runId: data.runId,
      totalExpected: data.totalExpected,
      shipSlug: data.shipSlug,
      cruiseLineSlug: data.cruiseLineSlug,
      success: false,
      error: String(error),
      sailingCount: 0,
    })
    throw error  // let pg-boss mark the job as failed
  }
}
```

## Cron Scheduling

Scheduled jobs run in the worker process only (where `schedule: true`).

```typescript
// worker/index.ts — schedule cron jobs after boss.start()

// Every 6 months: Jan 1 and Jul 1 at midnight
await boss.schedule('ship-scrape-cron', '0 0 1 1,7 *', {})

await boss.work('ship-scrape-cron', async () => {
  const runId = crypto.randomUUID()
  await db.insert(scrapeRuns).values({
    id: runId,
    type: 'ship',
    totalExpected: CRUISE_LINE_SLUGS.length,
  })

  for (const cruiseLineSlug of CRUISE_LINE_SLUGS) {
    await boss.send('ship-scrape', {
      cruiseLineSlug,
      runId,
      totalExpected: CRUISE_LINE_SLUGS.length,
    })
  }
})

// Hourly cleanup
await boss.schedule('export-cleanup', '0 * * * *', {})
```

## Run Tracking Schema

Track scrape runs at the application level alongside pg-boss internal state:

```typescript
// db/schema.ts
export const scrapeRuns = pgTable('scrape_runs', {
  id: text('id').primaryKey(),   // UUID
  type: text('type').notNull(),  // 'sailing' | 'ship' | 'review'
  cruiseLineSlug: text('cruise_line_slug'),
  totalExpected: integer('total_expected').notNull().default(0),
  totalCompleted: integer('total_completed').notNull().default(0),
  totalSucceeded: integer('total_succeeded').notNull().default(0),
  totalFailed: integer('total_failed').notNull().default(0),
  status: text('status').notNull().default('running'),  // running | completed | failed
  startedAt: timestamp('started_at').defaultNow(),
  completedAt: timestamp('completed_at'),
})
```

## Health Check Pattern

End-to-end test that the worker is alive and processing jobs:

```typescript
// app/api/health-test/route.ts
export async function GET() {
  const boss = await getBoss()
  const jobId = await boss.send('health-check', { ts: Date.now() })
  if (!jobId) throw new Error('Failed to enqueue health-check job')

  // Poll for completion (up to 10s)
  for (let i = 0; i < 50; i++) {
    await new Promise(r => setTimeout(r, 200))
    const job = await boss.getJobById('health-check', jobId)
    if (!job) throw new Error('Job disappeared')
    if (job.state === 'completed') {
      return Response.json({ status: 'ok', worker: 'alive' })
    }
    if (job.state === 'failed') {
      throw new Error('Health-check job failed in worker')
    }
  }
  throw new Error('Health-check job timed out (10s)')
}
```

## Docker Deployment

Run worker and Next.js in the same container with auto-restart:

```bash
#!/bin/sh
# docker-entrypoint.sh
set -e

echo "Running database migrations..."
npx tsx db/migrate.ts

echo "Starting worker..."
(
  while true; do
    node --import tsx worker/index.ts
    EXIT_CODE=$?
    echo "Worker exited with code $EXIT_CODE, restarting in 5s..."
    sleep 5
  done
) &

echo "Starting Next.js..."
exec node server.js
```

```dockerfile
# Dockerfile — relevant section
COPY docker-entrypoint.sh /app/docker-entrypoint.sh
RUN chmod +x /app/docker-entrypoint.sh
ENTRYPOINT ["/app/docker-entrypoint.sh"]
```

## Job States Reference

| State | Meaning |
|-------|---------|
| `created` | Queued, waiting to start |
| `active` | Currently being processed by a worker |
| `completed` | Successfully finished |
| `failed` | Processing threw an error |
| `retry` | Failed, waiting to retry |
| `expired` | Timed out before completion |
| `cancelled` | Manually cancelled |

## Querying Job Status Directly

For admin dashboards, query pg-boss's internal tables:

```typescript
const result = await db.execute(sql`
  SELECT id::text, name, state, data, created_on, started_on, completed_on, output
  FROM pgboss.job
  WHERE name IN ('sailing-scrape', 'sailing-report', 'ship-scrape')
  ORDER BY created_on DESC
  LIMIT ${limit}
`)

// Filter by runId in job data
const runJobs = await db.execute(sql`
  SELECT id::text, name, state, data, created_on, started_on, completed_on, output
  FROM pgboss.job
  WHERE data->>'runId' = ${runId}
  ORDER BY created_on DESC
`)
```
