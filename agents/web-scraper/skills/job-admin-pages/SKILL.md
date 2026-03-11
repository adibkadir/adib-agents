---
name: job-admin-pages
description: Next.js admin dashboard pages for monitoring and managing pg-boss job queues — API routes for job submission and status, admin-only access via better-auth, real-time polling, run detail views, and manual dispatch forms. Use when building job monitoring UI, admin pages for scrape pipelines, or job management endpoints.
---

# Job Admin Pages

## Architecture Overview

The admin jobs system has three layers:

1. **API routes** — Admin-protected endpoints for submitting jobs and querying status
2. **Admin page** — Server component shell with client component for real-time updates
3. **Client view** — Tabbed interface with job monitoring, run details, and manual dispatch

```
/admin/jobs (page.tsx — server component, auth check)
  └── JobsView.tsx (client component, polls API)
      ├── Tab: Job Monitor (all pg-boss jobs, filterable by queue)
      ├── Tab: Scrape Runs (application-level run tracking)
      └── Tab: Manual Dispatch (forms to trigger jobs)

/api/jobs (route.ts — GET for status, POST for dispatch)
```

## Admin Auth Guard

All job endpoints require admin role via better-auth:

```typescript
// lib/api-auth.ts
import { auth } from '@/lib/auth'

export async function requireAdminApi(request: Request): Promise<Response | null> {
  const session = await auth.api.getSession({ headers: request.headers })

  if (!session?.user) {
    return Response.json({ error: 'Unauthorized' }, { status: 401 })
  }

  if (session.user.role !== 'admin') {
    return Response.json({ error: 'Forbidden' }, { status: 403 })
  }

  return null // null = authorized, proceed
}
```

Usage in API routes:

```typescript
export async function GET(request: Request) {
  const denied = await requireAdminApi(request)
  if (denied) return denied

  // ... admin-only logic
}
```

## API Route: Job Status & Submission

```typescript
// app/api/jobs/route.ts
import { getBoss } from '@/worker/boss'
import { requireAdminApi } from '@/lib/api-auth'
import { db } from '@/db'
import { scrapeRuns, sailingScrapeJobs } from '@/db/schema'
import { eq, desc, sql } from 'drizzle-orm'

export async function GET(request: Request) {
  const denied = await requireAdminApi(request)
  if (denied) return denied

  const { searchParams } = new URL(request.url)
  const limit = Number(searchParams.get('limit') ?? 50)
  const queue = searchParams.get('queue')
  const runId = searchParams.get('runId')

  // If requesting a specific run, return run + its jobs
  if (runId) {
    const [run] = await db.select().from(scrapeRuns)
      .where(eq(scrapeRuns.id, runId)).limit(1)

    const scrapeJobs = await db.select().from(sailingScrapeJobs)
      .where(eq(sailingScrapeJobs.runId, runId))
      .orderBy(desc(sailingScrapeJobs.startedAt))

    // Also get pg-boss jobs for this run
    const pgBossJobs = await db.execute(sql`
      SELECT id::text, name, state, data, created_on, started_on, completed_on, output
      FROM pgboss.job
      WHERE data->>'runId' = ${runId}
      ORDER BY created_on DESC
    `)

    return Response.json({ run, scrapeJobs, pgBossJobs })
  }

  // Default: return recent pg-boss jobs
  let jobs: unknown[] = []
  try {
    const whereClause = queue
      ? sql`WHERE name = ${queue}`
      : sql`WHERE name IN ('sailing-scrape', 'sailing-report', 'ship-scrape', 'ship-report', 'health-check')`

    const result = await db.execute(sql`
      SELECT id::text, name, state, data, created_on, started_on, completed_on, output
      FROM pgboss.job
      ${whereClause}
      ORDER BY created_on DESC
      LIMIT ${limit}
    `)
    jobs = result as unknown as unknown[]
  } catch {
    // pgboss schema might not exist yet
  }

  // Also return recent scrape runs
  const runs = await db.select().from(scrapeRuns)
    .orderBy(desc(scrapeRuns.startedAt))
    .limit(20)

  return Response.json({ jobs, runs })
}

export async function POST(request: Request) {
  const denied = await requireAdminApi(request)
  if (denied) return denied

  const body = await request.json()
  const { action, cruiseLineSlug, shipSlug } = body

  const boss = await getBoss()

  if (action === 'scrape-sailings' && shipSlug) {
    const [ship] = await db.select().from(cruiseShips)
      .where(sql`${cruiseShips.slug} = ${shipSlug}`)
      .limit(1)

    if (!ship) return Response.json({ error: 'Ship not found' }, { status: 404 })

    const runId = crypto.randomUUID()
    await db.insert(scrapeRuns).values({
      id: runId,
      type: 'sailing',
      cruiseLineSlug: ship.cruiseLineSlug,
      totalExpected: 1,
    })

    await boss.send('sailing-scrape', {
      shipSlug: ship.slug,
      shipName: ship.name,
      shipCode: ship.shipCode,
      cruiseLineSlug: ship.cruiseLineSlug,
      runId,
      totalExpected: 1,
    })

    return Response.json({ message: `Enqueued sailing scrape for ${ship.name}`, runId })
  }

  if (action === 'scrape-all' && cruiseLineSlug) {
    // Dispatch all ships for a cruise line with staggered timing
    const ships = await db.select().from(cruiseShips)
      .where(eq(cruiseShips.cruiseLineSlug, cruiseLineSlug))

    const runId = crypto.randomUUID()
    await db.insert(scrapeRuns).values({
      id: runId,
      type: 'sailing',
      cruiseLineSlug,
      totalExpected: ships.length,
    })

    const delay = cruiseLineSlug === 'disney' ? 2 * 60 * 1000 : 30_000
    for (let i = 0; i < ships.length; i++) {
      await boss.send('sailing-scrape', {
        shipSlug: ships[i].slug,
        shipName: ships[i].name,
        shipCode: ships[i].shipCode,
        cruiseLineSlug,
        runId,
        totalExpected: ships.length,
      }, {
        ...(i > 0 ? { startAfter: new Date(Date.now() + delay * i) } : {}),
      })
    }

    return Response.json({
      message: `Enqueued ${ships.length} ships for ${cruiseLineSlug}`,
      runId,
    })
  }

  return Response.json({ error: 'Invalid action' }, { status: 400 })
}
```

