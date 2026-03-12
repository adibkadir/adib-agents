---
name: worker-entrypoint
description: Docker entrypoint scripts for running background workers alongside Next.js — auto-restart loops, Xvfb for headless browsers, migration execution, signal handling with exec, and health check endpoints. Use when a container needs to run both a web server and a background worker process.
---

# Worker Entrypoint Scripts

## Standard Entrypoint (Next.js + pg-boss Worker)

```bash
#!/bin/sh
# docker-entrypoint.sh
set -e

echo "[entrypoint] Running database migrations..."
npx tsx db/migrate.ts

echo "[entrypoint] Starting worker..."
(
  while true; do
    node --import tsx worker/index.ts
    EXIT_CODE=$?
    echo "[entrypoint] Worker exited with code $EXIT_CODE, restarting in 5s..."
    sleep 5
  done
) &

echo "[entrypoint] Starting Next.js..."
exec node server.js
```

### Why This Structure

1. **Migrations first** — Synchronous, blocks until complete. If they fail, `set -e` stops the container.
2. **Worker in background** — Runs in a subshell with auto-restart loop (`while true`). The 5s delay prevents crash loops from hammering the system.
3. **`exec` for Next.js** — Replaces the shell process (PID 1). This means SIGTERM from Docker goes directly to Node.js for clean shutdown.
4. **`set -e`** — Exits immediately if any command fails. Critical for migration failures.

## Entrypoint with Xvfb (Playwright/Chromium)

For containers that need headless browser automation:

```bash
#!/bin/sh
# docker-entrypoint.sh
set -e

echo "[entrypoint] Starting Xvfb..."
Xvfb :99 -screen 0 1920x1080x24 -nolisten tcp &
export DISPLAY=:99

echo "[entrypoint] Running database migrations..."
npx tsx db/migrate.ts

echo "[entrypoint] Starting worker..."
(
  while true; do
    node --import tsx worker/index.ts
    EXIT_CODE=$?
    echo "[entrypoint] Worker exited with code $EXIT_CODE, restarting in 5s..."
    sleep 5
  done
) &

echo "[entrypoint] Starting Next.js..."
exec node server.js
```

Xvfb provides a virtual X display. Playwright/Puppeteer needs this to run Chromium, even in headless mode, because some APIs still require an X server.

## Simple Entrypoint (No Worker)

When there's no background worker:

```bash
#!/bin/sh
set -e

echo "[entrypoint] Running database migrations..."
npx tsx db/migrate.ts

echo "[entrypoint] Starting Next.js..."
exec node server.js
```

## Motia Workflow Engine Entrypoint

```bash
#!/bin/sh
set -e

# Virtual display for Puppeteer
Xvfb :99 -screen 0 1280x1024x24 -nolisten tcp &
export DISPLAY=:99
sleep 1

# iii is the Motia runtime binary
exec iii --config config-production.yaml
```

## Dockerfile Integration

```dockerfile
# In the runner stage
COPY --chown=nextjs:nodejs docker-entrypoint.sh ./
RUN chmod +x docker-entrypoint.sh

USER nextjs
EXPOSE 3000
ENV PORT=3000 HOSTNAME="0.0.0.0"

ENTRYPOINT ["./docker-entrypoint.sh"]
```

## Health Check Endpoint

Verify the worker is alive and processing jobs:

```typescript
// app/api/health-test/route.ts
import { getBoss } from '@/worker/boss'

export async function GET() {
  const checks: Record<string, string> = {}

  // Check 1: Database connectivity
  try {
    await db.execute(sql`SELECT 1`)
    checks.database = 'ok'
  } catch {
    checks.database = 'fail'
  }

  // Check 2: Worker is processing (pg-boss e2e)
  try {
    const boss = await getBoss()
    const jobId = await boss.send('health-check', { ts: Date.now() })
    if (!jobId) throw new Error('Failed to enqueue')

    let completed = false
    for (let i = 0; i < 50; i++) {
      await new Promise(r => setTimeout(r, 200))
      const job = await boss.getJobById('health-check', jobId)
      if (!job) throw new Error('Job disappeared')
      if (job.state === 'completed') { completed = true; break }
      if (job.state === 'failed') throw new Error('Job failed')
    }

    checks.worker = completed ? 'ok' : 'timeout'
  } catch (e) {
    checks.worker = `fail: ${e instanceof Error ? e.message : 'unknown'}`
  }

  const allOk = Object.values(checks).every(v => v === 'ok')
  return Response.json(
    { status: allOk ? 'healthy' : 'degraded', checks },
    { status: allOk ? 200 : 503 }
  )
}
```

## Graceful Shutdown in Worker

```typescript
// worker/index.ts
const shutdown = async () => {
  logger.info('Shutting down worker...')
  await boss.stop({ graceful: true, timeout: 30_000 })
  process.exit(0)
}

process.on('SIGTERM', shutdown)
process.on('SIGINT', shutdown)
```

pg-boss `graceful: true` waits for in-flight jobs to complete (up to 30s) before shutting down. Combined with `exec` in the entrypoint, SIGTERM from Docker propagates correctly.

## Dev Mode: Running Worker + Next.js

### Option A: Background process

```json
{
  "scripts": {
    "dev": "next dev & dotenvx run -f .env.local -- tsx --watch worker/index.ts & wait"
  }
}
```

### Option B: concurrently (recommended)

```json
{
  "scripts": {
    "dev": "concurrently --kill-others-on-fail -n next,worker -c blue,green \"next dev\" \"pnpm worker:dev\"",
    "worker:dev": "dotenv -e .env.local -- tsx worker/index.ts"
  }
}
```

`--kill-others-on-fail` ensures that if either process crashes, both stop — preventing a zombie Next.js process with no worker.

## Common Pitfalls

| Problem | Cause | Fix |
|---------|-------|-----|
| Worker crashes silently | No auto-restart | Add `while true; do ... sleep 5; done` loop |
| Docker stop takes 10s | Shell is PID 1, not Node | Use `exec node server.js` |
| Migrations fail in CI | `DATABASE_URL` not set | Add dummy URL for build, real URL at runtime |
| Worker can't find tsx | Not in PATH | Use `node --import tsx` or `npx tsx` |
| Xvfb not starting | Debian deps missing | Install `xvfb xauth` and system libraries |
