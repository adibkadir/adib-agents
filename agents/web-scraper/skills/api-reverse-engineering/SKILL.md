---
name: api-reverse-engineering
description: Techniques for discovering undocumented APIs from websites including REST, GraphQL, Solr, and Algolia endpoints. Use when you need to find hidden API endpoints by analyzing network traffic and JavaScript bundles.
---

# API Reverse Engineering

## Discovery Techniques

### 1. Network Tab Analysis (First Step)

Open DevTools Network tab, filter by XHR/Fetch, and browse the site:

```
1. Open target site in browser
2. DevTools → Network tab → Filter: Fetch/XHR
3. Interact with the page (search, paginate, filter)
4. Look for JSON responses
5. Right-click request → Copy as cURL
6. Reproduce with fetch in Node.js
```

### 2. Minified JS Bundle Analysis

Find API endpoints hidden in JavaScript:

```bash
# Download and analyze the main JS bundle
curl -s 'https://example.com' | grep -oP 'src="[^"]*\.js"' | head -5

# Download the bundle
curl -s 'https://example.com/_next/static/chunks/main-abc123.js' > bundle.js

# Break minified code into readable lines
cat bundle.js | tr ';' '\n' | grep -i 'api\|endpoint\|graphql\|fetch' | head -20

# Find GraphQL queries
cat bundle.js | tr ';' '\n' | grep -i 'query\|mutation' | head -10
```

### 3. REST API Discovery

```javascript
// Common patterns to try:
const baseUrls = [
  '/api/',
  '/api/v1/',
  '/api/v2/',
  '/_api/',
  '/rest/',
  '/graphql',
]

// Check for API documentation endpoints
const docUrls = [
  '/api/docs',
  '/swagger.json',
  '/openapi.json',
  '/api-docs',
]
```

### 4. GraphQL Endpoint Discovery

```javascript
// Royal Caribbean / Celebrity pattern:
// Found by analyzing Next.js bundle, extracted query from AST

const GRAPHQL_ENDPOINT = 'https://www.example.com/graphql'

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

const response = await fetch(GRAPHQL_ENDPOINT, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    query,
    variables: { input: { cruiseLine: 'RC', guests: 2 } },
  }),
})
```

### 5. Solr / Search API Discovery

```javascript
// Holland America / Seabourn pattern:
// Sites using Solr expose faceted search APIs

// Step 1: Find the Solr endpoint
// Look for URLs containing /solr/ or /search/ in network traffic

// Step 2: Use facets to discover available values
const facetUrl = new URL('https://example.com/solr/sailings/select')
facetUrl.searchParams.set('q', '*:*')
facetUrl.searchParams.set('facet', 'true')
facetUrl.searchParams.set('facet.field', 'shipCode')
facetUrl.searchParams.set('facet.field', 'embarkPort')
facetUrl.searchParams.set('rows', '0')

const facets = await fetch(facetUrl).then(r => r.json())
// facets.facet_counts.facet_fields.shipCode → ['AB', 10, 'CD', 8, ...]

// Step 3: Query with specific parameters
facetUrl.searchParams.set('q', 'shipCode:AB')
facetUrl.searchParams.set('rows', '100')
facetUrl.searchParams.set('sort', 'embarkDate asc')
```

### 6. Algolia / BFF API Discovery

```javascript
// MSC Cruises pattern:
// Sites using Algolia expose a search BFF

const response = await fetch('https://example.com/api/search', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'X-Algolia-Application-Id': 'APP_ID',   // Found in page source
    'X-Algolia-API-Key': 'SEARCH_KEY',      // Public search-only key
  },
  body: JSON.stringify({
    query: '',
    filters: 'shipCode:FANTASTICA',
    hitsPerPage: 100,
  }),
})
```

## Reproducing Requests

Once you find an API endpoint, reproduce it in Node.js:

```javascript
// Copy the exact headers from the browser
const response = await fetch(apiUrl, {
  headers: {
    'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) ...',
    'Accept': 'application/json',
    'Accept-Language': 'en-US,en;q=0.9',
    'Referer': 'https://example.com/cruises',
    // Add any custom headers found in network tab
    'X-Client-Id': 'web-app',
    'X-Request-Id': crypto.randomUUID(),
  },
})
```

## Pagination Patterns

```javascript
// Offset-based
let offset = 0
const limit = 100
let allResults = []

while (true) {
  const url = `${apiUrl}?offset=${offset}&limit=${limit}`
  const data = await fetch(url).then(r => r.json())
  allResults.push(...data.results)
  if (data.results.length < limit) break
  offset += limit
}

// Cursor-based
let cursor = null
while (true) {
  const url = cursor ? `${apiUrl}?cursor=${cursor}` : apiUrl
  const data = await fetch(url).then(r => r.json())
  allResults.push(...data.results)
  cursor = data.nextCursor
  if (!cursor) break
}
```

## Anti-Detection Headers

Always include realistic headers to avoid being blocked:

```javascript
const HEADERS = {
  'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
  'Accept': 'application/json, text/plain, */*',
  'Accept-Language': 'en-US,en;q=0.9',
  'Accept-Encoding': 'gzip, deflate, br',
  'DNT': '1',
  'Connection': 'keep-alive',
  'Sec-Fetch-Dest': 'empty',
  'Sec-Fetch-Mode': 'cors',
  'Sec-Fetch-Site': 'same-origin',
}
```
