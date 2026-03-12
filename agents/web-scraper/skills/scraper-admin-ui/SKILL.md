---
name: scraper-admin-ui
description: "Next.js /admin/scrapers page — lists all available scrapers, execute them on-demand, see real-time status and errors, view returned JSON with copy button. Results cached in localStorage so they persist across page refreshes. Admin-only access via better-auth. Add to existing Next.js apps."
---

# Scraper Admin UI (`/admin/scrapers`)

A Next.js admin page that lists all scrapers, lets you run them on-demand, and shows status, errors, and returned JSON data with a copy button. Results persist in localStorage across refreshes.

## Architecture

```
app/
└── admin/
    ├── layout.tsx                    (admin shell: sidebar, auth check)
    └── scrapers/
        ├── page.tsx                  (server component: auth gate)
        ├── ScrapersView.tsx          (client component: UI + localStorage)
        └── scraper-registry.ts       (scraper definitions)

app/api/
└── admin/
    └── scrapers/
        ├── route.ts                  (GET: list scrapers, POST: execute)
        └── [id]/
            └── route.ts              (GET: scraper status by execution ID)
```

## Scraper Registry

Define all scrapers in a single registry. Each scraper is a function that returns JSON.

```typescript
// app/admin/scrapers/scraper-registry.ts

export interface ScraperDefinition {
  id: string
  name: string
  description: string
  /** Optional params the scraper accepts (shown as form fields) */
  params?: {
    name: string
    label: string
    type: 'text' | 'number' | 'select'
    required?: boolean
    options?: string[]  // for select type
    placeholder?: string
    defaultValue?: string
  }[]
  /** The function that does the actual scraping — returns JSON data */
  execute: (params: Record<string, string>) => Promise<{
    success: boolean
    data: unknown
    error?: string
    meta?: { count?: number; duration?: number; url?: string }
  }>
}

// Example registry — import each scraper's execute function
import { scrapeExampleSailings } from '@/scrapers/example-sailings'
import { scrapeExampleShips } from '@/scrapers/example-ships'
import { scrapeReviews } from '@/scrapers/reviews'

export const SCRAPER_REGISTRY: ScraperDefinition[] = [
  {
    id: 'example-sailings',
    name: 'Example Sailings',
    description: 'Scrapes sailing data from Example Cruise Line API',
    params: [
      { name: 'shipCode', label: 'Ship Code', type: 'text', placeholder: 'CE', required: true },
      { name: 'guests', label: 'Guest Count', type: 'number', defaultValue: '2' },
    ],
    execute: scrapeExampleSailings,
  },
  {
    id: 'example-ships',
    name: 'Example Ships',
    description: 'Scrapes fleet/ship list from Example Cruise Line',
    execute: scrapeExampleShips,
  },
  {
    id: 'reviews',
    name: 'Reviews',
    description: 'Scrapes reviews from multiple sources',
    params: [
      {
        name: 'source',
        label: 'Source',
        type: 'select',
        options: ['google', 'yelp', 'dealerrater'],
        required: true,
      },
      { name: 'url', label: 'Page URL', type: 'text', required: true },
    ],
    execute: scrapeReviews,
  },
]
```

## Scraper Execute Function Pattern

Each scraper's execute function wraps the standalone scraper logic:

