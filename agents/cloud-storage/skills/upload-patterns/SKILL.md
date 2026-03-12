---
name: upload-patterns
description: Direct upload patterns for Cloudflare R2 — server actions, API routes, buffer uploads, base64 conversion, avatar handling, URL-fetch-and-store, and file validation. Use when implementing file uploads that go through your server rather than presigned URLs.
---

# Upload Patterns

## Core Upload Function

```typescript
// lib/r2.ts
import { PutObjectCommand, DeleteObjectCommand } from '@aws-sdk/client-s3'
import { r2Client, R2_BUCKET, R2_PUBLIC_URL } from './r2'

export async function uploadToR2(
  key: string,
  body: Buffer | Uint8Array,
  contentType: string
): Promise<string> {
  await r2Client.send(
    new PutObjectCommand({
      Bucket: R2_BUCKET,
      Key: key,
      Body: body,
      ContentType: contentType,
    })
  )
  return `${R2_PUBLIC_URL}/${key}`
}

export async function uploadBuffer(
  key: string,
  buffer: Buffer,
  contentType: string
): Promise<void> {
  await r2Client.send(
    new PutObjectCommand({
      Bucket: R2_BUCKET,
      Key: key,
      Body: buffer,
      ContentType: contentType,
    })
  )
}

export async function deleteFromR2(key: string): Promise<void> {
  await r2Client.send(
    new DeleteObjectCommand({
      Bucket: R2_BUCKET,
      Key: key,
    })
  )
}

export async function deleteObjects(keys: string[]): Promise<void> {
  await Promise.all(keys.map(key => deleteFromR2(key)))
}
```

## Pattern 1: Server Action Upload (Blog Images)

```typescript
// app/actions.ts
'use server'

import { uploadToR2, deleteFromR2 } from '@/lib/r2'
import { R2_PUBLIC_URL } from '@/lib/r2'

export async function uploadImage(formData: FormData) {
  const session = await auth.api.getSession({ headers: await headers() })
  if (!session || session.user.role !== 'admin') {
    throw new Error('Unauthorized')
  }

  const file = formData.get('file') as File
  if (!file) throw new Error('No file uploaded')

  const ALLOWED = ['image/jpeg', 'image/png', 'image/gif', 'image/webp']
  if (!ALLOWED.includes(file.type)) {
    throw new Error('Invalid file type')
  }

  const buffer = Buffer.from(await file.arrayBuffer())
  const filename = `${Date.now()}-${file.name.replace(/\s/g, '-')}`

  const url = await uploadToR2(filename, buffer, file.type)
  return { success: true, url }
}

export async function deleteImage(imageUrl: string) {
  const session = await auth.api.getSession({ headers: await headers() })
  if (!session || session.user.role !== 'admin') {
    throw new Error('Unauthorized')
  }

  if (!imageUrl.startsWith(R2_PUBLIC_URL)) {
    return { success: false, error: 'Invalid image URL' }
  }

  const key = imageUrl.replace(`${R2_PUBLIC_URL}/`, '')
  await deleteFromR2(key)
  return { success: true }
}
```

## Pattern 2: API Route Upload (Avatars)

```typescript
// app/api/avatar/route.ts
import { uploadToR2 } from '@/lib/r2'
import { auth } from '@/lib/auth'
import { headers } from 'next/headers'

const ALLOWED_TYPES = new Set(['image/jpeg', 'image/png', 'image/webp', 'image/gif'])
const MAX_SIZE = 2 * 1024 * 1024 // 2MB
const EXT_MAP: Record<string, string> = {
  'image/jpeg': 'jpg',
  'image/png': 'png',
  'image/webp': 'webp',
  'image/gif': 'gif',
}

export async function POST(request: Request) {
  const session = await auth.api.getSession({ headers: await headers() })
  if (!session) return Response.json({ error: 'Unauthorized' }, { status: 401 })

  const formData = await request.formData()
  const file = formData.get('file') as File | null
  if (!file) return Response.json({ error: 'No file' }, { status: 400 })

  if (!ALLOWED_TYPES.has(file.type)) {
    return Response.json({ error: 'Invalid type' }, { status: 400 })
  }

  if (file.size > MAX_SIZE) {
    return Response.json({ error: 'File too large (max 2MB)' }, { status: 400 })
  }

  const ext = EXT_MAP[file.type] ?? 'jpg'
  const key = `avatars/${session.user.id}.${ext}`

  const buffer = Buffer.from(await file.arrayBuffer())
  const url = await uploadToR2(key, buffer, file.type)

  // Update user record with avatar URL
  await db.update(user)
    .set({ image: url })
    .where(eq(user.id, session.user.id))

  return Response.json({ url })
}
```