## Admin Page (Server Component)

```typescript
// app/admin/jobs/page.tsx
import { redirect } from 'next/navigation'
import { auth } from '@/lib/auth'
import { headers } from 'next/headers'
import { JobsView } from './JobsView'

export default async function AdminJobsPage() {
  const session = await auth.api.getSession({
    headers: await headers(),
  })

  if (!session?.user || session.user.role !== 'admin') {
    redirect('/')
  }

  return (
    <div className="p-6">
      <h1 className="text-2xl font-bold mb-6">Job Queue Monitor</h1>
      <JobsView />
    </div>
  )
}
```

## Jobs View (Client Component)

```tsx
'use client'

import { useState, useEffect, useCallback } from 'react'

const QUEUE_NAMES = [
  'sailing-scrape',
  'sailing-report',
  'ship-scrape',
  'ship-report',
  'ship-enrich',
  'health-check',
  'port-attractions-generate',
  'comment-moderate',
] as const

const STATE_COLORS: Record<string, string> = {
  created: 'text-zinc-400',
  retry: 'text-amber-400',
  active: 'text-blue-400',
  completed: 'text-emerald-400',
  cancelled: 'text-zinc-500',
  failed: 'text-red-400',
  expired: 'text-orange-400',
}

type Tab = 'jobs' | 'runs' | 'dispatch'

interface PgBossJob {
  id: string
  name: string
  state: string
  data: Record<string, unknown>
  created_on: string
  started_on: string | null
  completed_on: string | null
  output: unknown
}

interface ScrapeRun {
  id: string
  type: string
  cruiseLineSlug: string | null
  totalExpected: number
  totalCompleted: number
  totalSucceeded: number
  totalFailed: number
  status: string
  startedAt: string
  completedAt: string | null
}

export function JobsView() {
  const [tab, setTab] = useState<Tab>('jobs')
  const [jobs, setJobs] = useState<PgBossJob[]>([])
  const [runs, setRuns] = useState<ScrapeRun[]>([])
  const [queueFilter, setQueueFilter] = useState<string>('all')
  const [loading, setLoading] = useState(true)

  const fetchData = useCallback(async () => {
    const params = new URLSearchParams()
    if (queueFilter !== 'all') params.set('queue', queueFilter)

    const res = await fetch(`/api/jobs?${params}`)
    if (res.ok) {
      const data = await res.json()
      setJobs(data.jobs ?? [])
      setRuns(data.runs ?? [])
    }
    setLoading(false)
  }, [queueFilter])

  // Poll every 5 seconds
  useEffect(() => {
    fetchData()
    const interval = setInterval(fetchData, 5000)
    return () => clearInterval(interval)
  }, [fetchData])

  return (
    <div>
      {/* Tab bar */}
      <div className="flex gap-2 mb-6 border-b border-zinc-800 pb-2">
        {(['jobs', 'runs', 'dispatch'] as Tab[]).map(t => (
          <button
            key={t}
            onClick={() => setTab(t)}
            className={`px-4 py-2 rounded-t text-sm font-medium ${
              tab === t ? 'bg-zinc-800 text-white' : 'text-zinc-500 hover:text-zinc-300'
            }`}
          >
            {t === 'jobs' ? 'Job Monitor' : t === 'runs' ? 'Scrape Runs' : 'Manual Dispatch'}
          </button>
        ))}
      </div>

      {tab === 'jobs' && (
        <JobMonitorTab
          jobs={jobs}
          queueFilter={queueFilter}
          onQueueFilterChange={setQueueFilter}
          loading={loading}
        />
      )}
      {tab === 'runs' && <RunsTab runs={runs} />}
      {tab === 'dispatch' && <DispatchTab onSubmit={fetchData} />}
    </div>
  )
}

function JobMonitorTab({
  jobs,
  queueFilter,
  onQueueFilterChange,
  loading,
}: {
  jobs: PgBossJob[]
  queueFilter: string
  onQueueFilterChange: (q: string) => void
  loading: boolean
}) {
  return (
    <div>
      {/* Queue filter */}
      <div className="flex gap-2 mb-4 flex-wrap">
        <button
          onClick={() => onQueueFilterChange('all')}
          className={`px-3 py-1 rounded text-xs ${
            queueFilter === 'all' ? 'bg-zinc-700 text-white' : 'bg-zinc-900 text-zinc-400'
          }`}
        >
          All
        </button>
        {QUEUE_NAMES.map(q => (
          <button
            key={q}
            onClick={() => onQueueFilterChange(q)}
            className={`px-3 py-1 rounded text-xs ${
              queueFilter === q ? 'bg-zinc-700 text-white' : 'bg-zinc-900 text-zinc-400'
            }`}
          >
            {q}
          </button>
        ))}
      </div>

      {loading ? (
        <p className="text-zinc-500">Loading...</p>
      ) : (
        <table className="w-full text-sm">
          <thead>
            <tr className="text-left text-zinc-500 border-b border-zinc-800">
              <th className="py-2">Queue</th>
              <th>State</th>
              <th>Created</th>
              <th>Duration</th>
              <th>Data</th>
            </tr>
          </thead>
          <tbody>
            {jobs.map(job => (
              <tr key={job.id} className="border-b border-zinc-900">
                <td className="py-2 font-mono text-xs">{job.name}</td>
                <td className={STATE_COLORS[job.state] ?? 'text-zinc-400'}>
                  {job.state}
                </td>
                <td className="text-zinc-500 text-xs">
                  {new Date(job.created_on).toLocaleString()}
                </td>
                <td className="text-zinc-500 text-xs">
                  {job.started_on && job.completed_on
                    ? `${((new Date(job.completed_on).getTime() - new Date(job.started_on).getTime()) / 1000).toFixed(1)}s`
                    : '—'}
                </td>
                <td className="text-zinc-600 text-xs max-w-xs truncate">
                  {JSON.stringify(job.data).slice(0, 100)}
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      )}
    </div>
  )
}

function RunsTab({ runs }: { runs: ScrapeRun[] }) {
  return (
    <div className="space-y-3">
      {runs.map(run => (
        <div key={run.id} className="bg-zinc-900 rounded-lg p-4">
          <div className="flex justify-between items-center mb-2">
            <div>
              <span className="font-mono text-xs text-zinc-500">{run.id.slice(0, 8)}</span>
              <span className="ml-2 text-sm font-medium">{run.type}</span>
              {run.cruiseLineSlug && (
                <span className="ml-2 text-xs text-zinc-400">{run.cruiseLineSlug}</span>
              )}
            </div>
            <span className={`text-xs px-2 py-1 rounded ${
              run.status === 'completed' ? 'bg-emerald-900 text-emerald-300' :
              run.status === 'failed' ? 'bg-red-900 text-red-300' :
              'bg-blue-900 text-blue-300'
            }`}>
              {run.status}
            </span>
          </div>
          <div className="flex gap-4 text-xs text-zinc-400">
            <span>Expected: {run.totalExpected}</span>
            <span>Completed: {run.totalCompleted}</span>
            <span className="text-emerald-400">Succeeded: {run.totalSucceeded}</span>
            <span className="text-red-400">Failed: {run.totalFailed}</span>
          </div>
          {/* Progress bar */}
          <div className="mt-2 h-1.5 bg-zinc-800 rounded-full overflow-hidden">
            <div
              className="h-full bg-emerald-500 transition-all"
              style={{ width: `${run.totalExpected > 0 ? (run.totalCompleted / run.totalExpected) * 100 : 0}%` }}
            />
          </div>
        </div>
      ))}
    </div>
  )
}

function DispatchTab({ onSubmit }: { onSubmit: () => void }) {
  const [action, setAction] = useState('')
  const [shipSlug, setShipSlug] = useState('')
  const [cruiseLineSlug, setCruiseLineSlug] = useState('')
  const [message, setMessage] = useState('')
  const [submitting, setSubmitting] = useState(false)

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault()
    setSubmitting(true)
    setMessage('')

    const res = await fetch('/api/jobs', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ action, shipSlug, cruiseLineSlug }),
    })

    const data = await res.json()
    setMessage(res.ok ? data.message : data.error)
    setSubmitting(false)
    if (res.ok) onSubmit()
  }

  return (
    <form onSubmit={handleSubmit} className="max-w-md space-y-4">
      <div>
        <label className="block text-sm text-zinc-400 mb-1">Action</label>
        <select
          value={action}
          onChange={e => setAction(e.target.value)}
          className="w-full bg-zinc-900 border border-zinc-700 rounded px-3 py-2 text-sm"
        >
          <option value="">Select action...</option>
          <option value="scrape-sailings">Scrape Single Ship</option>
          <option value="scrape-all">Scrape All Ships (Cruise Line)</option>
        </select>
      </div>

      {action === 'scrape-sailings' && (
        <div>
          <label className="block text-sm text-zinc-400 mb-1">Ship Slug</label>
          <input
            type="text"
            value={shipSlug}
            onChange={e => setShipSlug(e.target.value)}
            placeholder="carnival-celebration"
            className="w-full bg-zinc-900 border border-zinc-700 rounded px-3 py-2 text-sm"
          />
        </div>
      )}

      {action === 'scrape-all' && (
        <div>
          <label className="block text-sm text-zinc-400 mb-1">Cruise Line Slug</label>
          <input
            type="text"
            value={cruiseLineSlug}
            onChange={e => setCruiseLineSlug(e.target.value)}
            placeholder="carnival"
            className="w-full bg-zinc-900 border border-zinc-700 rounded px-3 py-2 text-sm"
          />
        </div>
      )}

      <button
        type="submit"
        disabled={submitting || !action}
        className="px-4 py-2 bg-emerald-600 hover:bg-emerald-500 disabled:opacity-50 rounded text-sm font-medium"
      >
        {submitting ? 'Submitting...' : 'Dispatch Job'}
      </button>

      {message && (
        <p className="text-sm text-zinc-300 bg-zinc-900 rounded p-3">{message}</p>
      )}
    </form>
  )
}
```

