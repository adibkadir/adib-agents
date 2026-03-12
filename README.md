# adib-agents

Specialized Claude Code agents with domain-specific skills, built from patterns learned across production projects.

## Quick Start

Launch any agent using Claude Code's `--agent` flag:

```bash
# Launch an agent by path
claude --agent agents/web-scraper/AGENT.md

# Or from any project directory — use the full path
claude --agent /path/to/adib-agents/agents/web-scraper/AGENT.md

# Examples
claude --agent agents/nextjs-fullstack/AGENT.md
claude --agent agents/devops-deploy/AGENT.md
claude --agent agents/cloud-storage/AGENT.md
claude --agent agents/remotion-video/AGENT.md
claude --agent agents/news-video-producer/AGENT.md
claude --agent agents/video-producer/AGENT.md
```

The agent's persona, rules, and skills are loaded into the session. Skills are auto-discovered from the agent's `skills/` directory.

## Available Agents

| Agent | Description | Skills |
|-------|-------------|--------|
| **web-scraper** | Network-first scraping engineer. Intercepts API calls, replays with fetch, escalates through 5 tiers. Builds admin UI for scraper management. | 10 skills |
| **nextjs-fullstack** | Next.js App Router expert. Server Components, Server Actions, caching, Docker deployment. | 4 skills |
| **devops-deploy** | Docker + Dokploy + Traefik deployment. DB migrations, worker processes. | 4 skills |
| **cloud-storage** | Cloudflare R2 storage. Presigned URLs, media management, upload patterns. | 4 skills |
| **news-video-producer** | AI news video pipeline. Research, script generation, Remotion composition. | 4 skills |
| **video-producer** | Creative video direction. Footage capture, voiceover, creative direction. | 3 skills |
| **remotion-video** | Remotion video production. Frame-based animation, spring presets, segment architecture. | 2 skills |

## Structure

```
adib-agents/
├── agents/
│   ├── web-scraper/
│   │   ├── AGENT.md                    # Agent persona, rules, workflow
│   │   └── skills/
│   │       ├── http-replay-scraping/   # Tier 1: fetch + JSON replay
│   │       ├── playwright-interception/# Tier 2: browser response capture
│   │       ├── crawlee/                # Tier 3: managed anti-blocking
│   │       ├── firecrawl/              # Tier 4: paid proxy + anti-bot
│   │       ├── mobile-emulator-fallback/# Tier 5: ADB + uiautomator2
│   │       ├── api-reverse-engineering/# DevTools + JS bundle analysis
│   │       ├── data-normalization/     # Zod schemas, date/price formatting
│   │       ├── scraper-admin-ui/       # /admin/scrapers page
│   │       ├── pg-boss-jobs/           # Background job queues
│   │       └── job-admin-ui/           # /admin/jobs monitoring
│   ├── nextjs-fullstack/
│   ├── devops-deploy/
│   ├── cloud-storage/
│   ├── news-video-producer/
│   ├── video-producer/
│   └── remotion-video/
├── agents-recommendation.md            # Full analysis and rationale
└── CLAUDE.md                           # Project instructions
```

## How It Works

Each agent has:
1. **AGENT.md** — Persona, identity, critical rules, architecture patterns, workflow
2. **skills/** — Modular SKILL.md files following the [Anthropic Agent Skills spec](https://github.com/anthropics/skills)

Skills use progressive disclosure:
- **Level 1 (always loaded):** YAML frontmatter name + description (~100 tokens)
- **Level 2 (on trigger):** SKILL.md body with instructions (<5k tokens)
- **Level 3 (as needed):** Referenced files, templates, scripts

## Alternative Usage

### Copy skills into your project

```bash
# Copy an agent's skills to your project's .claude directory
cp -r agents/web-scraper/skills/* /path/to/project/.claude/skills/
```

### Reference in CLAUDE.md

Add agent instructions directly to your project's CLAUDE.md:

```markdown
When working on scraping tasks, follow the Web Scraper agent rules:
- Network-first: intercept and replay API calls before using browsers
- Try mobile web version first
- Exhaust free options before paid tools
```

## Adding a New Agent

1. Create `agents/{agent-name}/AGENT.md` with YAML frontmatter (`name`, `description`, `color`, `emoji`, `vibe`, `skills`)
2. Create skills in `agents/{agent-name}/skills/{skill-name}/SKILL.md`
3. Each SKILL.md needs `name` and `description` in frontmatter
4. Update `agents-recommendation.md` with the new agent

## Derived From

These agents encode patterns from real production projects:
- croozie-expo, croozie-next (cruise platform)
- slidemark (presentation SaaS)
- WebCred (reputation management SaaS)
- hearst-ai-videos, videos-slidemark, videos-croozie (Remotion video)
- scraper-prototyping (21 cruise line scrapers)
- expo-multi-tenant-demo (multi-tenant mobile)
- adibkadir-site (personal site + CMS)
