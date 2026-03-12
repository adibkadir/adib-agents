---
name: firecrawl
description: "Tier 4 — Firecrawl paid API for scraping bot-protected websites with managed proxy infrastructure, auto-escalation, structured LLM extraction, crawling, search, and browser sandbox. Use when free tools (fetch, Playwright, Crawlee) cannot bypass bot protection. Ask the user for a Firecrawl API key before using."
---

# Firecrawl API (Tier 4)

Firecrawl is a paid API that handles bot detection bypass with managed proxy infrastructure. Use it when free tools fail.

**Install:** `pnpm add @mendable/firecrawl-js`

**Docs:** https://docs.firecrawl.dev

**Requires:** `FIRECRAWL_API_KEY` environment variable. **Ask the user for this key before using Firecrawl.**

## When to Escalate to Tier 4

- Tier 3 Crawlee with proxy rotation still gets blocked
- Site has Queue-it, Datadome, or advanced Cloudflare protection
- You need structured data extraction from complex pages (LLM-powered)
- You need to crawl many pages of a bot-protected site
- Speed matters more than cost — Firecrawl is faster to set up than self-managed proxies

## Cost Awareness

| Operation | Cost | Notes |
|-----------|------|-------|
| Scrape (markdown/html) | 1 credit/page | Base scraping |
| Scrape (rawHtml) | 1 credit/page | No processing |
| Extract (LLM structured) | 5 credits/page | JSON extraction with Zod schema |
| Enhanced mode | +4 credits/page | Better anti-bot |
| Stealth proxy | 5 credits/page | For Datadome, advanced protection |
| Crawl | 1 credit/page discovered | Per-page billing |
| Map (URL discovery) | 1 credit/call | Returns URL list only |
| Search | 2 credits/10 results | Plus per-page scrape costs |
| Browser sandbox | 2 credits/minute | Interactive CDP sessions |
| Agent | Dynamic | 5 free daily runs |

**Plans:** Free (500 one-time credits), Hobby ($19/mo, 3K credits), Standard ($99/mo, 100K credits)

## Setup

```javascript
import Firecrawl from '@mendable/firecrawl-js'

const firecrawl = new Firecrawl({
  apiKey: process.env.FIRECRAWL_API_KEY,
})
```

## Basic Scrape (Markdown)

Get clean markdown from any page — best for content extraction:

```javascript
const result = await firecrawl.scrapeUrl('https://example.com/page', {
  formats: ['markdown'],
  timeout: 30000,
})

if (result.success) {
  const content = result.markdown
  // Parse markdown with regex or string matching
}
```

## Scrape with Raw HTML

When you need to regex-parse HTML directly:

```javascript
const result = await firecrawl.scrapeUrl('https://example.com/reviews', {
  formats: ['rawHtml'],
  timeout: 60000,
})

if (result.success) {
  const html = result.rawHtml
  // Parse with regex, Cheerio, etc.
}
```

## Structured Extraction (LLM-Powered)

Extract specific data using a Zod schema — Firecrawl uses an LLM to parse the page:

```javascript
import { z } from 'zod'

const ReviewSchema = z.object({
  reviews: z.array(z.object({
    author: z.string(),
    date: z.string(),
    rating: z.number().min(1).max(5),
    text: z.string(),
    ownerReply: z.string().optional(),
  })),
  totalReviews: z.number().optional(),
  averageRating: z.number().optional(),
})

const result = await firecrawl.scrapeUrl('https://example.com/reviews', {
  formats: ['extract'],
  extract: {
    schema: ReviewSchema,
    prompt: 'Extract all customer reviews including author, date, star rating, review text, and any business owner replies.',
  },
})

if (result.success && result.extract) {
  const reviews = result.extract.reviews
}
```

**Cost:** 5 credits/page for extraction. Use sparingly — prefer markdown + regex when possible.

## Scrape with Browser Actions

Interact with the page before extracting content:

```javascript
const result = await firecrawl.scrapeUrl('https://example.com/data', {
  formats: ['markdown'],
  timeout: 60000,
  actions: [
    { type: 'wait', milliseconds: 2000 },
    { type: 'click', selector: 'button.load-more' },
    { type: 'wait', milliseconds: 3000 },
    { type: 'scroll', direction: 'down' },
    { type: 'wait', milliseconds: 2000 },
    { type: 'screenshot' }, // capture a screenshot
  ],
})
```

Available actions: `wait`, `click`, `write`, `press`, `scroll`, `screenshot`, `executeJavascript`, `pdf`

**Limit:** Combined wait time across all actions must not exceed 60 seconds.

## Stealth Proxy Mode

For sites with Datadome, advanced Cloudflare, or aggressive bot detection:

