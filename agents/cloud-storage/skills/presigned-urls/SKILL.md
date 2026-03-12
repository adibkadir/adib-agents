---
name: presigned-urls
description: Presigned URL patterns for Cloudflare R2 — upload (PUT) and download (GET) URL generation with proper signature configuration, client-side upload flow, and URL caching. Use when implementing direct-to-R2 uploads from the browser or generating temporary download links for private assets.
---

# Presigned URLs

Presigned URLs let the browser upload directly to R2 without proxying through your server. This reduces server bandwidth and scales better for large files.

## Generate Presigned Upload URL (PUT)

```typescript
// lib/r2.ts
import { PutObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3'
import { getSignedUrl } from '@aws-sdk/s3-request-presigner'
import { r2Client, R2_BUCKET } from './r2'

export async function getSignedUploadUrl(
  key: string,
  contentType: string,
  expiresIn: number = 600 // 10 minutes
): Promise<string> {
  const command = new PutObjectCommand({
    Bucket: R2_BUCKET,
    Key: key,
    ContentType: contentType,
  })

  return getSignedUrl(r2Client, command, {
    expiresIn,
    unhoistableHeaders: new Set(['content-type']),
  })
}
```

### Why `unhoistableHeaders`?

By default, the S3 presigner "hoists" headers into query parameters. R2 requires `Content-Type` to stay as a header in the actual PUT request. Without `unhoistableHeaders: new Set(['content-type'])`, the signature will mismatch when the browser sends `Content-Type` as a header.

## Generate Presigned Download URL (GET)

```typescript
export async function getSignedDownloadUrl(
  key: string,
  expiresIn: number = 3600 // 1 hour
): Promise<string> {
  const command = new GetObjectCommand({
    Bucket: R2_BUCKET,
    Key: key,
  })

  return getSignedUrl(r2Client, command, { expiresIn })
}
```

## API Route: Issue Presigned URLs

```typescript
// app/api/media/presign/route.ts
import { auth } from '@/lib/auth'
import { headers } from 'next/headers'
import { db } from '@/db'
import { media } from '@/db/schema'
import { nanoid } from 'nanoid'
import { getSignedUploadUrl, getSignedDownloadUrl } from '@/lib/r2'

const ALLOWED_TYPES = ['image/jpeg', 'image/png', 'image/gif', 'image/webp']
const MAX_SIZE = 10 * 1024 * 1024 // 10MB

export async function POST(request: Request) {
  const session = await auth.api.getSession({ headers: await headers() })
  if (!session) {
    return Response.json({ error: 'Unauthorized' }, { status: 401 })
  }

  const { entityId, filename, contentType, size } = await request.json()

  // Validate type
  if (!ALLOWED_TYPES.includes(contentType)) {
    return Response.json(
      { error: `Invalid file type. Allowed: ${ALLOWED_TYPES.join(', ')}` },
      { status: 400 }
    )
  }

  // Validate size
  if (size && size > MAX_SIZE) {
    return Response.json(
      { error: `File too large. Maximum: ${MAX_SIZE / 1024 / 1024}MB` },
      { status: 400 }
    )
  }

  // Generate key scoped by user and entity
  const mediaId = nanoid(21)
  const extension = filename.split('.').pop() || 'jpg'
  const key = `${session.user.id}/${entityId}/${mediaId}.${extension}`

  // Track in database
  const [record] = await db.insert(media).values({
    id: mediaId,
    userId: session.user.id,
    entityId,
    key,
    filename,
    contentType,
    size: size || null,
  }).returning()

  // Generate URLs
  const uploadUrl = await getSignedUploadUrl(key, contentType)
  const downloadUrl = await getSignedDownloadUrl(key)

  return Response.json({
    id: record.id,
    uploadUrl,
    downloadUrl,
    key,
  })
}
```

## Client-Side Upload Flow

```typescript
// lib/upload.ts (client-side)

interface UploadResult {
  id: string
  url: string
}

export async function uploadFile(
  file: File,
  entityId: string
): Promise<UploadResult> {
  // 1. Get presigned URL from server
  const presignRes = await fetch('/api/media/presign', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      entityId,
      filename: file.name,
      contentType: file.type,
      size: file.size,
    }),
  })

  if (!presignRes.ok) {
    const { error } = await presignRes.json()
    throw new Error(error || 'Failed to get upload URL')
  }

  const { id, uploadUrl, downloadUrl } = await presignRes.json()

  // 2. Upload directly to R2
  const uploadRes = await fetch(uploadUrl, {
    method: 'PUT',
    headers: { 'Content-Type': file.type },
    body: file,
  })

  if (!uploadRes.ok) {
    throw new Error('Failed to upload file to storage')
  }

  return { id, url: downloadUrl }
}
```

## Presigned Download URL Caching

For repeated access to the same media (e.g., rendering images in a list), cache presigned download URLs client-side:

```typescript
// lib/media-url-cache.ts (client-side)
const failedIds = new Set<string>()
const pending = new Map<string, Promise<string>>()

export async function getMediaUrl(mediaId: string): Promise<string> {
  if (failedIds.has(mediaId)) {
    throw new Error('Media not found (cached)')
  }

  // Deduplicate in-flight requests
  const inflight = pending.get(mediaId)
  if (inflight) return inflight

  const promise = (async () => {
    const res = await fetch(`/api/media/${mediaId}/url`)
    if (res.status === 404) {
      failedIds.add(mediaId)
      throw new Error('Media not found')
    }
    const { url } = await res.json()
    return url as string
  })()

  pending.set(mediaId, promise)

  try {
    return await promise
  } finally {
    pending.delete(mediaId)
  }
}
```

## When to Use Presigned vs Direct Upload

| Factor | Presigned URL | Direct Upload |
|--------|---------------|---------------|
| File size | Large (>5MB) | Small (<5MB) |
| Server bandwidth | Low (browser → R2 direct) | High (browser → server → R2) |
| Complexity | Higher (2-step flow) | Lower (single request) |
| Auth | API generates URL, browser uploads | Server handles everything |
| Progress tracking | Native with `XMLHttpRequest` | Requires streaming |
| Use case | User media, rich text editor | Admin CMS, avatars, AI images |
