# Agents Recommendation

Based on a thorough review of the following codebases:

| Project | Type | Core Stack |
|---------|------|-----------|
| **croozie-expo** | Mobile app (Expo/React Native) | Expo 55, React 19, TanStack Query, better-auth, Reanimated, d3-sankey |
| **croozie-next** | Full-stack web app | Next.js 16, Drizzle ORM, PostgreSQL, pg-boss, Playwright scrapers, OpenRouter LLM |
| **slidemark** | SaaS web app | Next.js 16, TipTap editor, Drizzle ORM, pg-boss, PptxGenJS, Cloudflare R2, Polar billing |
| **videos-slidemark** | Promotional video | Remotion 4, React 19, Tailwind, spring animations |
| **videos-croozie** | Promotional video | Remotion 4, React 19, Tailwind, TransitionSeries, lyric sync |
| **expo-multi-tenant-demo** | Mobile app template | Expo 55, React Native, env-driven multi-tenancy |
| **scraper-prototyping** | Web scraping suite | Node.js, Playwright, Firecrawl, Cheerio, Zod, 21 scrapers across 11 cruise lines |
| **WebCred** | SaaS platform (monorepo) | Next.js 16, Motia workflow engine, Drizzle ORM, Apify, Firecrawl, Stripe, Better Auth |
| **hearst-ai-videos** | AI video generation | Remotion 4, React 19, OpenRouter, Ken Burns effects, frame-locked audio sync |
| **adibkadir-site** | Personal site + CMS | Next.js 16, Lexical editor, Drizzle ORM, Cloudflare R2, OpenRouter, Firecrawl |

---

## Recommended Agents

### 1. `nextjs-fullstack` — Next.js Full-Stack Engineer

**Why:** 7 of 10 projects use Next.js (16) with App Router, Server Components, `'use cache'`, Suspense streaming, and Server Actions. This is your dominant pattern.

**Capabilities:**
- Next.js 16 App Router architecture (route groups, layouts, parallel routes)
- Server Components vs Client Components boundary decisions
- `'use cache'` directive with `cacheLife()` and `cacheTag()` for PPR
- Server Actions for mutations (form handling, optimistic updates)
- Streaming with Suspense boundaries
- Middleware for auth, tenant resolution, redirects
- API routes for webhooks, streaming endpoints (SSE)
- Docker multi-stage builds (Alpine Node 20)
- Turbopack dev configuration

**Skills needed:** `nextjs-app-router`, `server-actions`, `cache-components`, `docker-nextjs`

---

### 2. `expo-mobile` — Expo / React Native Mobile Engineer

**Why:** croozie-expo and expo-multi-tenant-demo both use Expo 55 with Expo Router, and your Next.js apps serve as API backends for the mobile apps.

**Capabilities:**
- Expo 55 with Expo Router (file-based routing, typed routes)
- React Native with Reanimated for high-performance animations
- Bottom tab navigation with @react-navigation
- TanStack React Query for server state
- better-auth with expo-secure-store integration
- Context API provider patterns (Favorites, Match, Analytics, AppLaunch)
- EAS build profiles (development, preview, production)
- Multi-tenant architecture via environment-driven tenant selection
- Deep linking configuration
- Platform-specific code (iOS SF Symbols, Android minification)

**Skills needed:** `expo-router`, `react-native-animation`, `eas-builds`, `multi-tenant-expo`

---

### 3. `database-architect` — Database & ORM Architect

**Why:** Every full-stack project uses Drizzle ORM with PostgreSQL. Schema design, migrations, and query patterns are consistent across all projects.

**Capabilities:**
- Drizzle ORM schema design (tables, relations, indexes, enums)
- Drizzle Kit migrations (generate, push, seed)
- PostgreSQL optimization (connection pooling, query semaphore patterns)
- Multi-tenant schema patterns (agency/client/location hierarchies)
- better-auth schema integration (user, session, account, verification tables)
- Relational query patterns with joins
- pg-boss schema setup for background job queues
- Data normalization and denormalization decisions
- NanoID vs UUID for primary keys

**Skills needed:** `drizzle-orm`, `postgresql-patterns`, `better-auth-schema`

---

### 4. `auth-engineer` — Authentication & Authorization Engineer

**Why:** better-auth is used in croozie-next, croozie-expo, slidemark, WebCred, and adibkadir-site. Custom RBAC is used in WebCred.

**Capabilities:**
- better-auth setup and configuration (Drizzle adapter)
- Authentication strategies: email/password, magic link, OAuth
- Expo integration with expo-secure-store
- Session management (cookie-based, secure storage)
- Role-based access control (platform/agency/client hierarchies)
- Admin plugins and impersonation
- Rate limiting configuration
- API guards (`requireAuthApi()`, `requireAdminApi()`)
- Cross-subdomain cookie configuration for multi-tenant
- Polar subscription integration with better-auth

