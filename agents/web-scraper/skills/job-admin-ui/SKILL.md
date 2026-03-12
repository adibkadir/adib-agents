---
name: job-admin-ui
description: "Next.js /admin/jobs page — shows all pg-boss background jobs, their status, errors, output, run tracking with progress bars. Real-time polling, state color coding, filterable by queue. Only shown when scrapers kick off background jobs via pg-boss. Admin-only via better-auth."
---

# Job Admin UI (`/admin/jobs`)

A Next.js admin page for monitoring pg-boss background jobs. Only needed when scrapers are wired into job queues (not for standalone scraper execution — that's `/admin/scrapers`).

## Architecture

```
app/
└── admin/
    └── jobs/
        ├── page.tsx              (server component: auth gate)
        └── JobsView.tsx          (client component: polling + tabs)

app/api/
└── admin/
    └── jobs/
        └── route.ts              (GET: job status, POST: dispatch)
```

## API Route

```typescript
// app/api/admin/jobs/route.ts
import { getBoss } from '@/worker/boss'
import { auth } from '@/lib/auth'
import { db } from '@/db'
import { scrapeRuns } from '@/db/schema'
import { eq, desc, sql } from 'drizzle-orm'

async function requireAdmin(request: Request): Promise<Response | null> {
  const session = await auth.api.getSession({ headers: request.headers })
  if (!session?.user) return Response.json({ error: 'Unauthorized' }, { status: 401 })
  if (session.user.role !== 'admin') return Response.json({ error: 'Forbidden' }, { status: 403 })
  return null
}

export async function GET(request: Request) {
  const denied = await requireAdmin(request)
  if (denied) return denied

  const { searchParams } = new URL(request.url)
  const limit = Number(searchParams.get('limit') ?? 50)
  const queue = searchParams.get('queue')
  const runId = searchParams.get('runId')

  // Specific run detail
  if (runId) {
    const [run] = await db.select().from(scrapeRuns)
      .where(eq(scrapeRuns.id, runId)).limit(1)

    const pgBossJobs = await db.execute(sql`
      SELECT id::text, name, state, data, created_on, started_on, completed_on, output
      FROM pgboss.job
      WHERE data->>'runId' = ${runId}
      ORDER BY created_on DESC
    `)

    return Response.json({ run, jobs: pgBossJobs })
  }

  // List recent jobs
  let jobs: unknown[] = []
  try {
    const whereClause = queue
      ? sql`WHERE name = ${queue}`
      : sql`WHERE name NOT LIKE '__pgboss%'`

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

  // Recent runs
  let runs: unknown[] = []
  try {
    runs = await db.select().from(scrapeRuns)
      .orderBy(desc(scrapeRuns.startedAt))
      .limit(20)
  } catch {
    // scrape_runs table might not exist yet
  }

  return Response.json({ jobs, runs })
}

export async function POST(request: Request) {
  const denied = await requireAdmin(request)
  if (denied) return denied

  const body = await request.json()
  const { action, scraperId, params = {} } = body

  const boss = await getBoss()

  if (action === 'dispatch') {
    const runId = crypto.randomUUID()
    await boss.send('scraper-execute', {
      scraperId,
      params,
      runId,
      totalExpected: 1,
    })
    return Response.json({ message: `Job dispatched for ${scraperId}`, runId })
  }

  if (action === 'dispatch-all') {
    const { scraperIds = [] } = body
    const runId = crypto.randomUUID()

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

    return Response.json({ message: `${scraperIds.length} jobs dispatched`, runId })
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
  const session = await auth.api.getSession({ headers: await headers() })
  if (!session?.user || session.user.role !== 'admin') redirect('/')

  return (
    <div className="p-6 max-w-6xl mx-auto">
      <h1 className="text-2xl font-bold mb-6">Background Jobs</h1>
      <JobsView />
    </div>
  )
}
```

## Jobs View (Client Component)

```tsx
'use client'

import { useState, useEffect, useCallback } from 'react'

const STATE_COLORS: Record<string, string> = {
  created: 'text-zinc-400',
  retry: 'text-amber-400',
  active: 'text-blue-400',
  completed: 'text-emerald-400',
  cancelled: 'text-zinc-500',
  failed: 'text-red-400',
  expired: 'text-orange-400',
}

const STATE_BG: Record<string, string> = {
  created: 'bg-zinc-800',
  retry: 'bg-amber-900/30',
  active: 'bg-blue-900/30',
  completed: 'bg-emerald-900/30',
  cancelled: 'bg-zinc-800',
  failed: 'bg-red-900/30',
  expired: 'bg-orange-900/30',
}

type Tab = 'jobs' | 'runs'

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
  const [expandedJob, setExpandedJob] = useState<string | null>(null)
  const [loading, setLoading] = useState(true)

  // Discover available queue names from job data
  const queueNames = [...new Set(jobs.map(j => j.name))].sort()

  const fetchData = useCallback(async () => {
    const params = new URLSearchParams()
    if (queueFilter !== 'all') params.set('queue', queueFilter)

    const res = await fetch(`/api/admin/jobs?${params}`)
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
        {(['jobs', 'runs'] as Tab[]).map(t => (
          <button
            key={t}
            onClick={() => setTab(t)}
            className={`px-4 py-2 rounded-t text-sm font-medium transition-colors ${
              tab === t ? 'bg-zinc-800 text-white' : 'text-zinc-500 hover:text-zinc-300'
            }`}
          >
            {t === 'jobs' ? 'Job Monitor' : 'Scrape Runs'}
          </button>
        ))}
      </div>

      {tab === 'jobs' && (
        <div>
          {/* Queue filter chips */}
          <div className="flex gap-2 mb-4 flex-wrap">
            <button
              onClick={() => setQueueFilter('all')}
              className={`px-3 py-1 rounded text-xs transition-colors ${
                queueFilter === 'all' ? 'bg-zinc-700 text-white' : 'bg-zinc-900 text-zinc-400 hover:text-zinc-300'
              }`}
            >
              All
            </button>
            {queueNames.map(q => (
              <button
                key={q}
                onClick={() => setQueueFilter(q)}
                className={`px-3 py-1 rounded text-xs transition-colors ${
                  queueFilter === q ? 'bg-zinc-700 text-white' : 'bg-zinc-900 text-zinc-400 hover:text-zinc-300'
                }`}
              >
                {q}
              </button>
            ))}
          </div>

          {loading ? (
            <p className="text-zinc-500 text-sm">Loading jobs...</p>
          ) : jobs.length === 0 ? (
            <p className="text-zinc-500 text-sm py-8 text-center">No jobs found.</p>
          ) : (
            <div className="space-y-1">
              {jobs.map(job => (
                <div key={job.id} className={`rounded border border-zinc-800 ${STATE_BG[job.state] ?? ''}`}>
                  <button
                    onClick={() => setExpandedJob(expandedJob === job.id ? null : job.id)}
                    className="w-full px-4 py-2.5 flex items-center justify-between text-sm hover:bg-zinc-900/30 transition-colors"
                  >
                    <div className="flex items-center gap-3">
                      <span className="font-mono text-xs text-zinc-500 w-16 text-left">{job.name}</span>
                      <span className={`text-xs font-medium ${STATE_COLORS[job.state] ?? 'text-zinc-400'}`}>
                        {job.state}
                      </span>
                      <span className="text-xs text-zinc-600">
                        {new Date(job.created_on).toLocaleString()}
                      </span>
                    </div>
                    <div className="flex items-center gap-3">
                      {job.started_on && job.completed_on && (
                        <span className="text-xs text-zinc-500">
                          {((new Date(job.completed_on).getTime() - new Date(job.started_on).getTime()) / 1000).toFixed(1)}s
                        </span>
                      )}
                      <span className="text-xs text-zinc-600">{expandedJob === job.id ? '▼' : '▶'}</span>
                    </div>
                  </button>

                  {expandedJob === job.id && (
                    <div className="px-4 pb-3 border-t border-zinc-800/50">
                      <div className="mt-2 space-y-2">
                        <div>
                          <span className="text-xs text-zinc-500">Input:</span>
                          <pre className="text-xs text-zinc-400 font-mono bg-zinc-950 rounded p-2 mt-1 overflow-x-auto max-h-32">
                            {JSON.stringify(job.data, null, 2)}
                          </pre>
                        </div>
                        {job.output && (
                          <div>
                            <span className="text-xs text-zinc-500">Output:</span>
                            <pre className="text-xs text-zinc-400 font-mono bg-zinc-950 rounded p-2 mt-1 overflow-x-auto max-h-32">
                              {JSON.stringify(job.output, null, 2)}
                            </pre>
                          </div>
                        )}
                        <div className="flex gap-4 text-xs text-zinc-600">
                          <span>ID: {job.id}</span>
                          {job.started_on && <span>Started: {new Date(job.started_on).toLocaleString()}</span>}
                          {job.completed_on && <span>Completed: {new Date(job.completed_on).toLocaleString()}</span>}
                        </div>
                      </div>
                    </div>
                  )}
                </div>
              ))}
            </div>
          )}
        </div>
      )}

      {tab === 'runs' && (
        <div className="space-y-3">
          {runs.length === 0 ? (
            <p className="text-zinc-500 text-sm py-8 text-center">No scrape runs found.</p>
          ) : (
            runs.map(run => (
              <div key={run.id} className="bg-zinc-900 rounded-lg p-4">
                <div className="flex justify-between items-center mb-2">
                  <div className="flex items-center gap-2">
                    <span className="font-mono text-xs text-zinc-500">{run.id.slice(0, 8)}</span>
                    <span className="text-sm font-medium">{run.type}</span>
                  </div>
                  <span className={`text-xs px-2 py-1 rounded ${
                    run.status === 'completed' ? 'bg-emerald-900/50 text-emerald-300' :
                    run.status === 'failed' ? 'bg-red-900/50 text-red-300' :
                    'bg-blue-900/50 text-blue-300'
                  }`}>
                    {run.status}
                  </span>
                </div>
                <div className="flex gap-4 text-xs text-zinc-400">
                  <span>Expected: {run.totalExpected}</span>
                  <span>Done: {run.totalCompleted}</span>
                  <span className="text-emerald-400">OK: {run.totalSucceeded}</span>
                  <span className="text-red-400">Fail: {run.totalFailed}</span>
                </div>
                <div className="mt-2 h-1.5 bg-zinc-800 rounded-full overflow-hidden">
                  <div
                    className="h-full bg-emerald-500 transition-all duration-500"
                    style={{ width: `${run.totalExpected > 0 ? (run.totalCompleted / run.totalExpected) * 100 : 0}%` }}
                  />
                </div>
                <p className="text-xs text-zinc-600 mt-2">
                  {new Date(run.startedAt).toLocaleString()}
                  {run.completedAt && ` → ${new Date(run.completedAt).toLocaleString()}`}
                </p>
              </div>
            ))
          )}
        </div>
      )}
    </div>
  )
}
```

## State Color Reference

| State | Text Color | Background |
|-------|-----------|------------|
| created | `text-zinc-400` | `bg-zinc-800` |
| active | `text-blue-400` | `bg-blue-900/30` |
| completed | `text-emerald-400` | `bg-emerald-900/30` |
| failed | `text-red-400` | `bg-red-900/30` |
| retry | `text-amber-400` | `bg-amber-900/30` |
| expired | `text-orange-400` | `bg-orange-900/30` |
| cancelled | `text-zinc-500` | `bg-zinc-800` |

## When to Build This Page

Build `/admin/jobs` only when scrapers are configured as pg-boss background jobs. If scrapers only run on-demand via `/admin/scrapers`, this page is not needed.

Decision tree:
- Scrapers run on-demand only → `/admin/scrapers` is sufficient
- Scrapers also run on cron schedules or via job dispatch → build `/admin/jobs` too
- Admin sidebar should link to both pages when both exist
