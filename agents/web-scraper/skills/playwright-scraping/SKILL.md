---
name: playwright-scraping
description: Playwright browser automation for scraping JavaScript-heavy sites and SPAs with bot protection. Use when native fetch fails due to client-side rendering, bot detection, or dynamic content loading.
---

# Playwright Scraping

## When to Use

Escalate to Playwright when:
- Site requires JavaScript execution to render content
- Content loads via client-side API calls you can't replicate directly
- Bot protection (Akamai, Cloudflare) blocks raw HTTP requests
- You need to intercept network requests to capture API responses

## Setup

```javascript
import { chromium } from 'playwright'

const browser = await chromium.launch({
  headless: true,
  args: ['--no-sandbox', '--disable-setuid-sandbox'],
})

const context = await browser.newContext({
  userAgent: 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36',
  viewport: { width: 1280, height: 720 },
})

const page = await context.newPage()
```

## Network Interception (Best Pattern)

Intercept API responses instead of scraping the DOM:

```javascript
const apiData = []

// Listen for API responses before navigating
page.on('response', async (response) => {
  const url = response.url()
  if (url.includes('/api/sailings') && response.status() === 200) {
    const json = await response.json()
    apiData.push(...json.results)
  }
})

await page.goto('https://example.com/cruises', { waitUntil: 'networkidle' })

// Now apiData contains the intercepted API responses
console.log(`Captured ${apiData.length} sailings from API`)
```

## DOM Scraping (Fallback)

When there's no API to intercept:

```javascript
await page.goto('https://example.com/ships', { waitUntil: 'domcontentloaded' })
await page.waitForSelector('.ship-card', { timeout: 10000 })

const ships = await page.$$eval('.ship-card', (cards) =>
  cards.map((card) => ({
    name: card.querySelector('.ship-name')?.textContent?.trim(),
    image: card.querySelector('img')?.src,
    url: card.querySelector('a')?.href,
  }))
)
```

## Handling Bot Protection

```javascript
// Add stealth measures
const context = await browser.newContext({
  userAgent: 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) ...',
  locale: 'en-US',
  timezoneId: 'America/New_York',
})

// Wait for challenge to resolve
await page.goto(url, { waitUntil: 'networkidle', timeout: 30000 })

// Check for challenge page
const isChallenge = await page.$('.challenge-container')
if (isChallenge) {
  console.log('Waiting for bot challenge to resolve...')
  await page.waitForNavigation({ timeout: 15000 })
}
```

## Pagination

```javascript
let allResults = []
let hasNext = true

while (hasNext) {
  const pageResults = await page.$$eval('.result-item', items =>
    items.map(item => ({ /* extract fields */ }))
  )
  allResults.push(...pageResults)

  const nextButton = await page.$('button[aria-label="Next page"]:not([disabled])')
  if (nextButton) {
    await nextButton.click()
    await page.waitForLoadState('networkidle')
  } else {
    hasNext = false
  }
}
```

## Cleanup

```javascript
// Always close browser
try {
  // ... scraping logic
} finally {
  await browser.close()
}
```

## Running Headless in Docker

For production, use Xvfb for virtual display:

```dockerfile
RUN apt-get update && apt-get install -y xvfb
ENV DISPLAY=:99
CMD Xvfb :99 -screen 0 1280x720x24 & node scraper.mjs
```

Or use Playwright's built-in headless mode (preferred):

```javascript
const browser = await chromium.launch({ headless: true })
```
