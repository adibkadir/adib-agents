---
name: http-replay-scraping
description: Core scraping technique — intercept network requests via DevTools, strip down to essential headers, and replay API calls with native fetch. Covers REST, GraphQL, Solr, Algolia, pagination patterns, cookie/session management, and anti-detection headers. This is Tier 1 and should always be tried first.
---

# HTTP Replay Scraping (Tier 1)

The primary approach: find the API call, copy the headers, replay it with `fetch`.

## Mobile Web First

Before scraping the desktop site, always check if a mobile version exists. Mobile sites have simpler DOM, fewer resources, less JavaScript, lighter bot detection, and no mouse/hover concerns.

### Check for Mobile Version

```javascript
// Option 1: Try m.example.com subdomain
const MOBILE_URLS = [
  'https://m.example.com/cruises',
  'https://mobile.example.com/cruises',
  'https://www.example.com/cruises', // with mobile UA
]

// Option 2: Send mobile User-Agent to the same URL
const MOBILE_UA = 'Mozilla/5.0 (Linux; Android 14; Pixel 8) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Mobile Safari/537.36'

const response = await fetch('https://www.example.com/cruises', {
  headers: {
    'User-Agent': MOBILE_UA,
    'Accept': 'text/html,application/xhtml+xml',
    'Accept-Language': 'en-US,en;q=0.9',
  },
})
```

### Compare Mobile vs Desktop APIs

```javascript
// The mobile site often hits simpler, lighter APIs
// 1. Load mobile page, check DevTools Network tab
// 2. Compare mobile API response vs desktop API response
// 3. If mobile has the same data fields → use mobile (simpler, faster)
// 4. If mobile is missing fields you need → fall back to desktop

// Common wins:
// - m.example.com/api/search returns flat JSON (desktop returns nested + UI metadata)
// - Mobile APIs skip image URLs, ad payloads, and tracking fields
// - Mobile APIs often have simpler auth (no CSRF tokens, fewer cookies)
// - Mobile pages load faster → less bot detection timeout risk
```

### When to Fall Back to Desktop

- Mobile version is missing critical data fields (prices, dates, etc.)
- Mobile API returns less data per page (smaller page sizes)
- Mobile site redirects to app store instead of showing content
- Mobile version is a completely different product (different data source)

## mitmproxy Debugging

When you're getting 403s, empty responses, or bot challenges and can't figure out why, use mitmproxy to compare a real browser request against your scripted request side-by-side.

### Setup

```bash
# Install mitmproxy
brew install mitmproxy   # macOS
pip install mitmproxy    # or via pip

# Start mitmproxy in transparent mode
mitmproxy -p 8080

# Or use mitmweb for a browser-based UI
mitmweb -p 8080
```

### Capture Real Browser Traffic

```bash
# Configure Chrome to use the proxy
# macOS: System Preferences → Network → Proxies → HTTP Proxy → localhost:8080
# Or launch Chrome with proxy flag:
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --proxy-server=http://localhost:8080 \
  --ignore-certificate-errors

# Browse the target site normally. mitmproxy captures every request.
# Find the API call you want to replay. Note ALL headers, cookies, TLS info.
```

### Capture Scripted Request

```bash
# Run your Node script through the proxy
HTTPS_PROXY=http://localhost:8080 \
NODE_TLS_REJECT_UNAUTHORIZED=0 \
node scraper.mjs

# Or in your fetch code:
import { ProxyAgent } from 'undici'

const response = await fetch(url, {
  headers: HEADERS,
  dispatcher: new ProxyAgent('http://localhost:8080'),
})
```

### Diff and Fix

Compare the two requests in mitmproxy and fix differences:

