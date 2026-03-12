---
name: r2-client-setup
description: Cloudflare R2 S3 client configuration with AWS SDK — endpoint, credentials, forcePathStyle, checksum settings, singleton pattern for Next.js, and environment variable setup. Use when initializing R2 storage in a new project or debugging client configuration issues.
---

# R2 Client Setup

## Dependencies

```bash
pnpm add @aws-sdk/client-s3 @aws-sdk/s3-request-presigner
```

## Standard Client Configuration

```typescript
// lib/r2.ts
import {
  S3Client,
  PutObjectCommand,
  GetObjectCommand,
  DeleteObjectCommand,
} from '@aws-sdk/client-s3'

export const r2Client = new S3Client({
  region: 'auto',
  endpoint: process.env.R2_ENDPOINT!,
  credentials: {
    accessKeyId: process.env.R2_ACCESS_KEY_ID!,
    secretAccessKey: process.env.R2_SECRET_ACCESS_KEY!,
  },
  forcePathStyle: true,
  requestChecksumCalculation: 'WHEN_REQUIRED',
  responseChecksumValidation: 'WHEN_REQUIRED',
})

export const R2_BUCKET = process.env.R2_BUCKET_NAME!
export const R2_PUBLIC_URL = process.env.R2_PUBLIC_URL!
```

### Why Each Option Matters

| Option | Value | Why |
|--------|-------|-----|
| `region` | `'auto'` | R2 auto-routes to nearest region |
| `endpoint` | `https://{accountId}.r2.cloudflarestorage.com` | R2's S3-compatible endpoint |
| `forcePathStyle` | `true` | R2 doesn't support virtual-hosted-style (`bucket.endpoint`). Without this, you get signature mismatch errors |
| `requestChecksumCalculation` | `'WHEN_REQUIRED'` | Newer AWS SDK versions (v3.400+) default to always computing checksums, which can cause R2 signature mismatches |
| `responseChecksumValidation` | `'WHEN_REQUIRED'` | Same reason — prevents validation errors on download |

## Singleton Pattern (for Next.js)

Next.js hot-reloads modules in dev, which would create multiple S3Client instances. Use a singleton:

```typescript
// lib/r2.ts — singleton variant
import { S3Client } from '@aws-sdk/client-s3'

let _r2: S3Client | null = null

export function getR2(): S3Client {
  if (!_r2) {
    _r2 = new S3Client({
      region: 'auto',
      endpoint: process.env.R2_ENDPOINT!,
      credentials: {
        accessKeyId: process.env.R2_ACCESS_KEY_ID!,
        secretAccessKey: process.env.R2_SECRET_ACCESS_KEY!,
      },
      forcePathStyle: true,
      requestChecksumCalculation: 'WHEN_REQUIRED',
      responseChecksumValidation: 'WHEN_REQUIRED',
    })
  }
  return _r2
}
```

Use `getR2()` for hot-reload safety. Use the direct `r2Client` export when the module is only loaded once (worker processes, scripts).

## Alternative: Construct Endpoint from Account ID

```typescript
const R2_ACCOUNT_ID = process.env.R2_ACCOUNT_ID!

export const r2Client = new S3Client({
  region: 'auto',
  endpoint: `https://${R2_ACCOUNT_ID}.r2.cloudflarestorage.com`,
  credentials: {
    accessKeyId: process.env.R2_ACCESS_KEY_ID!,
    secretAccessKey: process.env.R2_SECRET_ACCESS_KEY!,
  },
  forcePathStyle: true,
})
```

## Environment Variables

```bash
# .env.example
R2_ENDPOINT=https://<account-id>.r2.cloudflarestorage.com
R2_ACCESS_KEY_ID=
R2_SECRET_ACCESS_KEY=
R2_BUCKET_NAME=my-bucket
R2_PUBLIC_URL=https://assets.example.com

# Optional
R2_ACCOUNT_ID=               # If constructing endpoint dynamically
NEXT_PUBLIC_R2_PUBLIC_URL=    # For client-side URL construction
```

### Getting R2 Credentials

1. Go to Cloudflare Dashboard → R2 → Manage R2 API Tokens
2. Create a new API token with "Object Read & Write" permissions
3. Copy the Access Key ID and Secret Access Key
4. The endpoint is `https://{account-id}.r2.cloudflarestorage.com`
5. For public access, set up a custom domain in the R2 bucket settings → Public Access

## Helper Functions

```typescript
// lib/r2.ts — common helpers

export function r2PublicUrl(key: string): string {
  return `${R2_PUBLIC_URL}/${key}`
}

export function r2KeyFromUrl(url: string): string | null {
  if (!url.startsWith(R2_PUBLIC_URL)) return null
  return url.replace(`${R2_PUBLIC_URL}/`, '')
}
```

## Next.js Config

Add R2 public domain to `next/image` allowed patterns:

```typescript
// next.config.ts
const nextConfig: NextConfig = {
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 'assets.example.com', // your R2_PUBLIC_URL domain
      },
    ],
  },
}
```

## Server Action Body Size

If uploading via server actions, increase the body size limit:

```typescript
// next.config.ts
const nextConfig: NextConfig = {
  serverActions: {
    bodySizeLimit: '10mb',
  },
}
```