**Skills needed:** `better-auth`, `rbac-patterns`, `session-management`

---

### 5. `background-jobs` — Background Job & Worker Engineer

**Why:** pg-boss is used in croozie-next, slidemark, and WebCred. Motia workflow engine is used in WebCred. Job queue architecture is critical to all SaaS projects.

**Capabilities:**
- pg-boss setup (send-only client vs full worker client pattern)
- Queue design (14 queues in croozie-next, 3 in slidemark)
- Cron job polling patterns
- Batch processing with sequential iteration
- Job status tracking and error handling
- Motia workflow engine (step files, queue triggers, cron triggers, HTTP triggers)
- Event-driven architecture (topic-based pub/sub)
- Scrape orchestration patterns (per-source jobs, coordinator, failure isolation)
- Worker process separation from web server
- State management (in-memory dev, Redis prod)

**Skills needed:** `pg-boss-worker`, `motia-workflows`, `job-orchestration`

---

### 6. `web-scraper` — Web Scraper & Data Extraction Engineer

**Why:** scraper-prototyping has 21 scrapers across 11 cruise lines. croozie-next has integrated scrapers. WebCred scrapes 10+ review sources. Primary approach is intercepting network requests and replaying raw API calls — browsers and paid tools are last resorts.

**Capabilities:**
- Network request interception and raw HTTP replay with correct headers (primary approach)
- API reverse engineering from DevTools network tab, minified JS bundles, GraphQL introspection, Solr facets
- Direct fetch with REST, GraphQL, Solr, Algolia BFF replay
- Pagination: offset, cursor, date windowing, infinite scroll
- Cookie/session management and token extraction from HTML
- Playwright response interception (`page.on('response')`) for SPAs with auth cookies
- In-browser fetch (`page.evaluate`) to reuse authenticated sessions
- Anti-detection: navigator spoofing, `headless: false` + Xvfb, bot challenge detection
- Crawlee framework: AdaptivePlaywrightCrawler, proxy rotation, session pools, fingerprinting, RequestQueue
- Firecrawl API: markdown/HTML/rawHtml scraping, LLM-powered structured extraction with Zod, stealth proxy, crawling, search, agent, browser sandbox
- Mobile app emulation: Python + uiautomator2, mobile-mcp, scrcpy-mcp, raw ADB commands
- Standalone Node.js scripts — tested independently before integration
- Data normalization to standardized JSON with Zod validation
- 5-tier escalation ladder: fetch → Playwright → Crawlee → Firecrawl → Mobile emulator
- Next.js `/admin/scrapers` page: list scrapers, execute on-demand, show status/errors, JSON textarea with copy button, localStorage caching across refreshes
- Next.js `/admin/jobs` page: pg-boss job monitoring, queue filters, run tracking with progress bars, 5s polling
- pg-boss v12 dual-client pattern, cron scheduling, staggered dispatch, job chaining
- better-auth admin role gating for all admin endpoints

**Skills:** `api-reverse-engineering`, `http-replay-scraping`, `data-normalization`, `playwright-interception`, `crawlee`, `firecrawl`, `mobile-emulator-fallback`, `scraper-admin-ui`, `pg-boss-jobs`, `job-admin-ui`

---

### 7. `remotion-video` — Remotion Video Production Engineer

**Why:** 3 projects (videos-slidemark, videos-croozie, hearst-ai-videos) use Remotion for programmatic video generation with sophisticated animation systems.

**Capabilities:**
- Remotion 4 composition setup (multi-variant, multi-aspect-ratio)
- Spring animation system (configurable damping, stiffness, mass)
- Interpolation and easing functions
- Frame-based deterministic animation (no external state)
- TransitionSeries with crossfade patterns
- Audio synchronization (voiceover + background music, volume control)
- Lyric/subtitle timing sync (word-by-word reveal)
- Ken Burns camera effects (6 directions with focus points)
- Component library: BrowserChrome, PhoneMockup, ImageMosaic, FloatingType, StatReveal
- OffthreadVideo for performance
- Tailwind v4 integration with Remotion
- JPEG output optimization
- Captions and subtitles (SRT import, transcription, display)
- Audio visualization (spectrum bars, waveforms, bass-reactive effects)
- 3D content with React Three Fiber
- Charts and data visualization (bar, pie, line, stock)
- Maps with Mapbox integration
- AI-generated voiceover with ElevenLabs TTS
- Transparent video rendering
- Dynamic metadata with calculateMetadata
- Parametrizable videos with Zod schemas

