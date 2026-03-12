---
name: news-research
description: Firecrawl-powered news discovery and content extraction. Search for trending news by topic, extract article content and images, summarize key facts, and output structured research data for news script generation. Use when you need to discover and process recent news articles.
---

# News Research

## When to Use

Use this skill whenever you need to:
- Discover recent news articles on a topic via Firecrawl search
- Extract article content, images, key facts, and quotes
- Rank and select the most relevant articles for a news video
- Download hero images for use in video composition
- Prepare structured research data for script generation

## Pipeline Overview

```
Topic Query (user input or NEWS_BRIEF_TEMPLATE.md)
  │
  ▼
Firecrawl Search (news sources, time-filtered)
  │
  ▼
Rank & Select Top Articles (3-5)
  │
  ▼
Firecrawl Scrape (each article URL → markdown)
  │
  ▼
Firecrawl Extract (structured facts, quotes, images)
  │
  ▼
Download Images (hero images → public/images/)
  │
  ▼
Output: research/articles.json + research/extracted-facts.json
```

## Step 1: News Search with Firecrawl

Use Firecrawl's search endpoint to discover recent news articles:

```typescript
import Firecrawl from '@mendable/firecrawl-js'

const firecrawl = new Firecrawl({ apiKey: process.env.FIRECRAWL_API_KEY })

interface NewsSearchConfig {
  topic: string
  maxArticles: number
  timeRange: 'past 24 hours' | 'past week' | 'past month'
  excludeSources?: string[]
}

const TIME_RANGE_MAP: Record<string, string> = {
  'past 24 hours': 'qdr:d',
  'past week': 'qdr:w',
  'past month': 'qdr:m',
}

async function searchNews(config: NewsSearchConfig) {
  const results = await firecrawl.search(config.topic, {
    limit: config.maxArticles * 2, // Fetch extra for ranking
    scrapeOptions: {
      formats: ['markdown'],
    },
  })

  // Filter out excluded sources
  let articles = results.data ?? []
  if (config.excludeSources?.length) {
    articles = articles.filter(
      (a) => !config.excludeSources!.some((s) => a.url?.includes(s))
    )
  }

  return articles.slice(0, config.maxArticles)
}
```

## Step 2: Structured Fact Extraction

Extract structured data from each article using Firecrawl's extract endpoint with a schema:

```typescript
import { z } from 'zod'

const NewsArticleSchema = z.object({
  headline: z.string(),
  summary: z.string().describe('2-3 sentence summary of the article'),
  keyFacts: z.array(z.string()).describe('Key factual claims, max 5'),
  quotes: z.array(
    z.object({
      text: z.string(),
      attribution: z.string(),
    })
  ),
  images: z.array(
    z.object({
      url: z.string().url(),
      caption: z.string().optional(),
      credit: z.string().optional(),
    })
  ),
  publishedDate: z.string(),
  source: z.string().describe('Publication name'),
  sourceUrl: z.string().url(),
  category: z
    .enum(['politics', 'technology', 'business', 'science', 'world', 'sports', 'entertainment'])
    .optional(),
})

type NewsArticle = z.infer<typeof NewsArticleSchema>

async function extractArticleFacts(articleUrl: string): Promise<NewsArticle> {
  const result = await firecrawl.scrapeUrl(articleUrl, {
    formats: ['extract'],
    extract: {
      schema: NewsArticleSchema,
    },
  })

  return NewsArticleSchema.parse(result.extract)
}
```

## Step 3: Batch Article Processing

Process multiple articles with rate limiting:

```typescript
interface ResearchOutput {
  articles: NewsArticle[]
  topImages: { url: string; localPath: string; caption?: string }[]
  topQuotes: { text: string; attribution: string; source: string }[]
  factSummary: string[]
}

async function processArticles(
  articleUrls: string[],
  outputDir: string
): Promise<ResearchOutput> {
  const articles: NewsArticle[] = []

  for (const url of articleUrls) {
    try {
      const article = await extractArticleFacts(url)
      articles.push(article)
      console.log(`Extracted: ${article.headline}`)
    } catch (error) {
      console.warn(`Failed to extract ${url}:`, error)
      continue
    }

    // Rate limit: 1 second between requests
    await new Promise((r) => setTimeout(r, 1000))
  }

  // Collect all images across articles
  const allImages = articles.flatMap((a) =>
    a.images.map((img) => ({ ...img, source: a.source }))
  )

  // Download top images
  const topImages = await downloadImages(
    allImages.slice(0, 8),
    `${outputDir}/public/images`
  )

  // Collect best quotes
  const topQuotes = articles
    .flatMap((a) => a.quotes.map((q) => ({ ...q, source: a.source })))
    .slice(0, 5)

  // Compile unique facts
  const factSummary = [...new Set(articles.flatMap((a) => a.keyFacts))]

  return { articles, topImages, topQuotes, factSummary }
}
```

## Step 4: Image Download

Fetch article hero images and save them locally for Remotion:

```typescript
import { mkdir, writeFile } from 'fs/promises'
import path from 'path'

async function downloadImages(
  images: { url: string; caption?: string; credit?: string }[],
  outputDir: string
): Promise<{ url: string; localPath: string; caption?: string }[]> {
  await mkdir(outputDir, { recursive: true })

  const downloaded: { url: string; localPath: string; caption?: string }[] = []

  for (let i = 0; i < images.length; i++) {
    const img = images[i]
    try {
      const response = await fetch(img.url)
      if (!response.ok) continue

      const contentType = response.headers.get('content-type') ?? 'image/jpeg'
      const ext = contentType.includes('png') ? 'png' : 'jpg'
      const filename = `article-${String(i + 1).padStart(2, '0')}-hero.${ext}`
      const localPath = path.join(outputDir, filename)

      const buffer = Buffer.from(await response.arrayBuffer())

      // Skip images smaller than 400px wide (too low quality)
      // In practice, check image dimensions with sharp or similar
      if (buffer.length < 5000) {
        console.warn(`Skipping small image: ${img.url}`)
        continue
      }

      await writeFile(localPath, buffer)
      downloaded.push({
        url: img.url,
        localPath: filename, // Relative to public/images/
        caption: img.caption,
      })

      console.log(`Downloaded: ${filename} (${(buffer.length / 1024).toFixed(0)}KB)`)
    } catch (error) {
      console.warn(`Failed to download ${img.url}:`, error)
    }
  }

  return downloaded
}
```

## Step 5: Article Ranking

Rank articles by relevance for the video:

```typescript
interface RankedArticle extends NewsArticle {
  score: number
}

function rankArticles(articles: NewsArticle[]): RankedArticle[] {
  return articles
    .map((article) => {
      let score = 0

      // Recency boost (more recent = higher score)
      const ageHours =
        (Date.now() - new Date(article.publishedDate).getTime()) / (1000 * 60 * 60)
      if (ageHours < 24) score += 30
      else if (ageHours < 72) score += 20
      else if (ageHours < 168) score += 10

      // Image availability (essential for video)
      score += Math.min(article.images.length * 10, 30)

      // Quote availability (great for video storytelling)
      score += Math.min(article.quotes.length * 8, 24)

      // Fact density (more facts = richer content)
      score += Math.min(article.keyFacts.length * 5, 25)

      // Summary length (indicates article depth)
      if (article.summary.length > 200) score += 10

      return { ...article, score }
    })
    .sort((a, b) => b.score - a.score)
}
```

## Step 6: Research Output

Save structured research data for the script generation skill:

```typescript
import { writeFile } from 'fs/promises'

async function saveResearch(
  research: ResearchOutput,
  outputDir: string
) {
  await mkdir(`${outputDir}/research`, { recursive: true })

  // Raw article data
  await writeFile(
    `${outputDir}/research/articles.json`,
    JSON.stringify(research.articles, null, 2)
  )

  // Structured facts for script generation
  const extractedFacts = {
    topicSummary: research.factSummary,
    sources: research.articles.map((a) => ({
      name: a.source,
      url: a.sourceUrl,
      headline: a.headline,
    })),
    availableImages: research.topImages.map((img) => ({
      path: img.localPath,
      caption: img.caption,
    })),
    bestQuotes: research.topQuotes,
    articleCount: research.articles.length,
  }

  await writeFile(
    `${outputDir}/research/extracted-facts.json`,
    JSON.stringify(extractedFacts, null, 2)
  )

  console.log(
    `Research saved: ${research.articles.length} articles, ` +
    `${research.topImages.length} images, ` +
    `${research.topQuotes.length} quotes`
  )
}
```

## User-Provided URLs

When the user provides specific article URLs instead of a search query:

```typescript
async function researchFromUrls(
  urls: string[],
  outputDir: string
): Promise<ResearchOutput> {
  // Skip search, go straight to extraction
  return processArticles(urls, outputDir)
}
```

## Advanced Patterns

### Handling Paywalled Articles

If Firecrawl returns thin content, the article may be paywalled. Fall back to the search result's snippet:

```typescript
async function extractWithFallback(url: string, searchSnippet?: string) {
  try {
    const result = await extractArticleFacts(url)
    if (result.summary.length < 50 && searchSnippet) {
      // Thin content — likely paywalled. Use search snippet instead.
      result.summary = searchSnippet
    }
    return result
  } catch {
    // Complete failure — build minimal article from search data
    return {
      headline: 'Article content unavailable',
      summary: searchSnippet ?? '',
      keyFacts: [],
      quotes: [],
      images: [],
      publishedDate: new Date().toISOString(),
      source: new URL(url).hostname,
      sourceUrl: url,
    }
  }
}
```

### Deduplication

When multiple articles cover the same story, deduplicate overlapping facts:

```typescript
function deduplicateFacts(facts: string[]): string[] {
  const unique: string[] = []
  for (const fact of facts) {
    const isDuplicate = unique.some(
      (existing) =>
        existing.toLowerCase().includes(fact.toLowerCase().slice(0, 30)) ||
        fact.toLowerCase().includes(existing.toLowerCase().slice(0, 30))
    )
    if (!isDuplicate) unique.push(fact)
  }
  return unique
}
```