```
Real browser request vs Your scripted request
─────────────────────────────────────────────
✓ URL & method              → Should match exactly
✓ Headers                   → Order, casing, missing headers
✓ Cookie values             → Session cookies, tracking cookies
✓ TLS fingerprint           → JA3/JA4 hash differences
✓ HTTP/2 vs HTTP/1.1        → Protocol version matters for some WAFs
✓ Header order              → Some bot detectors check header ordering
✓ Accept-Encoding           → Missing compression headers = bot signal
✓ Sec-Ch-Ua headers         → Client hints must match User-Agent
✓ Request timing            → Too-fast requests = bot signal
```

### Common Fixes After Diffing

```javascript
// Fix 1: Add missing Sec-Ch-Ua headers (Chrome client hints)
const headers = {
  'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) ...',
  'Sec-Ch-Ua': '"Chromium";v="131", "Not_A Brand";v="24"',
  'Sec-Ch-Ua-Mobile': '?0',
  'Sec-Ch-Ua-Platform': '"macOS"',
}

// Fix 2: Match header order (use array of tuples if needed)
// Some WAFs (Akamai, Datadome) check that headers arrive in browser order

// Fix 3: Add cookies from initial page load
// Real browsers send cookies from prior page views

// Fix 4: Match Accept-Encoding exactly
// 'gzip, deflate, br, zstd' (Chrome 131+) vs 'gzip' (default Node)
```

Iterate: fix one difference at a time, re-send through mitmproxy, compare again. When your scripted request is indistinguishable from the real one, the 403s stop.

## Discovery Process

### Step 1: Network Tab Interception

Open the target site in Chrome. DevTools → Network → Filter: Fetch/XHR. Browse the site and watch for JSON responses.

```
1. Open target site in Chrome
2. DevTools → Network tab → Filter: Fetch/XHR
3. Interact with the page (search, paginate, filter, scroll)
4. Look for JSON responses (check Response tab)
5. Right-click request → Copy → Copy as cURL
6. Right-click request → Copy → Copy as fetch (Node.js)
7. Test the cURL in terminal — does it work outside the browser?
```

### Step 2: Strip Headers to Minimum

Start with all headers from the browser, then remove them one by one until you find the minimum set that still works:

```javascript
// Start with everything from the browser
const ALL_HEADERS = {
  'User-Agent': 'Mozilla/5.0 ...',
  'Accept': 'application/json',
  'Accept-Language': 'en-US,en;q=0.9',
  'Accept-Encoding': 'gzip, deflate, br',
  'Referer': 'https://example.com/cruises',
  'Origin': 'https://example.com',
  'DNT': '1',
  'Connection': 'keep-alive',
  'Sec-Fetch-Dest': 'empty',
  'Sec-Fetch-Mode': 'cors',
  'Sec-Fetch-Site': 'same-origin',
  'Cookie': 'session=abc123; _ga=...',
  'X-Client-Id': 'web-app',
  'X-Request-Id': crypto.randomUUID(),
}

// Test: which headers can you remove and still get 200?
// Usually the minimum is: User-Agent + Accept + Referer
// Sometimes just User-Agent alone works
```

### Step 3: Reproduce in Node.js

```javascript
const response = await fetch('https://api.example.com/v2/data', {
  headers: {
    'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36',
    'Accept': 'application/json',
    'Referer': 'https://example.com/',
  },
})

if (!response.ok) {
  const text = await response.text()
  throw new Error(`${response.status}: ${text.slice(0, 500)}`)
}

const data = await response.json()
```

## API Types

### REST API Replay

```javascript
// Direct query params
const url = new URL('https://api.example.com/sailings')
url.searchParams.set('ship', 'carnival-celebration')
url.searchParams.set('guests', '2')
url.searchParams.set('sort', 'date')
url.searchParams.set('limit', '100')

const response = await fetch(url, { headers: HEADERS })
const data = await response.json()
```

### GraphQL Replay

Found by analyzing network requests or minified JS bundles:

```javascript
const GRAPHQL_URL = 'https://www.example.com/graphql'

const query = `
  query GetSailings($input: SailingSearchInput!) {
    sailingSearch(input: $input) {
      sailings {
        shipName
        embarkDate
        debarkDate
        itinerary { portName }
        pricing { interior oceanView balcony suite }
      }
    }
  }