```typescript
// scrapers/example-sailings.ts
import { z } from 'zod'

const SailingSchema = z.object({
  ship: z.string(),
  embark_date: z.string(),
  debark_date: z.string(),
  price: z.number().nullable(),
})

export async function scrapeExampleSailings(
  params: Record<string, string>
): Promise<{ success: boolean; data: unknown; error?: string; meta?: Record<string, unknown> }> {
  const startTime = Date.now()

  try {
    const { shipCode, guests = '2' } = params

    const response = await fetch(`https://api.example.com/sailings?ship=${shipCode}&guests=${guests}`, {
      headers: {
        'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36',
        'Accept': 'application/json',
      },
    })

    if (!response.ok) {
      return {
        success: false,
        data: null,
        error: `HTTP ${response.status}: ${await response.text().then(t => t.slice(0, 500))}`,
      }
    }

    const raw = await response.json()
    const sailings = raw.results.map((item: any) => ({
      ship: item.shipName,
      embark_date: item.departureDate?.split('T')[0],
      debark_date: item.returnDate?.split('T')[0],
      price: item.lowestPrice ?? null,
    }))

    const validated = sailings.filter((s: unknown) => SailingSchema.safeParse(s).success)

    return {
      success: true,
      data: validated,
      meta: {
        count: validated.length,
        totalRaw: sailings.length,
        duration: Date.now() - startTime,
        url: response.url,
      },
    }
  } catch (error) {
    return {
      success: false,
      data: null,
      error: error instanceof Error ? error.message : String(error),
      meta: { duration: Date.now() - startTime },
    }
  }
}
```

## API Route

```typescript
// app/api/admin/scrapers/route.ts
import { auth } from '@/lib/auth'
import { SCRAPER_REGISTRY } from '@/app/admin/scrapers/scraper-registry'

async function requireAdmin(request: Request): Promise<Response | null> {
  const session = await auth.api.getSession({ headers: request.headers })
  if (!session?.user) return Response.json({ error: 'Unauthorized' }, { status: 401 })
  if (session.user.role !== 'admin') return Response.json({ error: 'Forbidden' }, { status: 403 })
  return null
}

// GET: List all available scrapers (without execute functions)
export async function GET(request: Request) {
  const denied = await requireAdmin(request)
  if (denied) return denied

  const scrapers = SCRAPER_REGISTRY.map(({ execute, ...rest }) => rest)
  return Response.json({ scrapers })
}

// POST: Execute a scraper
export async function POST(request: Request) {
  const denied = await requireAdmin(request)
  if (denied) return denied

  const body = await request.json()
  const { scraperId, params = {} } = body

  const scraper = SCRAPER_REGISTRY.find(s => s.id === scraperId)
  if (!scraper) {
    return Response.json({ error: `Scraper '${scraperId}' not found` }, { status: 404 })
  }

  // Validate required params
  for (const param of scraper.params ?? []) {
    if (param.required && !params[param.name]) {
      return Response.json(
        { error: `Missing required parameter: ${param.label}` },
        { status: 400 }
      )
    }
  }

  // Execute the scraper
  const result = await scraper.execute(params)

  return Response.json({
    scraperId: scraper.id,
    scraperName: scraper.name,
    executedAt: new Date().toISOString(),
    ...result,
  })
}
```

## Admin Page (Server Component)

```typescript
// app/admin/scrapers/page.tsx
import { redirect } from 'next/navigation'
import { auth } from '@/lib/auth'
import { headers } from 'next/headers'
import { ScrapersView } from './ScrapersView'

export default async function AdminScrapersPage() {
  const session = await auth.api.getSession({
    headers: await headers(),
  })

  if (!session?.user || session.user.role !== 'admin') {
    redirect('/')
  }

  return (
    <div className="p-6 max-w-6xl mx-auto">
      <h1 className="text-2xl font-bold mb-6">Scrapers</h1>
      <ScrapersView />
    </div>
  )
}
```

## Scrapers View (Client Component with localStorage)

```tsx
'use client'

import { useState, useEffect, useCallback } from 'react'

// --- Types ---

interface ScraperParam {
  name: string
  label: string
  type: 'text' | 'number' | 'select'
  required?: boolean
  options?: string[]
  placeholder?: string
  defaultValue?: string
}

interface ScraperInfo {
  id: string
  name: string
  description: string
  params?: ScraperParam[]
}

interface ScraperExecution {
  scraperId: string
  scraperName: string
  executedAt: string
  success: boolean
  data: unknown
  error?: string
  meta?: { count?: number; duration?: number; url?: string }
}

// --- localStorage helpers ---

const STORAGE_KEY = 'admin-scraper-results'

function loadCachedResults(): Record<string, ScraperExecution> {
  try {
    const stored = localStorage.getItem(STORAGE_KEY)
    return stored ? JSON.parse(stored) : {}
  } catch {
    return {}
  }
}