## Key Patterns

### Polling for Real-Time Updates

The admin view polls the API every 5 seconds instead of using WebSockets. This is simpler and sufficient for admin monitoring.

```typescript
useEffect(() => {
  fetchData()
  const interval = setInterval(fetchData, 5000)
  return () => clearInterval(interval)
}, [fetchData])
```

### State Color Coding

Every job state has a distinct color for quick visual scanning:

| State | Color | Tailwind Class |
|-------|-------|---------------|
| created | Gray | `text-zinc-400` |
| active | Blue | `text-blue-400` |
| completed | Green | `text-emerald-400` |
| failed | Red | `text-red-400` |
| retry | Amber | `text-amber-400` |
| expired | Orange | `text-orange-400` |
| cancelled | Dark gray | `text-zinc-500` |

### Run Progress Tracking

Application-level runs track aggregate progress across multiple pg-boss jobs:

```
Run (runId: abc123)
├── totalExpected: 15
├── totalCompleted: 12
├── totalSucceeded: 11
├── totalFailed: 1
├── status: 'running'
└── pg-boss jobs:
    ├── sailing-scrape (carnival-celebration) → completed
    ├── sailing-scrape (carnival-jubilee) → completed
    ├── sailing-scrape (carnival-venezia) → failed
    └── sailing-scrape (carnival-firenze) → active
```

### Admin Layout Integration

The jobs page lives under the admin layout which handles the sidebar nav and auth shell:

```
app/
└── admin/
    ├── layout.tsx          (admin shell: sidebar, auth check)
    ├── jobs/
    │   ├── page.tsx        (server component: auth + render)
    │   └── JobsView.tsx    (client component: tabs + polling)
    └── ...
```

### Linking Runs to pg-boss Jobs

Use the `runId` stored in job data to query both application tables and pg-boss tables together:

```typescript
// Get run details with all associated jobs
const [run] = await db.select().from(scrapeRuns)
  .where(eq(scrapeRuns.id, runId)).limit(1)

const pgBossJobs = await db.execute(sql`
  SELECT id::text, name, state, data, created_on, started_on, completed_on, output
  FROM pgboss.job
  WHERE data->>'runId' = ${runId}
  ORDER BY created_on DESC
`)
```

This pattern gives you both the aggregate run view (how many succeeded/failed) and the individual job details (timing, output, errors).
