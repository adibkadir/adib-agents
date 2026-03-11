---
name: footage-capture
description: Browser automation for capturing pixel-perfect screenshots, interaction sequences, and UI state recordings from live URLs or local dev servers using Playwright. Includes Firecrawl for deep content extraction to inform creative direction.
---

# Footage Capture

## When to Use

Use this skill whenever you need to:
- Capture screenshots of a website or web app for video production
- Record multi-step user flows (signup, checkout, dashboard navigation)
- Extract marketing copy and feature descriptions via Firecrawl for script writing
- Spin up a local dev server from a repo and capture footage from it

## Capture Strategy Selection

| Scenario | Approach | Tool |
|----------|----------|------|
| Public marketing site | Direct URL capture | Playwright |
| Authenticated app (credentials provided) | Login + navigate + capture | Playwright |
| Bot-protected site | Proxy-backed scraping | Firecrawl (content) + Playwright (screenshots) |
| Local app repo provided | Dev server + capture | pnpm dev + Playwright |
| Need marketing copy / feature text | Content extraction | Firecrawl |

## Content Research with Firecrawl

Before capturing any screenshots, use Firecrawl to deeply read the target site. This informs the creative brief and voiceover script.

```typescript
import Firecrawl from '@mendable/firecrawl-js'

const firecrawl = new Firecrawl({ apiKey: process.env.FIRECRAWL_API_KEY })

// Deep-read the marketing site for copy, features, and structure
const result = await firecrawl.scrapeUrl(targetUrl, {
  formats: ['markdown'],
})

// Crawl multiple pages to understand the full product
const crawlResult = await firecrawl.crawlUrl(targetUrl, {
  limit: 20,
  scrapeOptions: { formats: ['markdown'] },
})

// Extract structured feature data for the script
const extracted = await firecrawl.scrapeUrl(targetUrl, {
  formats: ['extract'],
  extract: {
    schema: {
      type: 'object',
      properties: {
        productName: { type: 'string' },
        tagline: { type: 'string' },
        features: {
          type: 'array',
          items: {
            type: 'object',
            properties: {
              name: { type: 'string' },
              description: { type: 'string' },
              benefit: { type: 'string' },
            },
          },
        },
        pricing: { type: 'string' },
        callToAction: { type: 'string' },
      },
    },
  },
})
```

## Screenshot Capture Setup

```typescript
import { chromium, type Page, type Browser } from 'playwright'
import { mkdir, writeFile } from 'fs/promises'
import path from 'path'

interface CaptureConfig {
  targetUrl: string
  outputDir: string
  viewport: { width: number; height: number }
  credentials?: { email: string; password: string }
  shotList: ShotDefinition[]
}

interface ShotDefinition {
  id: string           // e.g., "01-landing", "02-dashboard"
  url: string          // Page URL or path
  description: string  // What this shot shows
  waitFor?: string     // CSS selector to wait for
  actions?: Action[]   // Interactions before capture
  fullPage?: boolean   // Capture full scrollable page
  clip?: { x: number; y: number; width: number; height: number }
}

type Action =
  | { type: 'click'; selector: string }
  | { type: 'hover'; selector: string }
  | { type: 'type'; selector: string; text: string }
  | { type: 'scroll'; y: number }
  | { type: 'wait'; ms: number }

async function captureFootage(config: CaptureConfig): Promise<string[]> {
  const browser = await chromium.launch({ headless: true })
  const context = await browser.newContext({
    viewport: config.viewport,
    deviceScaleFactor: 2, // Retina for crisp screenshots
    userAgent: 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36',
  })

  const page = await context.newPage()
  await mkdir(config.outputDir, { recursive: true })

  // Login if credentials provided
  if (config.credentials) {
    await login(page, config.targetUrl, config.credentials)
  }

  const capturedPaths: string[] = []

  for (const shot of config.shotList) {
    const filePath = await captureShot(page, shot, config)
    capturedPaths.push(filePath)
    console.log(`Captured: ${shot.id} — ${shot.description}`)
  }

  await browser.close()
  return capturedPaths
}
```

## Authentication Flow

```typescript
async function login(
  page: Page,
  baseUrl: string,
  credentials: { email: string; password: string }
) {
  await page.goto(`${baseUrl}/login`, { waitUntil: 'networkidle' })

  // Common login form patterns
  const emailInput = await page.$('input[type="email"], input[name="email"], #email')
  const passwordInput = await page.$('input[type="password"], input[name="password"], #password')

  if (emailInput && passwordInput) {
    await emailInput.fill(credentials.email)
    await passwordInput.fill(credentials.password)

    const submitButton = await page.$(
      'button[type="submit"], input[type="submit"], button:has-text("Sign in"), button:has-text("Log in")'
    )
    if (submitButton) await submitButton.click()

    await page.waitForNavigation({ waitUntil: 'networkidle', timeout: 15000 })
    console.log('Login successful')
  }
}
```

## Shot Capture with Actions

