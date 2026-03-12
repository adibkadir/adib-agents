---
name: remotion-best-practices
description: Reference to the official Remotion skills from the remotion-dev/skills repository. Covers animations, compositions, sequencing, timing, audio, text animations, transitions, voiceover, subtitles, captions, and more. Use this as a companion to all Remotion video composition work.
---

# Remotion Best Practices

## Installation

Install the Remotion skills from the official repository for use with Claude Code:

```bash
# Clone the skills repo
git clone https://github.com/remotion-dev/skills.git .claude/skills/remotion

# Or install via npx (if available)
npx remotion add skills
```

The skills are loaded as rule files and provide comprehensive guidance for Remotion development.

## Key Rules for News Videos

The following rules from `remotion-dev/skills` are most relevant for news video production. Reference them when building compositions:

### Core Patterns

| Rule File | Purpose | When to Reference |
|-----------|---------|------------------|
| `animations.md` | `useCurrentFrame()`, `interpolate()`, `spring()`, easing | Every animation |
| `compositions.md` | Composition setup, default props, folders, stills | Project scaffolding |
| `sequencing.md` | `<Sequence>`, `<Series>`, timeline ordering | Scene arrangement |
| `timing.md` | Frame-based timing, duration calculation, spring configs | All timing logic |
| `calculate-metadata.md` | Dynamic duration, dimensions, props from data | Audio-driven duration |

### Media

| Rule File | Purpose | When to Reference |
|-----------|---------|------------------|
| `audio.md` | `<Audio>` component, volume, trimming, looping | Voiceover + music mixing |
| `images.md` | `<Img>` component, static files, remote URLs | News article images |
| `fonts.md` | Google Fonts via `@remotion/google-fonts`, local fonts | Multi-language typography |

### Text & Subtitles

| Rule File | Purpose | When to Reference |
|-----------|---------|------------------|
| `text-animations.md` | Typewriter effects, word highlighting | Headline animations |
| `subtitles.md` | Caption types, subtitle workflows | Subtitle data structure |
| `measuring-text.md` | Text dimensions, fitting to containers | Dynamic text sizing |

### Transitions & Effects

| Rule File | Purpose | When to Reference |
|-----------|---------|------------------|
| `transitions.md` | `<TransitionSeries>`, fade, slide, wipe, flip | Scene transitions |
| `light-leaks.md` | Light leak overlays via `@remotion/light-leaks` | Visual polish (optional) |

### Advanced

| Rule File | Purpose | When to Reference |
|-----------|---------|------------------|
| `voiceover.md` | ElevenLabs TTS integration, dynamic duration | Voiceover generation |
| `parameters.md` | Zod schemas for parametrized videos | Config-driven rendering |
| `tailwind.md` | TailwindCSS v4 integration | Styling (if using Tailwind) |
| `get-audio-duration.md` | Audio duration detection | Duration calculation |

## Critical Remotion Rules

These rules are non-negotiable for deterministic video rendering:

1. **Never use CSS animations or transitions.** All motion must be driven by `useCurrentFrame()` combined with `interpolate()` or `spring()`. CSS animations are non-deterministic across renders.

2. **Use `staticFile()` for assets in `public/`.** Never use relative paths. Always `staticFile('images/hero.jpg')` instead of `'./public/images/hero.jpg'`.

3. **Use `getAudioDurationInSeconds()` for dynamic duration.** In `calculateMetadata`, measure the voiceover audio to set composition length. Never hardcode duration.

4. **Use `<Img>` from Remotion, not `<img>`.** The Remotion `<Img>` component handles loading states and prevents render artifacts from images that haven't loaded yet.

5. **Use `<OffthreadVideo>` for video embeds.** Better performance than `<Video>` for embedded video clips.

6. **Load fonts with `@remotion/google-fonts`.** This ensures fonts are available at render time. Never rely on system fonts or external CDN links.

7. **All interpolations must be clamped.** Always pass `{ extrapolateLeft: 'clamp', extrapolateRight: 'clamp' }` to `interpolate()` to prevent values going out of bounds.

8. **Use `spring()` for organic motion.** Prefer `spring()` over linear `interpolate()` for entrance animations, scale effects, and position changes. Configure `damping` (12-20) and `stiffness` (80-120) for natural feel.

9. **Compositions must have fixed dimensions.** Always 1080x1920 for vertical news videos. Never use percentage-based sizing for the root composition.

10. **Audio volume must be frame-driven.** Use `interpolate()` to drive volume changes (fade in/out). Never use static volume values that change abruptly.

## Asset Templates

The skills repo includes ready-to-use component templates in `rules/assets/`:

- **charts-bar-chart.tsx** — Animated bar chart with staggered entrance
- **text-animations-typewriter.tsx** — Character-by-character typewriter effect
- **text-animations-word-highlight.tsx** — Spring-animated word highlight wipe

These can be adapted for news data visualizations (polling data, market stats, etc.).
