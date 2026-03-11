---
name: nextjs-app-router
description: Next.js 16 App Router patterns including route groups, layouts, parallel routes, intercepting routes, and metadata. Use when building page structure, routing, or navigation in a Next.js app.
---

# Next.js 16 App Router

## Route Groups

Use parentheses to organize routes without affecting the URL:

```
app/
├── (marketing)/          # URL: /
│   ├── layout.tsx        # Marketing layout (no sidebar)
│   └── page.tsx
├── (app)/                # URL: /dashboard, /settings
│   ├── layout.tsx        # App layout (sidebar + nav)
│   ├── dashboard/
│   └── settings/
└── (admin)/              # URL: /admin/*
    ├── layout.tsx        # Admin layout (admin nav)
    └── users/
```

## Layouts

Layouts wrap child routes and persist across navigation:

```typescript
// app/(app)/layout.tsx
import { getSession } from '@/lib/auth'
import { redirect } from 'next/navigation'

export default async function AppLayout({ children }: { children: React.ReactNode }) {
  const session = await getSession()
  if (!session) redirect('/login')

  return (
    <div className="flex min-h-screen">
      <Sidebar user={session.user} />
      <main className="flex-1 p-6">{children}</main>
    </div>
  )
}
```

## Dynamic Routes

```typescript
// app/posts/[slug]/page.tsx
export default async function PostPage({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params
  const post = await getPost(slug)
  if (!post) notFound()
  return <Article post={post} />
}

// Generate static params for SSG
export async function generateStaticParams() {
  const posts = await getAllPostSlugs()
  return posts.map(({ slug }) => ({ slug }))
}
```

## Metadata

```typescript
// app/posts/[slug]/page.tsx
import type { Metadata } from 'next'

export async function generateMetadata({ params }: { params: Promise<{ slug: string }> }): Promise<Metadata> {
  const { slug } = await params
  const post = await getPost(slug)
  return {
    title: post?.title,
    description: post?.excerpt,
    openGraph: { images: [post?.image ?? '/og-default.png'] },
  }
}
```

## Loading & Error States

```typescript
// app/posts/loading.tsx — Shows during page load
export default function Loading() {
  return <PostSkeleton />
}

// app/posts/error.tsx — Catches errors in the segment
'use client'
export default function Error({ error, reset }: { error: Error; reset: () => void }) {
  return (
    <div>
      <h2>Something went wrong</h2>
      <button onClick={reset}>Try again</button>
    </div>
  )
}

// app/posts/not-found.tsx — Custom 404 for the segment
export default function NotFound() {
  return <h2>Post not found</h2>
}
```

## Middleware

```typescript
// middleware.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  // Multi-tenant subdomain resolution
  const hostname = request.headers.get('host') ?? ''
  const subdomain = hostname.split('.')[0]

  if (subdomain !== 'www' && subdomain !== 'app') {
    request.headers.set('x-tenant', subdomain)
  }

  // Auth redirect for protected routes
  const session = request.cookies.get('session_token')
  if (request.nextUrl.pathname.startsWith('/app') && !session) {
    return NextResponse.redirect(new URL('/login', request.url))
  }

  return NextResponse.next()
}

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico).*)'],
}
```

## Linking and Navigation

```typescript
import Link from 'next/link'
import { useRouter, usePathname, useSearchParams } from 'next/navigation'

// Declarative navigation
<Link href="/posts/my-post" prefetch={true}>Read Post</Link>

// Programmatic navigation (Client Components only)
const router = useRouter()
router.push('/dashboard')
router.replace('/login')
router.refresh() // Re-fetch server components without full page reload
```