**Skills:** `remotion` (official Remotion best practices — 38 rule files covering animations, timing, sequencing, transitions, audio, video, captions, 3D, charts, fonts, maps, voiceover, and more), `video-production-patterns` (production patterns from hearst-ai-videos, videos-croozie, videos-slidemark — segment-driven architecture, Ken Burns, spring presets, frame-locked voiceover sync, cinematic overlays, editorial typography)

---

### 8. `ai-integration` — AI/LLM Integration Engineer

**Why:** OpenRouter, Google GenAI, and OpenAI are used across croozie-next, slidemark, WebCred, hearst-ai-videos, and adibkadir-site for content generation, analysis, and moderation.

**Capabilities:**
- OpenRouter SDK setup (multi-model routing, model selection)
- Structured output with AI (slide formatting, review analysis, content generation)
- Streaming responses (SSE endpoints)
- AI-powered content moderation (comment moderation, review analysis)
- Content generation pipelines (article generation, social posts, tag suggestions)
- AI-assisted scraping (Firecrawl + LLM extraction)
- Image generation (Google Gemini)
- Cost tracking and rate limiting for AI features
- System prompt engineering for specific output formats
- Batch processing of AI tasks through job queues

**Skills needed:** `openrouter-sdk`, `ai-content-pipeline`, `structured-ai-output`

---

### 9. `cloud-storage` — Cloudflare R2 Storage Engineer

**Why:** Cloudflare R2 (S3-compatible) is used in croozie-next, slidemark, and adibkadir-site with distinct upload strategies (direct, presigned, base64). Each project has different patterns for client configuration, file naming, and lifecycle management.

**Capabilities:**
- R2 S3 client configuration (`forcePathStyle`, `requestChecksumCalculation`, singleton pattern)
- Presigned URL generation for direct-to-R2 browser uploads (PUT with `unhoistableHeaders`)
- Presigned download URLs with client-side caching and deduplication
- Direct server uploads via Server Actions and API routes
- Base64 buffer uploads for AI-generated images
- URL-fetch-and-store pattern (download from external URL → upload to R2)
- Media database tracking with Drizzle ORM (key, filename, contentType, size)
- Orphaned media cleanup cron jobs (content-aware detection)
- Cascade deletion of R2 objects when entities are deleted
- File validation (type allowlists, size limits)
- CDN domain configuration for public asset serving
- Avatar upload pattern (overwrite-on-update with user-scoped keys)
- Worker-side uploads for export/generation pipelines

**Skills needed:** `r2-client-setup`, `presigned-urls`, `upload-patterns`, `media-management`

---

### 10. `ui-components` — UI Component & Design System Engineer

**Why:** Consistent UI patterns across projects: Radix UI, shadcn/ui, TipTap/Lexical editors, Tailwind CSS 4, Tremor charts.

**Capabilities:**
- Tailwind CSS 4 configuration and theming
- Radix UI primitives (dialog, dropdown, select, avatar, tooltip)
- shadcn/ui component patterns
- TipTap rich text editor setup (extensions, markdown, slash commands)
- Lexical editor integration (code, lists, links, HTML)
- Recharts / D3 data visualization
- Dark mode with next-themes
- Mermaid diagram rendering
- Form patterns (React Hook Form + Zod validation)
- Sonner toast notifications
- DnD Kit for sortable lists
- Glass morphism effects (expo-glass-effect)

**Skills needed:** `tailwind-v4`, `radix-shadcn`, `rich-text-editor`, `data-visualization`

---

### 11. `devops-deploy` — DevOps & Deployment Engineer

**Why:** Every production project uses Docker multi-stage builds with Next.js standalone output. croozie-next and slidemark run pg-boss workers alongside Next.js via entrypoint scripts. WebCred uses Dokploy with Traefik wildcard routing and init containers for migrations.

**Capabilities:**
- Docker multi-stage builds (3-stage: deps → builder → runner, Alpine and Debian Slim)
- Next.js standalone output with `serverExternalPackages` for pg-boss/postgres
- Monorepo Docker builds with `outputFileTracingRoot` and pnpm workspaces
- Dokploy deployment with docker-compose, env_file, and GitHub integration
- Traefik labels for wildcard subdomain routing (`HostRegexp`)
- Let's Encrypt TLS (HTTP challenge for apex, DNS challenge for wildcard)
- Docker entrypoint scripts (migrations → worker → Next.js)
- Worker auto-restart loops with background subshells
- Xvfb + Chromium for headless browser automation in containers
- Init containers for database migrations (`service_completed_successfully`)
- Drizzle ORM migration strategies (`drizzle-kit push` vs `drizzle-kit migrate`)
- Schema verification after migration
- Health check endpoints validating database + worker connectivity
- Non-root users (UID 1001) in production containers
- `.dockerignore` patterns for clean builds
- Build-time vs runtime environment variable management
- Dev mode: `concurrently` for Next.js + worker with `--kill-others-on-fail`
- Graceful shutdown with SIGTERM/SIGINT handlers