function saveCachedResults(results: Record<string, ScraperExecution>) {
  try {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(results))
  } catch {
    // localStorage full or unavailable
  }
}

// --- Main Component ---

export function ScrapersView() {
  const [scrapers, setScrapers] = useState<ScraperInfo[]>([])
  const [results, setResults] = useState<Record<string, ScraperExecution>>({})
  const [running, setRunning] = useState<Set<string>>(new Set())
  const [paramValues, setParamValues] = useState<Record<string, Record<string, string>>>({})
  const [expandedResult, setExpandedResult] = useState<string | null>(null)
  const [copied, setCopied] = useState<string | null>(null)
  const [loading, setLoading] = useState(true)

  // Load scrapers list + cached results on mount
  useEffect(() => {
    const cached = loadCachedResults()
    setResults(cached)

    fetch('/api/admin/scrapers')
      .then(res => res.json())
      .then(data => {
        setScrapers(data.scrapers ?? [])
        // Init default param values
        const defaults: Record<string, Record<string, string>> = {}
        for (const s of data.scrapers ?? []) {
          defaults[s.id] = {}
          for (const p of s.params ?? []) {
            if (p.defaultValue) defaults[s.id][p.name] = p.defaultValue
          }
        }
        setParamValues(defaults)
      })
      .finally(() => setLoading(false))
  }, [])

  // Execute a scraper
  const executeScraper = useCallback(async (scraperId: string) => {
    setRunning(prev => new Set(prev).add(scraperId))

    try {
      const res = await fetch('/api/admin/scrapers', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          scraperId,
          params: paramValues[scraperId] ?? {},
        }),
      })

      const result: ScraperExecution = await res.json()

      setResults(prev => {
        const updated = { ...prev, [scraperId]: result }
        saveCachedResults(updated)
        return updated
      })

      // Auto-expand result
      setExpandedResult(scraperId)
    } catch (error) {
      const errorResult: ScraperExecution = {
        scraperId,
        scraperName: scrapers.find(s => s.id === scraperId)?.name ?? scraperId,
        executedAt: new Date().toISOString(),
        success: false,
        data: null,
        error: error instanceof Error ? error.message : 'Network error',
      }
      setResults(prev => {
        const updated = { ...prev, [scraperId]: errorResult }
        saveCachedResults(updated)
        return updated
      })
      setExpandedResult(scraperId)
    } finally {
      setRunning(prev => {
        const next = new Set(prev)
        next.delete(scraperId)
        return next
      })
    }
  }, [paramValues, scrapers])

  // Copy JSON to clipboard
  const copyJson = useCallback((scraperId: string) => {
    const result = results[scraperId]
    if (!result) return
    navigator.clipboard.writeText(JSON.stringify(result.data, null, 2))
    setCopied(scraperId)
    setTimeout(() => setCopied(null), 2000)
  }, [results])

  // Clear a single result
  const clearResult = useCallback((scraperId: string) => {
    setResults(prev => {
      const updated = { ...prev }
      delete updated[scraperId]
      saveCachedResults(updated)
      return updated
    })
    if (expandedResult === scraperId) setExpandedResult(null)
  }, [expandedResult])

  // Clear all cached results
  const clearAllResults = useCallback(() => {
    setResults({})
    saveCachedResults({})
    setExpandedResult(null)
  }, [])

  if (loading) return <p className="text-zinc-500">Loading scrapers...</p>

  return (
    <div>
      {/* Header with clear all button */}
      {Object.keys(results).length > 0 && (
        <div className="flex justify-end mb-4">
          <button
            onClick={clearAllResults}
            className="text-xs text-zinc-500 hover:text-red-400 transition-colors"
          >
            Clear all cached results
          </button>
        </div>
      )}

      {/* Scraper cards */}
      <div className="space-y-4">
        {scrapers.map(scraper => {
          const result = results[scraper.id]
          const isRunning = running.has(scraper.id)
          const isExpanded = expandedResult === scraper.id

          return (
            <div key={scraper.id} className="border border-zinc-800 rounded-lg overflow-hidden">
              {/* Scraper header */}
              <div className="p-4 bg-zinc-900/50">
                <div className="flex items-center justify-between">
                  <div>
                    <h3 className="font-medium text-sm">{scraper.name}</h3>
                    <p className="text-xs text-zinc-500 mt-0.5">{scraper.description}</p>
                  </div>
                  <div className="flex items-center gap-2">
                    {/* Status badge */}
                    {result && (
                      <span className={`text-xs px-2 py-0.5 rounded-full ${
                        result.success
                          ? 'bg-emerald-900/50 text-emerald-400'
                          : 'bg-red-900/50 text-red-400'
                      }`}>
                        {result.success ? 'Success' : 'Failed'}
                      </span>
                    )}
                    {/* Run button */}
                    <button
                      onClick={() => executeScraper(scraper.id)}
                      disabled={isRunning}
                      className="px-3 py-1.5 bg-emerald-600 hover:bg-emerald-500 disabled:opacity-50 disabled:cursor-not-allowed rounded text-xs font-medium transition-colors"
                    >
                      {isRunning ? (
                        <span className="flex items-center gap-1.5">
                          <span className="w-3 h-3 border-2 border-white/30 border-t-white rounded-full animate-spin" />
                          Running...
                        </span>
                      ) : 'Run'}
                    </button>
                  </div>
                </div>

                {/* Parameter inputs */}
                {scraper.params && scraper.params.length > 0 && (
                  <div className="flex flex-wrap gap-3 mt-3">
                    {scraper.params.map(param => (
                      <div key={param.name} className="flex-1 min-w-[150px]">
                        <label className="block text-xs text-zinc-500 mb-1">
                          {param.label}
                          {param.required && <span className="text-red-400 ml-0.5">*</span>}
                        </label>
                        {param.type === 'select' ? (
                          <select
                            value={paramValues[scraper.id]?.[param.name] ?? ''}
                            onChange={e => setParamValues(prev => ({
                              ...prev,
                              [scraper.id]: { ...prev[scraper.id], [param.name]: e.target.value },
                            }))}
                            className="w-full bg-zinc-900 border border-zinc-700 rounded px-2 py-1.5 text-xs"
                          >
                            <option value="">Select...</option>
                            {param.options?.map(opt => (
                              <option key={opt} value={opt}>{opt}</option>
                            ))}
                          </select>
                        ) : (
                          <input
                            type={param.type}
                            value={paramValues[scraper.id]?.[param.name] ?? ''}
                            onChange={e => setParamValues(prev => ({
                              ...prev,
                              [scraper.id]: { ...prev[scraper.id], [param.name]: e.target.value },
                            }))}
                            placeholder={param.placeholder}
                            className="w-full bg-zinc-900 border border-zinc-700 rounded px-2 py-1.5 text-xs"
                          />
                        )}
                      </div>
                    ))}
                  </div>
                )}
              </div>

              {/* Result section */}
              {result && (
                <div className="border-t border-zinc-800">
                  {/* Result summary bar */}
                  <button
                    onClick={() => setExpandedResult(isExpanded ? null : scraper.id)}
                    className="w-full px-4 py-2 flex items-center justify-between text-xs text-zinc-400 hover:bg-zinc-900/30 transition-colors"
                  >
                    <div className="flex items-center gap-3">
                      <span>{new Date(result.executedAt).toLocaleString()}</span>
                      {result.meta?.count !== undefined && (
                        <span className="text-zinc-500">{result.meta.count} records</span>
                      )}
                      {result.meta?.duration !== undefined && (
                        <span className="text-zinc-500">{(result.meta.duration / 1000).toFixed(1)}s</span>
                      )}
                    </div>
                    <span>{isExpanded ? '▼' : '▶'}</span>
                  </button>

                  {/* Expanded result */}
                  {isExpanded && (
                    <div className="px-4 pb-4">
                      {/* Error message */}
                      {result.error && (
                        <div className="bg-red-900/20 border border-red-900/50 rounded p-3 mb-3">
                          <p className="text-xs text-red-400 font-mono whitespace-pre-wrap">
                            {result.error}
                          </p>
                        </div>
                      )}

                      {/* JSON data */}
                      {result.data && (
                        <div className="relative">
                          <div className="flex items-center justify-between mb-2">
                            <span className="text-xs text-zinc-500">Response Data</span>
                            <div className="flex gap-2">
                              <button
                                onClick={() => copyJson(scraper.id)}
                                className="px-2 py-1 bg-zinc-800 hover:bg-zinc-700 rounded text-xs transition-colors"
                              >
                                {copied === scraper.id ? '✓ Copied' : 'Copy JSON'}
                              </button>
                              <button
                                onClick={() => clearResult(scraper.id)}
                                className="px-2 py-1 bg-zinc-800 hover:bg-red-900/50 rounded text-xs text-zinc-400 hover:text-red-400 transition-colors"
                              >
                                Clear
                              </button>
                            </div>
                          </div>
                          <textarea
                            readOnly
                            value={JSON.stringify(result.data, null, 2)}
                            className="w-full h-64 bg-zinc-950 border border-zinc-800 rounded p-3 font-mono text-xs text-zinc-300 resize-y"
                          />
                        </div>
                      )}

                      {/* Meta info */}
                      {result.meta?.url && (
                        <p className="text-xs text-zinc-600 mt-2 font-mono truncate">
                          Source: {result.meta.url}
                        </p>
                      )}
                    </div>
                  )}
                </div>
              )}
            </div>
          )
        })}
      </div>

      {scrapers.length === 0 && (
        <div className="text-center py-12 text-zinc-500">
          <p className="text-sm">No scrapers registered.</p>
          <p className="text-xs mt-1">Add scrapers to the registry in scraper-registry.ts</p>
        </div>
      )}
    </div>
  )
}
```

## Key Patterns

### localStorage Caching

Results are cached by scraper ID. When the page refreshes, the last result for each scraper is still visible:

```typescript
// Save after every execution
saveCachedResults(updated)

