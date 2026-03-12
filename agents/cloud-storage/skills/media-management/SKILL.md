---
name: media-management
description: Database-tracked media management with Cloudflare R2 — Drizzle ORM schema for media records, cascade deletion, orphaned media cleanup cron, download buffer helper, and batch operations. Use when building a media library or when uploads need tracking, cleanup, and lifecycle management.
---

# Media Management

## Media Database Schema

Track every R2 upload in the database for auditing, cleanup, and URL regeneration:

```typescript
// db/schema.ts
import { pgTable, text, integer, timestamp } from 'drizzle-orm/pg-core'
import { sql } from 'drizzle-orm'
import { nanoid } from 'nanoid'

export const media = pgTable('media', {
  id: text('id').primaryKey().$defaultFn(() => nanoid(21)),
  userId: text('user_id').notNull()
    .references(() => user.id, { onDelete: 'cascade' }),
  entityId: text('entity_id').notNull(),  // slide, post, etc.
  key: text('key').notNull(),             // R2 object key
  filename: text('filename').notNull(),   // original filename
  contentType: text('content_type').notNull(),
  size: integer('size'),                  // bytes
  mediaType: text('media_type').notNull().default('image'),
  width: integer('width'),
  height: integer('height'),
  createdAt: timestamp('created_at').default(sql`CURRENT_TIMESTAMP`).notNull(),
})
```

## Download Buffer Helper

For server-side processing of stored files:

```typescript
// lib/r2.ts
import { GetObjectCommand } from '@aws-sdk/client-s3'
import { r2Client, R2_BUCKET } from './r2'

export async function downloadBuffer(key: string): Promise<Buffer> {
  const command = new GetObjectCommand({
    Bucket: R2_BUCKET,
    Key: key,
  })

  const response = await r2Client.send(command)
  const bytes = await response.Body!.transformToByteArray()
  return Buffer.from(bytes)
}
```

## Cascade Delete on Entity Deletion

When deleting an entity (slide, post, etc.), delete its R2 objects too:

```typescript
// app/actions.ts
'use server'

import { db } from '@/db'
import { media } from '@/db/schema'
import { eq, and } from 'drizzle-orm'
import { deleteObjects } from '@/lib/r2'

export async function deleteEntity(entityId: string) {
  const session = await auth.api.getSession({ headers: await headers() })
  if (!session) throw new Error('Unauthorized')

  // 1. Get all media keys for this entity
  const entityMedia = await db.select({ key: media.key })
    .from(media)
    .where(and(
      eq(media.entityId, entityId),
      eq(media.userId, session.user.id)
    ))

  // 2. Delete from R2
  if (entityMedia.length > 0) {
    await deleteObjects(entityMedia.map(m => m.key))
  }

  // 3. Delete from database (cascade will also remove media records if FK set)
  await db.delete(entities)
    .where(and(
      eq(entities.id, entityId),
      eq(entities.userId, session.user.id)
    ))
    .returning()
}
```

## Orphaned Media Cleanup Cron

Finds and deletes media that is no longer referenced by any entity content:

```typescript
// app/api/cron/cleanup-media/route.ts
import { db } from '@/db'
import { media, entities } from '@/db/schema'
import { eq, inArray, isNull } from 'drizzle-orm'
import { deleteObjects } from '@/lib/r2'

const CRON_SECRET = process.env.CRON_SECRET

export async function POST(request: Request) {
  // Auth: verify cron secret
  const authHeader = request.headers.get('authorization')
  const url = new URL(request.url)
  const queryKey = url.searchParams.get('key')
  const providedSecret = authHeader?.replace('Bearer ', '') || queryKey

  if (!CRON_SECRET || providedSecret !== CRON_SECRET) {
    return Response.json({ error: 'Unauthorized' }, { status: 401 })
  }

  // 1. Find media whose entity no longer exists
  const allMedia = await db.select({
    id: media.id,
    key: media.key,
    entityId: media.entityId,
  }).from(media)

  const entityIds = [...new Set(allMedia.map(m => m.entityId))]

  const existingEntities = await db.select({ id: entities.id })
    .from(entities)
    .where(inArray(entities.id, entityIds))

  const existingIds = new Set(existingEntities.map(e => e.id))

  // 2. Find media referencing non-existent entities
  const orphanedByEntity = allMedia.filter(m => !existingIds.has(m.entityId))

  // 3. Find media not referenced in entity content (optional — content-aware check)
  const orphanedByContent = await findContentOrphans(allMedia, existingIds)

  const allOrphaned = [...orphanedByEntity, ...orphanedByContent]
  const uniqueOrphaned = [...new Map(allOrphaned.map(m => [m.id, m])).values()]

  if (uniqueOrphaned.length === 0) {
    return Response.json({ deleted: 0, message: 'No orphaned media found' })
  }

  // 4. Delete from R2
  await deleteObjects(uniqueOrphaned.map(m => m.key))

  // 5. Delete from database
  await db.delete(media)
    .where(inArray(media.id, uniqueOrphaned.map(m => m.id)))

  return Response.json({
    deleted: uniqueOrphaned.length,
    keys: uniqueOrphaned.map(m => m.key),
  })
}

async function findContentOrphans(
  allMedia: { id: string; key: string; entityId: string }[],
  existingEntityIds: Set<string>
) {
  // Get entities with their content to check for media references
  const entitiesWithContent = await db.select({
    id: entities.id,
    content: entities.content,  // JSON content field
  }).from(entities)
    .where(inArray(entities.id, [...existingEntityIds]))

  // Extract media IDs referenced in content
  const referencedIds = new Set<string>()
  for (const entity of entitiesWithContent) {
    if (entity.content) {
      extractMediaIds(entity.content, referencedIds)
    }
  }

  // Media that exists in DB but not referenced in any content
  return allMedia.filter(m =>
    existingEntityIds.has(m.entityId) && !referencedIds.has(m.id)
  )
}

function extractMediaIds(content: unknown, ids: Set<string>): void {
  if (!content || typeof content !== 'object') return

  const obj = content as Record<string, unknown>
  const attrs = obj.attrs as Record<string, unknown> | undefined

  if (attrs?.['data-image-id']) ids.add(attrs['data-image-id'] as string)
  if (attrs?.['data-video-id']) ids.add(attrs['data-video-id'] as string)

  const children = obj.content as unknown[] | undefined
  if (Array.isArray(children)) {
    for (const child of children) {
      extractMediaIds(child, ids)
    }
  }
}
```

## Scheduling the Cleanup Cron

### With pg-boss (recommended)

```typescript
// worker/index.ts
await boss.schedule('media-cleanup', '0 * * * *', {})  // hourly

await boss.work('media-cleanup', async () => {
  const res = await fetch(`${process.env.APP_URL}/api/cron/cleanup-media`, {
    method: 'POST',
    headers: { 'Authorization': `Bearer ${process.env.CRON_SECRET}` },
  })
  const data = await res.json()
  logger.info('Media cleanup', data)
})
```

### With External Cron (Vercel, Railway)

Configure external cron to POST to `/api/cron/cleanup-media?key={CRON_SECRET}` on schedule.

## Media URL Endpoint

For private assets that need presigned download URLs on demand:

```typescript
// app/api/media/[id]/url/route.ts
import { db } from '@/db'
import { media } from '@/db/schema'
import { eq } from 'drizzle-orm'
import { getSignedDownloadUrl } from '@/lib/r2'

export async function GET(
  _request: Request,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params
  const session = await auth.api.getSession({ headers: await headers() })
  if (!session) return Response.json({ error: 'Unauthorized' }, { status: 401 })

  const [record] = await db.select()
    .from(media)
    .where(eq(media.id, id))
    .limit(1)

  if (!record) {
    return Response.json({ error: 'Not found' }, { status: 404 })
  }

  // Verify ownership
  if (record.userId !== session.user.id) {
    return Response.json({ error: 'Forbidden' }, { status: 403 })
  }

  const url = await getSignedDownloadUrl(record.key)
  return Response.json({ url })
}
```
