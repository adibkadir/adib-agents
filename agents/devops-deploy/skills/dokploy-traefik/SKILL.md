---
name: dokploy-traefik
description: Dokploy deployment with Traefik reverse proxy — docker-compose labels for routing, wildcard TLS with Let's Encrypt DNS challenge, subdomain routing with HostRegexp, HTTP-to-HTTPS redirect, service networking, and environment variable management. Use when deploying to Dokploy or configuring Traefik routing for multi-tenant apps.
---

# Dokploy + Traefik Deployment

## docker-compose.yml with Traefik Labels

```yaml
services:
  # Init container: run migrations then exit
  db-push:
    build:
      context: .
      dockerfile: packages/shared/Dockerfile.db-push
    env_file:
      - .env
    networks:
      - dokploy-network

  # Backend service (Motia workflow engine or similar)
  backend:
    build:
      context: .
      dockerfile: backend/Dockerfile
    restart: unless-stopped
    expose:
      - "3111"
    depends_on:
      db-push:
        condition: service_completed_successfully
    env_file:
      - .env
    environment:
      - NODE_ENV=production
      - PORT=3111
      - FRONTEND_URL=http://frontend:3001
    networks:
      - dokploy-network

  # Frontend (Next.js + optional worker)
  frontend:
    build:
      context: .
      dockerfile: frontend/Dockerfile
      args:
        NEXT_PUBLIC_ROOT_DOMAIN: ${NEXT_PUBLIC_ROOT_DOMAIN:-example.com}
        NEXT_PUBLIC_R2_PUBLIC_URL: ${NEXT_PUBLIC_R2_PUBLIC_URL:-}
    restart: unless-stopped
    expose:
      - "3001"
    depends_on:
      db-push:
        condition: service_completed_successfully
      backend:
        condition: service_started
    env_file:
      - .env
    environment:
      - NODE_ENV=production
      - PORT=3001
      - BACKEND_URL=http://backend:3111

    labels:
      - traefik.enable=true
      - traefik.docker.network=dokploy-network

      # === Wildcard subdomain routing ===

      # HTTP → HTTPS redirect (wildcard)
      - traefik.http.routers.myapp-wildcard-web.rule=HostRegexp(`^[a-z0-9-]+\.example\.com$$`)
      - traefik.http.routers.myapp-wildcard-web.entrypoints=web
      - traefik.http.routers.myapp-wildcard-web.priority=10
      - traefik.http.routers.myapp-wildcard-web.middlewares=redirect-to-https@file

      # HTTPS (wildcard) with DNS challenge cert
      - traefik.http.routers.myapp-wildcard-secure.rule=HostRegexp(`^[a-z0-9-]+\.example\.com$$`)
      - traefik.http.routers.myapp-wildcard-secure.entrypoints=websecure
      - traefik.http.routers.myapp-wildcard-secure.priority=10
      - traefik.http.routers.myapp-wildcard-secure.tls.certresolver=letsencrypt-dns
      - traefik.http.routers.myapp-wildcard-secure.tls.domains[0].main=example.com
      - traefik.http.routers.myapp-wildcard-secure.tls.domains[0].sans=*.example.com

      # === Apex domain routing ===

      # HTTP → HTTPS redirect (apex)
      - traefik.http.routers.myapp-apex-web.rule=Host(`example.com`)
      - traefik.http.routers.myapp-apex-web.entrypoints=web
      - traefik.http.routers.myapp-apex-web.middlewares=redirect-to-https@file

      # HTTPS (apex) with HTTP challenge cert
      - traefik.http.routers.myapp-apex-secure.rule=Host(`example.com`)
      - traefik.http.routers.myapp-apex-secure.entrypoints=websecure
      - traefik.http.routers.myapp-apex-secure.tls.certresolver=letsencrypt

      # === Service port ===
      - traefik.http.services.myapp-frontend.loadbalancer.server.port=3001

networks:
  dokploy-network:
    external: true
```