```javascript
const result = await firecrawl.scrapeUrl('https://heavily-protected-site.com', {
  formats: ['rawHtml'],
  timeout: 60000,
  proxy: 'stealth', // 5 credits/page
})
```

## Multi-Page Crawling

Crawl an entire site with path filtering:

```javascript
const crawlResult = await firecrawl.crawlUrl('https://example.com', {
  limit: 100,
  maxDepth: 3,
  includePaths: ['/products/*', '/catalog/*'],
  excludePaths: ['/blog/*', '/news/*', '/about'],
  formats: ['markdown'],
})

for (const page of crawlResult.data) {
  console.log(`URL: ${page.metadata.sourceURL}`)
  console.log(`Content: ${page.markdown.substring(0, 200)}...`)
}
```

## URL Discovery (Map)

Get all URLs from a site without scraping content (1 credit total):

```javascript
const mapResult = await firecrawl.mapUrl('https://example.com', {
  limit: 5000,
})

// mapResult.links contains all discovered URLs
const productUrls = mapResult.links.filter(url => url.includes('/product/'))
```

## Web Search

Search the web and get full page content for results:

```javascript
const searchResult = await firecrawl.search('best cruise deals 2025', {
  limit: 10,
  scrapeOptions: {
    formats: ['markdown'],
  },
})

for (const result of searchResult.data) {
  console.log(`${result.title}: ${result.url}`)
  console.log(result.markdown.substring(0, 200))
}
```

## Agent (Autonomous Multi-Page)

For complex multi-page data gathering with natural language instructions:

```javascript
const agentResult = await firecrawl.agent({
  prompt: 'Find all cruise ship specifications for Royal Caribbean fleet. Get ship name, year built, passenger capacity, and tonnage for each ship.',
  urls: ['https://www.royalcaribbean.com/fleet'],
  schema: ShipSchema,
  maxCredits: 50, // spending limit
})
```

**Cost:** Dynamic, based on pages visited. 5 free daily runs.

## Browser Sandbox (CDP Access)

Create an interactive browser session with Chrome DevTools Protocol:

```javascript
const session = await firecrawl.browser.create({
  ttl: 300, // 5 minutes
})

// session.cdpUrl — connect with Playwright or Puppeteer
// 2 credits/minute while session is active

await firecrawl.browser.delete(session.id) // cleanup
```

## Content Filtering

Control what content is extracted:

```javascript
const result = await firecrawl.scrapeUrl(url, {
  formats: ['markdown'],
  onlyMainContent: true,     // strip nav, footer, ads (default: true)
  includeTags: ['article', '.content', '#main'],
  excludeTags: ['nav', 'footer', '.sidebar', '.ads'],
})
```

## PDF Parsing

Firecrawl can parse PDFs:

```javascript
const result = await firecrawl.scrapeUrl('https://example.com/report.pdf', {
  formats: ['markdown'],
  parsers: [{ type: 'pdf', mode: 'auto', maxPages: 50 }],
})
// Modes: 'fast' (text only), 'auto' (text → OCR fallback), 'ocr' (force OCR)
```

## Pagination with Firecrawl

For paginated sites, manually iterate:

```javascript
const allReviews = []

for (let page = 0; page < 10; page++) {
  const url = `https://example.com/reviews?start=${page * 10}`
  const result = await firecrawl.scrapeUrl(url, {
    formats: ['rawHtml'],
    timeout: 60000,
  })

  if (!result.success) break

  // Parse reviews from HTML
  const reviews = parseReviews(result.rawHtml)
  if (reviews.length === 0) break

  allReviews.push(...reviews)
  await sleep(1000) // rate limit
}
```

## Error Handling

```javascript
try {
  const result = await firecrawl.scrapeUrl(url, { formats: ['markdown'] })

  if (!result.success) {
    console.error(`Firecrawl failed: ${result.error}`)
    // Consider: escalate to Tier 5 (mobile) or ask user for help
    return null
  }

  return result.markdown
} catch (error) {
  if (error.statusCode === 429) {
    console.error('Rate limited by Firecrawl. Wait and retry.')
  } else if (error.statusCode === 402) {
    console.error('Insufficient Firecrawl credits. Ask user to top up.')
  }
  throw error
}
```

## When to Use Which Format

| Format | Cost | Use When |
|--------|------|----------|
| `markdown` | 1 credit | Content extraction, text analysis |
| `rawHtml` | 1 credit | Regex parsing, Cheerio processing |
| `html` | 1 credit | Cleaned HTML (boilerplate removed) |
| `extract` | 5 credits | Structured data with Zod schema |
| `links` | 1 credit | URL discovery from a page |
| `screenshot` | 1 credit | Visual verification |

**Rule of thumb:** Use `markdown` or `rawHtml` (1 credit) and parse with regex/Cheerio yourself. Only use `extract` (5 credits) when the page structure is too complex for regex.