```typescript
async function captureShot(
  page: Page,
  shot: ShotDefinition,
  config: CaptureConfig
): Promise<string> {
  // Navigate to the shot's URL
  const fullUrl = shot.url.startsWith('http')
    ? shot.url
    : `${config.targetUrl}${shot.url}`

  await page.goto(fullUrl, { waitUntil: 'networkidle' })

  // Wait for specific element if needed
  if (shot.waitFor) {
    await page.waitForSelector(shot.waitFor, { timeout: 10000 })
  }

  // Execute pre-capture actions (hover effects, scrolls, clicks)
  if (shot.actions) {
    for (const action of shot.actions) {
      switch (action.type) {
        case 'click':
          await page.click(action.selector)
          break
        case 'hover':
          await page.hover(action.selector)
          await page.waitForTimeout(300) // Let hover animation settle
          break
        case 'type':
          await page.fill(action.selector, action.text)
          break
        case 'scroll':
          await page.evaluate((y) => window.scrollTo({ top: y, behavior: 'smooth' }), action.y)
          await page.waitForTimeout(500) // Let scroll animation settle
          break
        case 'wait':
          await page.waitForTimeout(action.ms)
          break
      }
    }
  }

  // Capture the screenshot
  const filePath = path.join(config.outputDir, `${shot.id}.png`)
  await page.screenshot({
    path: filePath,
    fullPage: shot.fullPage ?? false,
    clip: shot.clip,
    type: 'png',
  })

  return filePath
}
```

## Multi-Viewport Capture

Capture the same flow at multiple viewport sizes for platform flexibility:

```typescript
const VIEWPORTS = {
  youtube: { width: 1920, height: 1080 },     // 16:9
  tiktok: { width: 1080, height: 1920 },      // 9:16 (mobile mockup)
  desktop: { width: 1440, height: 900 },      // Desktop in browser frame
  mobile: { width: 390, height: 844 },        // iPhone 14 Pro
} as const

async function captureAllViewports(
  shotList: ShotDefinition[],
  config: Omit<CaptureConfig, 'viewport'>
) {
  for (const [name, viewport] of Object.entries(VIEWPORTS)) {
    await captureFootage({
      ...config,
      viewport,
      outputDir: path.join(config.outputDir, name),
      shotList,
    })
  }
}
```

## Dev Server Mode

When the user provides an app repo path, spin up the dev server first:

```typescript
import { exec, spawn, type ChildProcess } from 'child_process'
import { promisify } from 'util'

const execAsync = promisify(exec)

async function startDevServer(repoPath: string): Promise<{
  process: ChildProcess
  url: string
  cleanup: () => void
}> {
  // Install dependencies
  await execAsync('pnpm install', { cwd: repoPath })

  // Detect the dev command and port from package.json
  const pkg = JSON.parse(
    await readFile(path.join(repoPath, 'package.json'), 'utf-8')
  )
  const devScript = pkg.scripts?.dev ?? 'next dev'
  const port = 3000 // Default, can parse from script

  // Start the dev server
  const devProcess = spawn('pnpm', ['dev'], {
    cwd: repoPath,
    stdio: 'pipe',
    env: { ...process.env, PORT: String(port) },
  })

  // Wait for server to be ready
  await waitForServer(`http://localhost:${port}`, 30000)

  return {
    process: devProcess,
    url: `http://localhost:${port}`,
    cleanup: () => devProcess.kill('SIGTERM'),
  }
}

async function waitForServer(url: string, timeoutMs: number) {
  const start = Date.now()
  while (Date.now() - start < timeoutMs) {
    try {
      await fetch(url)
      return
    } catch {
      await new Promise((r) => setTimeout(r, 500))
    }
  }
  throw new Error(`Server at ${url} did not start within ${timeoutMs}ms`)
}
```

## Scroll Recording (Full Page Walkthrough)

For smooth scroll-through shots, capture a sequence of frames:

```typescript
async function captureScrollSequence(
  page: Page,
  outputDir: string,
  options: { frameInterval: number; scrollStep: number }
) {
  const totalHeight = await page.evaluate(() => document.body.scrollHeight)
  const viewportHeight = page.viewportSize()!.height
  let currentScroll = 0
  let frameIndex = 0

  while (currentScroll < totalHeight - viewportHeight) {
    await page.screenshot({
      path: path.join(outputDir, `scroll-${String(frameIndex).padStart(4, '0')}.png`),
    })

    currentScroll += options.scrollStep
    await page.evaluate((y) => window.scrollTo({ top: y }), currentScroll)
    await page.waitForTimeout(options.frameInterval)
    frameIndex++
  }

  console.log(`Captured ${frameIndex} scroll frames`)
}
```

## Capture Checklist

Before starting capture, verify:

- [ ] Target URL is accessible (or dev server is running)
- [ ] Credentials work (test login flow manually first)
- [ ] Output directory exists and is writable
- [ ] Shot list covers all scenes in the voiceover script
- [ ] Each shot has a unique ID matching the script's `[SHOT: id]` markers
- [ ] Viewport matches the target platform (1920x1080 for YouTube, 1080x1920 for TikTok)
- [ ] Device scale factor is 2 for retina-quality captures
- [ ] Network is stable (no flaky loading states)
