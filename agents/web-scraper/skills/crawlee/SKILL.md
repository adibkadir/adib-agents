---
name: crawlee
description: "Tier 3 — Crawlee open-source framework for managed crawling with anti-blocking, proxy rotation, session management, fingerprint randomization, and adaptive HTTP/browser switching. Use when self-managed Playwright gets blocked by sophisticated bot detection. Free alternative to Firecrawl."
---

# Crawlee Framework (Tier 3)

Crawlee is a free, open-source web scraping framework that handles anti-blocking, proxy rotation, and session management. Use it when raw Playwright (Tier 2) gets blocked by sophisticated bot detection.

**Install:** `pnpm add crawlee playwright`

**Docs:** https://crawlee.dev

## When to Escalate to Tier 3

- Tier 2 Playwright gets IP-banned or fingerprinted
- You need proxy rotation across many requests
- You need persistent session/cookie management
- You're crawling hundreds of pages with retries and queuing
- You need adaptive HTTP vs browser switching

## Crawler Types

| Crawler | When to Use | Speed | Resource |
|---------|------------|-------|----------|
| `HttpCrawler` | APIs and raw HTTP responses | Fastest | Minimal |
| `CheerioCrawler` | Static HTML parsing | Fast | Low |
| `PlaywrightCrawler` | JavaScript-rendered pages | Slow | High |
| `AdaptivePlaywrightCrawler` | Auto-select HTTP vs browser | Adaptive | Variable |

## AdaptivePlaywrightCrawler (Recommended)

Automatically determines if a page needs a browser or can be fetched with HTTP. Starts with HTTP, falls back to browser when needed:

```javascript
import { AdaptivePlaywrightCrawler, Dataset } from 'crawlee'

const crawler = new AdaptivePlaywrightCrawler({
  maxRequestsPerCrawl: 500,
  maxConcurrency: 5,
  requestHandlerTimeoutSecs: 60,

  async requestHandler({ request, page, body, $ }) {
    // $ is available for Cheerio-like parsing (HTTP mode)
    // page is available for Playwright interaction (browser mode)

    if ($) {
      // HTTP mode — parse with Cheerio
      const title = $('h1').text()
      const items = $('li.product').map((_, el) => ({
        name: $(el).find('.name').text(),
        price: $(el).find('.price').text(),
      })).get()

      await Dataset.pushData({ title, items, url: request.url })
    } else if (page) {
      // Browser mode — Playwright interaction
      await page.waitForSelector('.product-card')
      const items = await page.$$eval('.product-card', cards =>
        cards.map(c => ({
          name: c.querySelector('.name')?.textContent,
          price: c.querySelector('.price')?.textContent,
        }))
      )

      await Dataset.pushData({ items, url: request.url })
    }
  },
})

await crawler.run(['https://example.com/products'])

// Export results
const dataset = await Dataset.open()
await dataset.exportToJSON('output.json')
```

## PlaywrightCrawler with Proxy Rotation

```javascript
import { PlaywrightCrawler, ProxyConfiguration } from 'crawlee'

const proxyConfiguration = new ProxyConfiguration({
  proxyUrls: [
    'http://user:pass@proxy1.example.com:8080',
    'http://user:pass@proxy2.example.com:8080',
    'http://user:pass@proxy3.example.com:8080',
  ],
})

const crawler = new PlaywrightCrawler({
  proxyConfiguration,
  useSessionPool: true,
  sessionPoolOptions: {
    maxPoolSize: 20,
    sessionOptions: {
      maxUsageCount: 50, // rotate after 50 requests per session
    },
  },

  async requestHandler({ page, request, session }) {
    // Crawlee automatically rotates proxies and manages sessions
    const data = await page.$$eval('.item', items =>
      items.map(item => ({ /* ... */ }))
    )

    // Mark session as good if request succeeded
    session?.markGood()

    await Dataset.pushData(data)
  },

  async failedRequestHandler({ request, error }) {
    console.error(`Failed: ${request.url} — ${error.message}`)
  },
})
```

## HttpCrawler for API Scraping

When you need managed retries and queuing for API calls:

```javascript
import { HttpCrawler, Dataset } from 'crawlee'

const crawler = new HttpCrawler({
  maxRequestsPerCrawl: 1000,
  maxConcurrency: 3,
  requestHandlerTimeoutSecs: 30,

  async requestHandler({ request, body }) {
    const data = JSON.parse(body)

    // Process and store
    await Dataset.pushData({
      source: request.url,
      results: data.results,
      total: data.total,
    })

    // Enqueue next page if paginated
    if (data.nextPage) {
      await crawler.addRequests([{ url: data.nextPage }])
    }
  },
})

await crawler.run([
  'https://api.example.com/v1/data?page=1',
])
```

## CheerioCrawler for HTML

```javascript
import { CheerioCrawler, Dataset } from 'crawlee'

const crawler = new CheerioCrawler({
  maxRequestsPerCrawl: 200,

  async requestHandler({ request, $, enqueueLinks }) {
    // Parse with Cheerio
    const items = $('div.product').map((_, el) => ({
      name: $(el).find('h3').text().trim(),
      price: $(el).find('.price').text().trim(),
      url: $(el).find('a').attr('href'),
    })).get()

    await Dataset.pushData(items)

    // Auto-enqueue links matching a pattern
    await enqueueLinks({
      globs: ['https://example.com/products/page/*'],
    })
  },
})
```

## Session Management

Crawlee manages cookies and sessions automatically:

```javascript
const crawler = new PlaywrightCrawler({
  useSessionPool: true,
  persistCookiesPerSession: true,
  sessionPoolOptions: {
    maxPoolSize: 10,
    sessionOptions: {
      maxUsageCount: 100,    // max requests per session
      maxErrorScore: 3,      // retire session after 3 errors
    },
  },
})
```

## Request Queue (Persistent)

Survives restarts, handles deduplication:

```javascript
import { RequestQueue } from 'crawlee'

const queue = await RequestQueue.open('my-crawl')

// Add URLs (auto-deduped by URL)
await queue.addRequests([
  { url: 'https://example.com/page/1', userData: { type: 'list' } },
  { url: 'https://example.com/page/2', userData: { type: 'list' } },
])

// Use with crawler
const crawler = new PlaywrightCrawler({
  requestQueue: queue,
  async requestHandler({ request }) {
    if (request.userData.type === 'list') {
      // Handle list page
    }
  },
})
```

## Fingerprinting (Anti-Detection)

Crawlee can randomize browser fingerprints using browserforge:

```javascript
const crawler = new PlaywrightCrawler({
  browserPoolOptions: {
    fingerprintOptions: {
      fingerprintGeneratorOptions: {
        browsers: ['chrome'],
        operatingSystems: ['macos', 'windows'],
        locales: ['en-US'],
      },
    },
  },
})
```

## Key Advantages Over Raw Playwright

| Feature | Raw Playwright | Crawlee |
|---------|---------------|---------|
| Proxy rotation | Manual | Built-in |
| Session management | Manual cookies | Automatic pool |
| Retry logic | Manual try/catch | Built-in with backoff |
| Request queuing | Manual array | Persistent RequestQueue |
| Concurrency control | Manual | Automatic with `maxConcurrency` |
| Fingerprinting | Manual scripts | Built-in browserforge |
| Data export | Manual file I/O | Dataset with JSON/CSV export |
| Adaptive HTTP/browser | Not possible | AdaptivePlaywrightCrawler |
