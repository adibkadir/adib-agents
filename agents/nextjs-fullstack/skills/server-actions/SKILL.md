---
name: server-actions
description: Next.js Server Actions for data mutations, form handling, optimistic updates, and cache revalidation. Use when implementing create/update/delete operations or form submissions.
---

# Server Actions

Server Actions are async functions that run on the server. They replace API routes for mutations.

## Basic Pattern

```typescript
// app/actions/posts.ts
'use server'

import { revalidateTag } from 'next/cache'
import { redirect } from 'next/navigation'
import { db } from '@/lib/db'
import { posts } from '@/db/schema'
import { z } from 'zod'

const CreatePostSchema = z.object({
  title: z.string().min(1).max(200),
  content: z.string().min(1),
  slug: z.string().regex(/^[a-z0-9-]+$/),
})

export async function createPost(formData: FormData) {
  const parsed = CreatePostSchema.safeParse({
    title: formData.get('title'),
    content: formData.get('content'),
    slug: formData.get('slug'),
  })

  if (!parsed.success) {
    return { error: parsed.error.flatten().fieldErrors }
  }

  const [post] = await db.insert(posts).values(parsed.data).returning()

  revalidateTag('posts')
  redirect(`/posts/${post.slug}`)
}
```

## Form Integration

```typescript
// components/PostForm.tsx
'use client'

import { useActionState } from 'react'
import { createPost } from '@/app/actions/posts'

export function PostForm() {
  const [state, formAction, isPending] = useActionState(createPost, null)

  return (
    <form action={formAction}>
      <input name="title" required />
      {state?.error?.title && <p className="text-red-500">{state.error.title}</p>}

      <textarea name="content" required />
      <input name="slug" required />

      <button type="submit" disabled={isPending}>
        {isPending ? 'Creating...' : 'Create Post'}
      </button>
    </form>
  )
}
```

## Optimistic Updates

```typescript
'use client'

import { useOptimistic } from 'react'
import { toggleFavorite } from '@/app/actions/favorites'

export function FavoriteButton({ isFavorited, sailingId }: Props) {
  const [optimisticFavorited, setOptimisticFavorited] = useOptimistic(isFavorited)

  async function handleToggle() {
    setOptimisticFavorited(!optimisticFavorited)
    await toggleFavorite(sailingId)
  }

  return (
    <form action={handleToggle}>
      <button>{optimisticFavorited ? '★' : '☆'}</button>
    </form>
  )
}
```

## Auth-Protected Actions

```typescript
'use server'

import { getSession } from '@/lib/auth'

export async function updateProfile(formData: FormData) {
  const session = await getSession()
  if (!session) throw new Error('Unauthorized')

  // Only allow users to update their own profile
  const userId = session.user.id

  await db.update(users)
    .set({ name: formData.get('name') as string })
    .where(eq(users.id, userId))

  revalidateTag(`user-${userId}`)
  return { success: true }
}
```

## Cache Revalidation

```typescript
'use server'

import { revalidateTag, revalidatePath } from 'next/cache'

// Revalidate by tag (preferred - surgical)
revalidateTag('posts')
revalidateTag(`post-${slug}`)
revalidateTag(`comments-${slug}`)

// Revalidate by path (broader)
revalidatePath('/admin/posts')
revalidatePath('/posts/[slug]', 'page')
```

## File Upload Action

```typescript
'use server'

import { PutObjectCommand, S3Client } from '@aws-sdk/client-s3'
import { getSignedUrl } from '@aws-sdk/s3-request-presigner'

export async function getPresignedUploadUrl(filename: string, contentType: string) {
  const session = await getSession()
  if (!session) throw new Error('Unauthorized')

  const key = `uploads/${session.user.id}/${Date.now()}-${filename}`

  const command = new PutObjectCommand({
    Bucket: process.env.R2_BUCKET,
    Key: key,
    ContentType: contentType,
  })

  const url = await getSignedUrl(s3Client, command, { expiresIn: 300 })
  return { url, key }
}
```