// Load on component mount
const cached = loadCachedResults()
setResults(cached)
```

The cache stores the full `ScraperExecution` object per scraper ID — including `data`, `error`, `meta`, and `executedAt`. Only the most recent execution per scraper is kept.

### Copy Button

One-click copy of the JSON data to clipboard:

```typescript
navigator.clipboard.writeText(JSON.stringify(result.data, null, 2))
```

Shows "✓ Copied" feedback for 2 seconds after clicking.

### Parameter Forms

Scrapers define their input parameters in the registry. The UI auto-generates form fields:

- `text` → text input
- `number` → number input
- `select` → dropdown with options
- `required` → red asterisk, validated before submission

### Status Indicators

| State | Visual |
|-------|--------|
| Never run | No badge |
| Running | Spinning indicator + disabled button |
| Success | Green "Success" badge |
| Failed | Red "Failed" badge + error message |

### Scraper Function Contract

Every scraper execute function must return:

```typescript
{
  success: boolean        // true if data was extracted
  data: unknown           // the JSON payload (array or object)
  error?: string          // error message if success is false
  meta?: {
    count?: number        // number of records extracted
    duration?: number     // execution time in ms
    url?: string          // API URL that was hit
  }
}
```

This contract is enforced by the `ScraperDefinition` type in the registry.

## Adding a New Scraper

1. Write the scraper logic as a function matching the execute signature
2. Add it to `SCRAPER_REGISTRY` in `scraper-registry.ts`
3. The UI automatically picks it up — no frontend changes needed

```typescript
// scrapers/new-source.ts
export async function scrapeNewSource(params: Record<string, string>) {
  const startTime = Date.now()
  try {
    // ... scraping logic ...
    return { success: true, data: results, meta: { count: results.length, duration: Date.now() - startTime } }
  } catch (error) {
    return { success: false, data: null, error: String(error), meta: { duration: Date.now() - startTime } }
  }
}

// scraper-registry.ts — just add to the array:
{
  id: 'new-source',
  name: 'New Source',
  description: 'Scrapes data from New Source',
  execute: scrapeNewSource,
}
```