## Pattern 3: API Route with Dual Input (File or URL)

```typescript
// app/api/upload/route.ts
import { uploadToR2 } from '@/lib/r2'

export async function POST(request: Request) {
  const session = await auth.api.getSession({ headers: await headers() })
  if (!session || session.user.role !== 'admin') {
    return Response.json({ error: 'Unauthorized' }, { status: 401 })
  }

  const contentType = request.headers.get('content-type') ?? ''

  // JSON body = upload from external URL
  if (contentType.includes('application/json')) {
    const { sourceUrl, key } = await request.json()
    const res = await fetch(sourceUrl, { signal: AbortSignal.timeout(15_000) })
    if (!res.ok) return Response.json({ error: 'Fetch failed' }, { status: 502 })

    const buffer = Buffer.from(await res.arrayBuffer())
    const mime = res.headers.get('content-type') ?? 'image/jpeg'
    const url = await uploadToR2(key, buffer, mime)
    return Response.json({ url })
  }

  // FormData = direct file upload
  const formData = await request.formData()
  const file = formData.get('file') as File | null
  const key = formData.get('key') as string | null

  if (!file || !key) {
    return Response.json({ error: 'Missing file or key' }, { status: 400 })
  }

  const buffer = Buffer.from(await file.arrayBuffer())
  const url = await uploadToR2(key, buffer, file.type)
  return Response.json({ url })
}
```

## Pattern 4: Base64 Buffer Upload (AI-Generated Images)

```typescript
// app/actions/ai.ts
'use server'

import { uploadToR2 } from '@/lib/r2'

export async function uploadGeneratedImage(
  base64: string,
  mimeType: string,
  prefix: string = 'ai-generated'
): Promise<{ success: boolean; url?: string; error?: string }> {
  const session = await auth.api.getSession({ headers: await headers() })
  if (!session || session.user.role !== 'admin') {
    return { success: false, error: 'Unauthorized' }
  }

  const buffer = Buffer.from(base64, 'base64')
  const extension = mimeType.split('/')[1] || 'png'
  const filename = `${prefix}-${Date.now()}.${extension}`

  const url = await uploadToR2(filename, buffer, mimeType)
  return { success: true, url }
}
```

## Pattern 5: Worker Upload (Export Files)

Used by background workers to store generated files:

```typescript
// worker/jobs/export-handler.ts
import { uploadBuffer } from '@/lib/r2'

export async function handleExportJob(data: ExportJobInput) {
  // Generate file in worker
  const pptxBuffer = await generatePptx(data.slides)

  // Upload to R2
  const key = `exports/${data.userId}/${data.jobId}.pptx`
  await uploadBuffer(
    key,
    pptxBuffer,
    'application/vnd.openxmlformats-officedocument.presentationml.presentation'
  )

  // Update job record with output path
  await db.update(exportJobs)
    .set({ r2Key: key, status: 'completed' })
    .where(eq(exportJobs.id, data.jobId))
}
```

## File Validation Helper

```typescript
// lib/upload-validation.ts

interface ValidationResult {
  valid: boolean
  error?: string
}

export function validateImageFile(
  file: File,
  maxSize: number = 10 * 1024 * 1024
): ValidationResult {
  const ALLOWED = ['image/jpeg', 'image/png', 'image/gif', 'image/webp']

  if (!ALLOWED.includes(file.type)) {
    return { valid: false, error: `Invalid type. Allowed: ${ALLOWED.join(', ')}` }
  }

  if (file.size > maxSize) {
    return { valid: false, error: `Too large. Max: ${maxSize / 1024 / 1024}MB` }
  }

  return { valid: true }
}
```
