---
name: data-normalization
description: Standardized schemas and normalization patterns for scraped data. Use when defining output formats, validating scraped data with Zod, or transforming raw data into consistent JSON structures.
---

# Data Normalization

## Standard Output Schemas

### Ships Schema

```javascript
import { z } from 'zod'

const ShipSchema = z.object({
  name: z.string(),
  slug: z.string().regex(/^[a-z0-9-]+$/),
  shipCode: z.string().nullable(),
  image_url: z.string().url().nullable(),
  url: z.string().url(),
})

const ShipsOutput = z.object({
  ships: z.array(ShipSchema),
})
```

### Sailings Schema

```javascript
const SailingSchema = z.object({
  cruise_line: z.string(),
  cruise_line_slug: z.string(),
  ship: z.string(),
  shipCode: z.string(),
  embark_date: z.string().regex(/^\d{4}-\d{2}-\d{2}$/),
  debark_date: z.string().regex(/^\d{4}-\d{2}-\d{2}$/),
  embark_city: z.string(),
  debark_city: z.string(),
  url: z.string().url(),
  itinerary: z.array(z.string()),
  guest_count: z.number().int().positive(),
  best_price: z.object({
    price: z.number().positive().nullable(),
    room_type: z.enum(['interior', 'ocean_view', 'balcony', 'suite']).nullable(),
  }),
  combo: z.boolean().default(false),
})
```

### Reviews Schema

```javascript
const ReviewSchema = z.object({
  source: z.string(),           // 'google', 'yelp', 'cars_com'
  author: z.string(),
  date: z.string().regex(/^\d{4}-\d{2}-\d{2}$/),
  rating: z.number().min(1).max(5),
  text: z.string(),
  ownerReply: z.string().nullable(),
  externalId: z.string().nullable(),
})
```

## Normalization Functions

### Date Normalization

```javascript
// Always output YYYY-MM-DD
function normalizeDate(raw) {
  // Handle various input formats
  const formats = [
    // "March 15, 2024"
    /^(\w+)\s+(\d{1,2}),?\s+(\d{4})$/,
    // "15/03/2024"
    /^(\d{1,2})\/(\d{1,2})\/(\d{4})$/,
    // "2024-03-15" (already correct)
    /^(\d{4})-(\d{2})-(\d{2})$/,
    // "03/15/2024"
    /^(\d{2})\/(\d{2})\/(\d{4})$/,
  ]

  // Use Date constructor as fallback
  const d = new Date(raw)
  if (isNaN(d.getTime())) return null

  return d.toISOString().split('T')[0] // YYYY-MM-DD
}
```

### Slug Generation

```javascript
function slugify(text) {
  return text
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, '-')
    .replace(/^-|-$/g, '')
}
```

### Price Normalization

```javascript
function normalizePrice(raw) {
  if (!raw || raw === 'N/A' || raw === 'Sold Out') {
    return { price: null, room_type: null }
  }

  // Handle "$1,234" or "1234" or "$1,234.56"
  const price = typeof raw === 'string'
    ? parseFloat(raw.replace(/[$,]/g, ''))
    : raw

  return {
    price: isNaN(price) ? null : Math.round(price),
    room_type: null, // Set by caller if known
  }
}
```

### Room Type Normalization

```javascript
function normalizeRoomType(raw) {
  if (!raw) return null

  const lower = raw.toLowerCase()
  const mapping = {
    interior: ['interior', 'inside', 'inner'],
    ocean_view: ['ocean view', 'oceanview', 'outside', 'porthole'],
    balcony: ['balcony', 'verandah', 'veranda'],
    suite: ['suite', 'grand suite', 'penthouse'],
  }

  for (const [normalized, variants] of Object.entries(mapping)) {
    if (variants.some(v => lower.includes(v))) return normalized
  }
  return null
}
```

## Validation & Output

```javascript
// Validate all scraped data before writing
function validateAndWrite(data, schema, outputPath) {
  const result = schema.safeParse(data)

  if (!result.success) {
    console.error('Validation errors:')
    for (const issue of result.error.issues) {
      console.error(`  ${issue.path.join('.')}: ${issue.message}`)
    }
    // Write anyway but log warnings
    console.warn(`Writing ${data.length} records with ${result.error.issues.length} validation issues`)
  }

  writeFileSync(outputPath, JSON.stringify(data, null, 2))
  console.log(`Wrote ${Array.isArray(data) ? data.length : 1} records to ${outputPath}`)
}
```

## Deduplication

```javascript
function deduplicate(sailings) {
  const seen = new Set()
  return sailings.filter(s => {
    const key = `${s.shipCode}-${s.embark_date}-${s.debark_date}`
    if (seen.has(key)) return false
    seen.add(key)
    return true
  })
}
```

## Output File Naming Convention

```
cruises/output/{cruise_line_slug}_ships.json
cruises/output/sailings_{cruise_line_slug}_{shipCode}_{extra}.json
cruises/output/sailings_{cruise_line_slug}_all.json
reviews/output/{source}_{location_slug}.json
```
