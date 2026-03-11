# adib-agents

Specialized Claude Code agents with domain-specific skills, built from patterns learned across production projects.

## Structure

```
adib-agents/
├── agents/                          # Agent definitions
│   ├── nextjs-fullstack/            # Next.js Full-Stack Engineer (COMPLETE)
│   │   ├── AGENT.md                 # Agent persona, rules, patterns
│   │   └── skills/                  # Agent-specific skills
│   │       ├── nextjs-app-router/
│   │       │   └── SKILL.md
│   │       ├── server-actions/
│   │       │   └── SKILL.md
│   │       ├── cache-components/
│   │       │   └── SKILL.md
│   │       └── docker-nextjs/
│   │           └── SKILL.md
│   ├── web-scraper/                 # Web Scraping Engineer (COMPLETE)
│   │   ├── AGENT.md
│   │   └── skills/
│   │       ├── playwright-scraping/
│   │       │   └── SKILL.md
│   │       ├── firecrawl/
│   │       │   └── SKILL.md
│   │       ├── api-reverse-engineering/
│   │       │   └── SKILL.md
│   │       └── data-normalization/
│   │           └── SKILL.md
│   ├── expo-mobile/                 # Expo/React Native (stub)
│   ├── database-architect/          # Drizzle ORM + PostgreSQL (stub)
│   ├── auth-engineer/               # better-auth + RBAC (stub)
│   ├── background-jobs/             # pg-boss + Motia (stub)
│   ├── remotion-video/              # Remotion video production (stub)
│   ├── ai-integration/              # OpenRouter + LLM pipelines (stub)
│   ├── cloud-storage/               # R2 + Resend + Umami (stub)
│   ├── ui-components/               # Tailwind + Radix + editors (stub)
│   ├── devops-deploy/               # Docker + pnpm + EAS (stub)
│   └── saas-architect/              # Multi-tenant + billing (stub)
├── shared-skills/                   # Skills shared across agents
├── agents-recommendation.md         # Full analysis and rationale
└── CLAUDE.md                        # Project instructions
```

## How It Works

Each agent has:
1. **AGENT.md** — Persona, identity, critical rules, architecture patterns, workflow
2. **skills/** — Modular SKILL.md files following the [Anthropic Agent Skills spec](https://github.com/anthropics/skills)

Skills use progressive disclosure:
- **Level 1 (always loaded):** YAML frontmatter name + description (~100 tokens)
- **Level 2 (on trigger):** SKILL.md body with instructions (<5k tokens)
- **Level 3 (as needed):** Referenced files, templates, scripts

## Using an Agent

### In Claude Code

Copy the agent and its skills into your project:

```bash
# Copy an agent's skills to your project
cp -r agents/nextjs-fullstack/skills/* .claude/skills/

# Or install globally
cp -r agents/nextjs-fullstack/skills/* ~/.claude/skills/
```

Then activate in a Claude Code session:

```
"Activate Next.js Full-Stack Engineer mode and help me build a dashboard page with cached data"
```

### As Claude Code Custom Instructions

Reference the AGENT.md content in your project's CLAUDE.md:

```markdown
<!-- In your project's CLAUDE.md -->
When working on this project, follow the patterns in the Next.js Full-Stack Engineer agent.
Use Server Components by default. Use 'use cache' for expensive queries.
Use Server Actions for all mutations.
```

## Adding a New Agent

1. Create `agents/{agent-name}/AGENT.md` with YAML frontmatter
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
