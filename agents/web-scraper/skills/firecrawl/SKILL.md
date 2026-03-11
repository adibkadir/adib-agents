---
name: firecrawl
description: Firecrawl SDK for scraping bot-protected websites with proxy auto-escalation and structured data extraction. Use when sites have heavy bot protection (Queue-it, Akamai, Cloudflare) or when you need clean markdown/structured output from complex pages.
---

# Firecrawl SDK

## When to Use

Firecrawl is the escalation path when:
- Sites have Queue-it, Akamai, Cloudflare bot protection
- You need clean markdown from complex HTML
- Structured extraction with AI is needed
- Proxy rotation is required
- Simple fetch and Playwright both fail

## Setup

```javascript
import Firecrawl from '@mendable/firecrawl-js'

const firecrawl = new Firecrawl({ apiKey: process.env.FIRECRAWL_API_KEY })
```

## Basic Scrape

```javascript
const result = await firecrawl.scrapeUrl('https://example.com/ships', {
  formats: ['markdown', 'html'],
})

if (result.success) {
  const markdown = result.markdown
  const html = result.html
  // Parse the markdown/HTML for your data
}
```

## Structured Extraction

Extract specific data using a schema:

```javascript
import { z } from 'zod'

const ShipSchema = z.object({
  ships: z.array(z.object({
    name: z.string(),
    shipCode: z.string().optional(),
    imageUrl: z.string().url().optional(),
    yearBuilt: z.number().optional(),
  })),
})

const result = await firecrawl.scrapeUrl('https://example.com/fleet', {
  formats: ['extract'],
  extract: {
    schema: ShipSchema,
    prompt: 'Extract all cruise ships listed on this page with their names, codes, images, and build years.',
  },
})

if (result.success && result.extract) {
  const ships = result.extract.ships
}
```

## Crawl Multiple Pages

```javascript
const crawlResult = await firecrawl.crawlUrl('https://example.com', {
  limit: 50,
  includePaths: ['/ships/*', '/fleet/*'],
  excludePaths: ['/blog/*', '/news/*'],
  formats: ['markdown'],
})

for (const page of crawlResult.data) {
  console.log(`Page: ${page.metadata.sourceURL}`)
  console.log(`Content: ${page.markdown.substring(0, 200)}...`)
}
```

## Handling Protected Sites

Firecrawl auto-escalates through proxy tiers:
1. Direct request
2. Residential proxy
3. Premium proxy with browser rendering

No extra configuration needed — just call `scrapeUrl()` and Firecrawl handles escalation.

```javascript
// MSC Cruises (Queue-it protection) — just works
const result = await firecrawl.scrapeUrl('https://www.msccruises.com/fleet', {
  formats: ['markdown'],
  timeout: 30000,
})
```

## Rate Limiting

Firecrawl has built-in rate limiting. For batch operations, add your own delays:

```javascript
const urls = ['https://example.com/page1', 'https://example.com/page2', /* ... */]

for (const url of urls) {
  const result = await firecrawl.scrapeUrl(url, { formats: ['markdown'] })
  // Process result
  await new Promise(resolve => setTimeout(resolve, 1000)) // 1s delay
}
```

## Review Scraping Pattern (WebCred-style)

```javascript
const result = await firecrawl.scrapeUrl(reviewPageUrl, {
  formats: ['extract'],
  extract: {
    schema: z.object({
      reviews: z.array(z.object({
        author: z.string(),
        date: z.string(),
        rating: z.number().min(1).max(5),
        text: z.string(),
        ownerReply: z.string().optional(),
      })),
      totalReviews: z.number().optional(),
      averageRating: z.number().optional(),
    }),
    prompt: 'Extract all customer reviews from this page including author, date, star rating, review text, and any owner/business replies.',
  },
})
```