`

const response = await fetch(GRAPHQL_URL, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'User-Agent': 'Mozilla/5.0 ...',
  },
  body: JSON.stringify({
    query,
    variables: { input: { guests: 2, page: 1, pageSize: 50 } },
  }),
})
```

### Solr / Search API Replay

```javascript
// Discover facets first (what values exist?)
const facetUrl = new URL('https://example.com/solr/sailings/select')
facetUrl.searchParams.set('q', '*:*')
facetUrl.searchParams.set('facet', 'true')
facetUrl.searchParams.set('facet.field', 'shipCode')
facetUrl.searchParams.set('rows', '0')

const facets = await fetch(facetUrl).then(r => r.json())
// facets.facet_counts.facet_fields.shipCode → ['AB', 10, 'CD', 8, ...]

// Then query with specific filters
facetUrl.searchParams.set('q', 'shipCode:AB')
facetUrl.searchParams.set('rows', '100')
facetUrl.searchParams.set('sort', 'embarkDate asc')
```

### Algolia BFF Replay

```javascript
// Algolia search keys are public (found in page source)
const response = await fetch('https://algoliabff.example.com/v4/search', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'X-Algolia-Application-Id': 'APP_ID',   // from page source
    'X-Algolia-API-Key': 'SEARCH_KEY',      // public search-only key
  },
  body: JSON.stringify({
    query: '',
    filters: 'shipCode:FANTASTICA',
    hitsPerPage: 100,
    page: 0,
  }),
})
```

## Pagination Patterns

### Offset-Based

```javascript
let offset = 0
const limit = 100
const allResults = []

while (true) {
  const url = `${apiUrl}?offset=${offset}&limit=${limit}`
  const data = await fetch(url, { headers }).then(r => r.json())
  allResults.push(...data.results)
  if (data.results.length < limit) break
  offset += limit
  await sleep(500) // rate limit
}
```

### Cursor-Based

```javascript
let cursor = null
const allResults = []

while (true) {
  const url = cursor ? `${apiUrl}?cursor=${cursor}` : apiUrl
  const data = await fetch(url, { headers }).then(r => r.json())
  allResults.push(...data.results)
  cursor = data.nextCursor
  if (!cursor) break
  await sleep(500)
}
```

### Date Windowing (for APIs with result limits)

```javascript
// When API limits results per query, slide a date window across the range
function dateWindows(startDate, endDate, windowDays = 14) {
  const windows = []
  let current = new Date(startDate)
  const end = new Date(endDate)

  while (current < end) {
    const windowEnd = new Date(current)
    windowEnd.setDate(windowEnd.getDate() + windowDays)
    if (windowEnd > end) windowEnd.setTime(end.getTime())

    windows.push({
      from: current.toISOString().split('T')[0],
      to: windowEnd.toISOString().split('T')[0],
    })

    current = new Date(windowEnd)
    current.setDate(current.getDate() + 1)
  }
  return windows
}

// Fetch each window, dedup by ID
const allResults = []
const seen = new Set()

for (const window of dateWindows('2025-01-01', '2027-06-30')) {
  const data = await fetch(`${apiUrl}?from=${window.from}&to=${window.to}`, { headers })
    .then(r => r.json())

  for (const item of data.results) {
    if (!seen.has(item.id)) {
      seen.add(item.id)
      allResults.push(item)
    }
  }
  await sleep(500)
}
```

## Cookie & Session Management

### Stateless APIs (most common)

Most undocumented APIs don't require cookies. Just send realistic headers:

```javascript
const HEADERS = {
  'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36',
  'Accept': 'application/json, text/plain, */*',
  'Accept-Language': 'en-US,en;q=0.9',
}
```

### Cookie-Based Sessions

When the API requires cookies from a prior page load:

```javascript
// Step 1: Hit the main page to get session cookies
const initialResponse = await fetch('https://example.com', {
  headers: HEADERS,
  redirect: 'manual', // capture Set-Cookie headers
})

// Step 2: Extract cookies from response
const cookies = initialResponse.headers.getSetCookie?.() || []
const cookieString = cookies.map(c => c.split(';')[0]).join('; ')

// Step 3: Use cookies in API calls
const apiResponse = await fetch('https://example.com/api/data', {
  headers: {
    ...HEADERS,
    'Cookie': cookieString,
  },
})
```

### Token-Based Auth

When the API requires a token generated by the page:

```javascript
// Step 1: Fetch the page HTML
const html = await fetch('https://example.com').then(r => r.text())

// Step 2: Extract token from HTML/script tags
const tokenMatch = html.match(/apiToken['":\s]+['"]([^'"]+)['"]/)
const token = tokenMatch?.[1]

// Step 3: Use token in API calls
const data = await fetch('https://example.com/api/data', {
  headers: {
    ...HEADERS,
    'Authorization': `Bearer ${token}`,
  },
}).then(r => r.json())
```

## Anti-Detection Headers

Standard header set that mimics a real Chrome browser:

```javascript
const CHROME_HEADERS = {
  'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36',
  'Accept': 'application/json, text/plain, */*',
  'Accept-Language': 'en-US,en;q=0.9',
  'Accept-Encoding': 'gzip, deflate, br',
  'DNT': '1',
  'Connection': 'keep-alive',
  'Sec-Fetch-Dest': 'empty',
  'Sec-Fetch-Mode': 'cors',
  'Sec-Fetch-Site': 'same-origin',
  'Sec-Ch-Ua': '"Chromium";v="131", "Not_A Brand";v="24"',
  'Sec-Ch-Ua-Mobile': '?0',
  'Sec-Ch-Ua-Platform': '"macOS"',
}
```

## JS Bundle Analysis

Find hidden API endpoints in minified JavaScript:

```bash
# Find JS bundles
curl -s 'https://example.com' | grep -oP 'src="[^"]*\.js"' | head -10

# Download the main bundle
curl -s 'https://example.com/_next/static/chunks/main-abc123.js' > bundle.js

# Break minified code into readable lines
cat bundle.js | tr ';' '\n' | grep -i 'api\|endpoint\|graphql\|fetch' | head -30

# Find GraphQL query strings
cat bundle.js | tr ';' '\n' | grep -i 'query\|mutation' | head -20

# Find API base URLs
cat bundle.js | tr ';' '\n' | grep -E 'https?://[a-zA-Z0-9.-]+/api' | head -10

# Check for swagger/openapi docs
curl -s 'https://example.com/swagger.json' | head -5
curl -s 'https://example.com/openapi.json' | head -5
curl -s 'https://example.com/api-docs' | head -5
```

## Rate Limiting

```javascript
function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms))
}

// Standard: 500ms between requests
await sleep(500)

// Aggressive rate limiting for sensitive targets
await sleep(2000)

// Random jitter to look more human
await sleep(500 + Math.random() * 1000)
```

## Error Handling

```javascript
async function fetchWithRetry(url, options, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const response = await fetch(url, options)

      if (response.status === 429) {
        const retryAfter = response.headers.get('Retry-After')
        const waitMs = retryAfter ? parseInt(retryAfter) * 1000 : 5000 * attempt
        console.warn(`Rate limited (429). Waiting ${waitMs}ms...`)
        await sleep(waitMs)
        continue
      }

      if (response.status === 403) {
        console.error(`Blocked (403). Headers may be insufficient or bot detection triggered.`)
        throw new Error('Blocked by bot detection — escalate to Tier 2')
      }

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${await response.text().then(t => t.slice(0, 200))}`)
      }

      return response
    } catch (error) {
      if (attempt === maxRetries) throw error
      console.warn(`Attempt ${attempt} failed: ${error.message}. Retrying...`)
      await sleep(1000 * attempt)
    }
  }
}
```
