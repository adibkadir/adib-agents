# adib-agents

This repo contains specialized Claude Code agents with modular skills.

## Structure
- `agents/{name}/AGENT.md` — Agent persona, rules, architecture patterns
- `agents/{name}/skills/{skill}/SKILL.md` — Modular skills following Anthropic Agent Skills spec
- `shared-skills/` — Skills used across multiple agents
- `agents-recommendation.md` — Full analysis of recommended agents

## Conventions
- AGENT.md uses YAML frontmatter: name, description, color, emoji, vibe, skills
- SKILL.md uses YAML frontmatter: name, description
- Skills follow progressive disclosure (metadata → instructions → resources)
- All code examples use the actual patterns from production projects (croozie, slidemark, WebCred, etc.)
- TypeScript strict mode, pnpm, Node 20

## Key Tech Stack (across all projects)
- Next.js 16 App Router, React 19, Tailwind CSS 4
- Drizzle ORM + PostgreSQL
- better-auth for authentication
- pg-boss for background jobs
- Cloudflare R2 for storage
- OpenRouter for AI/LLM
- Remotion for video generation
- Expo 55 for mobile apps
