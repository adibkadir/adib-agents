---
name: cache-components
description: Next.js cache components (PPR) with 'use cache' directive, cacheLife(), cacheTag(), and Suspense streaming patterns. Use when implementing caching strategy or optimizing page load performance.
---

# Cache Components (Partial Prerendering)

## Three-Tier Caching

### Tier 1: Long Cache (hours)

For content that changes rarely — blog posts, product pages, site settings:

```typescript
// lib/data.ts
import { cacheLife, cacheTag } from 'next/cache'

export async function getPost(slug: string) {
  'use cache'
  cacheLife('hours')     // ~1h stale, ~2h revalidate, ~1d expire
  cacheTag(`post-${slug}`)

  return db.query.posts.findFirst({
    where: eq(posts.slug, slug),
    with: { author: true },
  })
}
```

### Tier 2: Short Cache (minutes)

For content that changes moderately — comments, activity feeds:

```typescript
export async function getComments(postSlug: string) {
  'use cache'
  cacheLife('minutes')   // ~5min stale, ~1min revalidate
  cacheTag(`comments-${postSlug}`)

  return db.query.comments.findMany({
    where: eq(comments.postSlug, postSlug),
    orderBy: desc(comments.createdAt),
  })
}
```

### Tier 3: No Cache (always fresh)

For user-specific or real-time data — admin panels, dashboards:

```typescript
import { connection } from 'next/server'

export default async function AdminPage() {
  await connection() // Opt out of static/cached rendering

  const users = await db.query.users.findMany()
  return <UserTable users={users} />
}
```

## Page Composition Pattern

Combine static shell with cached and dynamic content:

```typescript
// app/posts/[slug]/page.tsx
import { Suspense } from 'react'
import { connection } from 'next/server'

// This function is cached
async function PostContent({ slug }: { slug: string }) {
  'use cache'
  cacheLife('hours')
  cacheTag(`post-${slug}`)

  const post = await getPost(slug)
  if (!post) notFound()
  return <Article post={post} />
}

// This function is cached with shorter TTL
async function Comments({ slug }: { slug: string }) {
  'use cache'
  cacheLife('minutes')
  cacheTag(`comments-${slug}`)

  const comments = await getComments(slug)
  return <CommentList comments={comments} />
}

// This function is always fresh
async function RelatedPosts({ slug }: { slug: string }) {
  await connection()
  const related = await getRelatedPosts(slug)
  return <RelatedList posts={related} />
}

// Page assembles all three tiers
export default async function PostPage({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params

  return (
    <div>
      {/* Cached content included in static shell */}
      <PostContent slug={slug} />

      {/* Cached with shorter TTL */}
      <Suspense fallback={<CommentsSkeleton />}>
        <Comments slug={slug} />
      </Suspense>

      {/* Always fresh, streams in after shell */}
      <Suspense fallback={<RelatedSkeleton />}>
        <RelatedPosts slug={slug} />
      </Suspense>
    </div>
  )
}
```

## Cache Invalidation

```typescript
// In Server Actions after mutations:
'use server'

import { revalidateTag } from 'next/cache'

export async function updatePost(slug: string, data: PostUpdate) {
  await db.update(posts).set(data).where(eq(posts.slug, slug))

  // Invalidate specific caches
  revalidateTag(`post-${slug}`)      // This specific post
  revalidateTag('posts')              // Post listings
  revalidateTag(`comments-${slug}`)   // This post's comments
}
```

## Custom Cache Profiles

```typescript
// next.config.ts
const nextConfig = {
  experimental: {
    cacheLife: {
      // Custom profiles
      'blog-post': {
        stale: 3600,      // 1 hour
        revalidate: 900,   // 15 minutes
        expire: 86400,     // 1 day
      },
      'user-data': {
        stale: 60,         // 1 minute
        revalidate: 30,    // 30 seconds
        expire: 300,       // 5 minutes
      },
    },
  },
}
```

## Decision Guide

| Data Type | Cache Strategy | Why |
|-----------|---------------|-----|
| Landing page content | Static (build time) | Never changes per-request |
| Blog post body | `cacheLife('hours')` | Changes rarely, invalidated on edit |
| Comments | `cacheLife('minutes')` | Changes moderately, users expect freshness |
| User session data | No cache (`connection()`) | Must be per-request |
| Admin CRUD pages | No cache (`connection()`) | Always needs latest data |
| API webhook endpoints | No cache (Route Handler) | Must process every request |