## Key Traefik Concepts

### Certificate Resolvers

| Resolver | Challenge Type | Use Case | Supports Wildcard |
|----------|---------------|----------|-------------------|
| `letsencrypt` | HTTP-01 | Apex domain only | No |
| `letsencrypt-dns` | DNS-01 | Wildcard subdomains | Yes |

DNS-01 challenge requires Cloudflare API credentials configured in Traefik's static config (managed by Dokploy).

### HostRegexp vs Host

```yaml
# Matches any subdomain: app.example.com, tenant1.example.com
HostRegexp(`^[a-z0-9-]+\.example\.com$$`)

# Matches exact domain: example.com
Host(`example.com`)
```

Note the `$$` in `HostRegexp` — in YAML, `$` must be escaped as `$$` because Dokploy uses Docker Compose variable interpolation.

### Priority

Lower priority number = matches first. Set wildcard routes to `priority=10` so they don't override more specific routes.

### Entrypoints

| Name | Port | Protocol |
|------|------|----------|
| `web` | 80 | HTTP |
| `websecure` | 443 | HTTPS |

### HTTP → HTTPS Redirect

The `redirect-to-https@file` middleware is configured in Traefik's file provider (managed by Dokploy). It redirects all HTTP traffic to HTTPS.

## Single-Domain Setup (No Subdomains)

For simpler apps without multi-tenant subdomains:

```yaml
labels:
  - traefik.enable=true
  - traefik.docker.network=dokploy-network

  # HTTP → HTTPS
  - traefik.http.routers.myapp-web.rule=Host(`myapp.com`)
  - traefik.http.routers.myapp-web.entrypoints=web
  - traefik.http.routers.myapp-web.middlewares=redirect-to-https@file

  # HTTPS
  - traefik.http.routers.myapp-secure.rule=Host(`myapp.com`)
  - traefik.http.routers.myapp-secure.entrypoints=websecure
  - traefik.http.routers.myapp-secure.tls.certresolver=letsencrypt

  # Service
  - traefik.http.services.myapp.loadbalancer.server.port=3000
```

## Dokploy Deployment Steps

1. **Create Docker Compose application** in Dokploy dashboard
2. **Connect GitHub repo** and set Compose Path to `./docker-compose.yml`
3. **Configure domains** in the Domains tab (assign to the frontend service)
4. **Set environment variables** in the Environment tab — Dokploy writes these to `.env` which is loaded via `env_file`
5. **Deploy** — Dokploy pulls, builds, and starts all services

## Internal Service Communication

```yaml
# Frontend calls backend internally via Docker DNS
environment:
  - BACKEND_URL=http://backend:3111

# Backend calls frontend via public URL (for webhooks/callbacks)
environment:
  - FRONTEND_URL=https://example.com
```

Why the asymmetry: Frontend → Backend is internal (same Docker network). Backend → Frontend must go through Traefik for proper routing (especially with multi-tenant subdomains).

## Migration Init Container

A lightweight container that runs migrations and exits:

```dockerfile
# packages/shared/Dockerfile.db-push
FROM node:22-alpine
RUN corepack enable && corepack prepare pnpm@10.4.1 --activate
WORKDIR /app

COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc ./
COPY packages/shared/package.json packages/shared/
RUN pnpm install --frozen-lockfile --filter @myapp/shared

COPY packages/shared packages/shared
WORKDIR /app/packages/shared
CMD ["pnpm", "exec", "drizzle-kit", "push", "--force"]
```

The `--force` flag skips safety prompts in automated environments. Combined with `depends_on: service_completed_successfully`, this guarantees migrations run before the app starts.

## Required DNS Records

```
# Apex domain
A     example.com     → <server-ip>

# Wildcard subdomains
A     *.example.com   → <server-ip>

# CDN for R2 (CNAME to R2 public bucket)
CNAME assets.example.com → <bucket>.r2.dev
```