**Skills needed:** `docker-nextjs`, `dokploy-traefik`, `worker-entrypoint`, `db-migrations`

---

### 12. `saas-architect` — SaaS Platform Architect

**Why:** WebCred and slidemark are full SaaS platforms with multi-tenancy, billing, RBAC, and feature flags. These patterns repeat.

**Capabilities:**
- Multi-tenant architecture (subdomain-based, env-driven)
- Subscription/billing integration (Stripe in WebCred, Polar in slidemark)
- Feature flag system (code-driven, auto-sync, admin toggle)
- Free tier limits and quota enforcement
- Organization/workspace/team hierarchy
- Sharing and access control (private, org, team, public)
- Admin dashboards with user management
- Invitation and access request workflows
- Activity logging and audit trails
- URL filter state pattern for bookmarkable views
- DAL (Data Access Layer) pattern for centralized auth checks

**Skills needed:** `multi-tenant`, `stripe-billing`, `feature-flags`, `access-control`

---

## Agent-Skill Matrix

| Agent | Skills |
|-------|--------|
| `nextjs-fullstack` | `nextjs-app-router`, `server-actions`, `cache-components`, `docker-nextjs` |
| `expo-mobile` | `expo-router`, `react-native-animation`, `eas-builds`, `multi-tenant-expo` |
| `database-architect` | `drizzle-orm`, `postgresql-patterns`, `better-auth-schema` |
| `auth-engineer` | `better-auth`, `rbac-patterns`, `session-management` |
| `background-jobs` | `pg-boss-worker`, `motia-workflows`, `job-orchestration` |
| `web-scraper` | `api-reverse-engineering`, `http-replay-scraping`, `data-normalization`, `playwright-interception`, `crawlee`, `firecrawl`, `mobile-emulator-fallback`, `scraper-admin-ui`, `pg-boss-jobs`, `job-admin-ui` |
| `remotion-video` | `remotion`, `video-production-patterns` |
| `ai-integration` | `openrouter-sdk`, `ai-content-pipeline`, `structured-ai-output` |
| `cloud-storage` | `cloudflare-r2`, `presigned-uploads`, `resend-email` |
| `ui-components` | `tailwind-v4`, `radix-shadcn`, `rich-text-editor`, `data-visualization` |
| `devops-deploy` | `docker-nextjs`, `pnpm-monorepo`, `eas-builds`, `dokploy-traefik` |
| `saas-architect` | `multi-tenant`, `stripe-billing`, `feature-flags`, `access-control` |

---

## Project Coverage Matrix

Shows which agents are needed per project type:

| Project Type | Agents Required |
|-------------|----------------|
| **Next.js SaaS** (slidemark, WebCred) | `nextjs-fullstack`, `database-architect`, `auth-engineer`, `background-jobs`, `ai-integration`, `cloud-storage`, `ui-components`, `saas-architect`, `devops-deploy` |
| **Next.js App** (croozie-next, adibkadir-site) | `nextjs-fullstack`, `database-architect`, `auth-engineer`, `background-jobs`, `ai-integration`, `cloud-storage`, `ui-components`, `devops-deploy` |
| **Expo Mobile** (croozie-expo, expo-multi-tenant) | `expo-mobile`, `auth-engineer`, `ui-components` |
| **Remotion Video** (videos-*) | `remotion-video` |
| **Scraper Suite** (scraper-prototyping) | `web-scraper` |
| **Full Platform** (all of the above) | All 12 agents |

---

## Anthropic Skills Integration

Each agent's skills are implemented as SKILL.md files following the [Anthropic Agent Skills spec](https://github.com/anthropics/skills). Skills use progressive disclosure:

1. **Level 1 (Metadata):** YAML frontmatter with name + description (~100 tokens, always loaded)
2. **Level 2 (Instructions):** SKILL.md body with workflows and patterns (<5k tokens, loaded on trigger)
3. **Level 3 (Resources):** Reference files, templates, code snippets (loaded as needed)

Skills from the [anthropic/skills](https://github.com/anthropics/skills) repo that are directly applicable:
- `frontend-design` — UI component design patterns
- `webapp-testing` — Web application testing
- `skill-creator` — Creating and iterating on new skills
- `mcp-builder` — Building MCP servers
- `xlsx`, `pdf`, `pptx`, `docx` — Document generation skills

Custom skills are created for domain-specific patterns derived from the analyzed codebases.
