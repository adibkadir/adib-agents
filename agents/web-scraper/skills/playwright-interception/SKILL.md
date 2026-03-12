---
name: playwright-interception
description: "Tier 2 — Playwright browser automation focused on response interception, not DOM scraping. Use page.on('response') to capture API calls the SPA makes, then extract the JSON. Browser context inherits auth cookies automatically. Includes anti-detection techniques for Akamai, Cloudflare, and similar bot protection."
---

# Playwright Response Interception (Tier 2)

Use Playwright to let the browser make API calls naturally, then intercept the responses. The browser handles auth, cookies, and bot challenges — you just capture the JSON that comes back.

## When to Escalate to Tier 2

- Tier 1 `fetch` gets 403/captcha/empty response
- Site requires JavaScript execution to generate API tokens
- API requires cookies from a complex auth flow you can't reproduce
- Site uses client-side rendering with no static HTML

## Response Interception (Primary Pattern)

Let the browser navigate naturally and capture API responses:

```javascript
import { chromium } from 'playwright'

const browser = await chromium.launch({ headless: true })
const context = await browser.newContext({
  userAgent: 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36',
  viewport: { width: 1280, height: 720 },
})
const page = await context.newPage()

// --- Capture API responses BEFORE navigating ---
const capturedData = []

page.on('response', async (response) => {
  const url = response.url()
  if (url.includes('/api/sailings') && response.status() === 200) {
    try {
      const json = await response.json()
      capturedData.push(...json.results)
    } catch {
      // Not JSON, ignore
    }
  }
})

// Navigate — browser handles cookies, tokens, challenges
await page.goto('https://example.com/cruises', {
  waitUntil: 'networkidle',
  timeout: 30000,
})

console.log(`Captured ${capturedData.length} results from API interception`)
await browser.close()
```

## In-Browser Fetch (Reuse Auth Session)

Make API calls from inside the browser's authenticated context:

```javascript
// Browser has already loaded the page and has all cookies/tokens
const apiData = await page.evaluate(async () => {
  const response = await fetch('/api/v2/data', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ page: 1, pageSize: 100 }),
  })
  return response.json()
})

// apiData now has the response — browser's cookies were sent automatically
```

This is especially useful when:
- The API requires session cookies from the page load
- There's a CSRF token you can't easily extract
- The auth flow is complex (OAuth redirect chains, etc.)

## Infinite Scroll Interception

For SPAs that load data as you scroll (capture all API pages):

```javascript
const allData = []
let previousCount = 0

page.on('response', async (response) => {
  if (response.url().includes('/api/products') && response.status() === 200) {
    const json = await response.json()
    allData.push(...json.products)
  }
})

await page.goto(targetUrl, { waitUntil: 'domcontentloaded', timeout: 30000 })
await page.waitForSelector('.product-card', { timeout: 15000 })

// Scroll until no new data arrives
let attempts = 0
while (attempts < 20) {
  await page.evaluate(() => window.scrollTo(0, document.body.scrollHeight))
  await page.waitForTimeout(2000)

  if (allData.length > previousCount) {
    previousCount = allData.length
    attempts = 0 // reset — still getting data
  } else {
    attempts++
  }
}

console.log(`Captured ${allData.length} products from infinite scroll`)
```

## Anti-Detection Techniques

### Navigator Spoofing (for Akamai, etc.)

```javascript
const browser = await chromium.launch({
  headless: false, // Akamai detects headless mode — use Xvfb on server
  args: [
    '--disable-blink-features=AutomationControlled',
    '--no-sandbox',
    '--disable-setuid-sandbox',
  ],
})

const context = await browser.newContext({
  userAgent: 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36',
  locale: 'en-US',
  timezoneId: 'America/New_York',
  viewport: { width: 1280, height: 720 },
})

const page = await context.newPage()

// Hide automation signals
await page.addInitScript(() => {
  Object.defineProperty(navigator, 'webdriver', { get: () => false })
  Object.defineProperty(navigator, 'plugins', { get: () => [1, 2, 3, 4, 5] })
  Object.defineProperty(navigator, 'languages', { get: () => ['en-US', 'en'] })
  window.chrome = { runtime: {} }
})
```

### Bot Challenge Detection

```javascript
await page.goto(url, { waitUntil: 'networkidle', timeout: 30000 })

// Check for challenge pages
const challengeSelectors = [
  '.challenge-container',           // Cloudflare
  '#challenge-running',             // Cloudflare
  'iframe[src*="captcha"]',         // Generic CAPTCHA
  '#px-captcha',                    // PerimeterX
  '#distil_ident_block',            // Distil Networks
]

for (const selector of challengeSelectors) {
  const isChallenge = await page.$(selector)
  if (isChallenge) {
    console.log(`Bot challenge detected (${selector}). Waiting for auto-resolve...`)
    await page.waitForNavigation({ timeout: 15000 }).catch(() => {
      throw new Error('Bot challenge could not be resolved — escalate to Tier 3 or 4')
    })
  }
}
```

## DOM Scraping (Fallback Only)

When there's genuinely no API to intercept:

```javascript
await page.goto(url, { waitUntil: 'domcontentloaded' })
await page.waitForSelector('.product-card', { timeout: 10000 })

const products = await page.$$eval('.product-card', (cards) =>
  cards.map((card) => ({
    name: card.querySelector('.title')?.textContent?.trim() ?? '',
    price: parseFloat(card.querySelector('.price')?.textContent?.replace(/[$,]/g, '') ?? '0'),
    url: card.querySelector('a')?.href ?? '',
    image: card.querySelector('img')?.src ?? '',
  }))
)
```

## Click-Through Pagination

```javascript
const allResults = []

while (true) {
  const pageResults = await page.$$eval('.result-item', items =>
    items.map(item => ({
      title: item.querySelector('.title')?.textContent?.trim(),
      url: item.querySelector('a')?.href,
    }))
  )
  allResults.push(...pageResults)

  const nextButton = await page.$('button[aria-label="Next page"]:not([disabled])')
  if (!nextButton) break

  await nextButton.click()
  await page.waitForLoadState('networkidle')
  await page.waitForTimeout(500) // rate limit
}
```

## Running in Docker (Production)

For `headless: false` in Docker, use Xvfb:

```dockerfile
FROM node:20-slim

RUN apt-get update && apt-get install -y \
  xvfb \
  chromium \
  && rm -rf /var/lib/apt/lists/*

ENV DISPLAY=:99

# Start Xvfb, then run scraper
CMD Xvfb :99 -screen 0 1280x720x24 & node scraper.mjs
```

For `headless: true` (simpler, but some sites detect it):

```javascript
const browser = await chromium.launch({ headless: true })
```

## Cleanup

Always close the browser:

```javascript
try {
  // ... scraping logic
} finally {
  await browser.close()
}
```
