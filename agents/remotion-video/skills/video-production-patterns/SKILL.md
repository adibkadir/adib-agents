---
name: video-production-patterns
description: Production video composition patterns from real Remotion projects — multi-variant compositions, segment-driven architecture, Ken Burns camera effects, spring animation presets, frame-locked voiceover sync, cinematic overlays, ImageMosaic grids, editorial typography, and rendering configuration. Use when building professional-grade video compositions beyond basic Remotion usage.
---

# Video Production Patterns

Patterns extracted from production Remotion projects (hearst-ai-videos, videos-croozie, videos-slidemark) for creating professional automotive reviews, promotional videos, and branded content.

## Multi-Variant Composition Architecture

Register multiple compositions in Root.tsx to support different voiceovers, lengths, and visual treatments of the same content:

```tsx
// src/Root.tsx
import { Composition } from 'remotion'

export const RemotionRoot: React.FC = () => {
  return (
    <>
      <Composition
        id="ProductVideo-V1"
        component={ProductVideoV1}
        durationInFrames={2197}
        fps={30}
        width={1920}
        height={1080}
      />
      <Composition
        id="ProductVideo-V2"
        component={ProductVideoV2}
        durationInFrames={2970}
        fps={30}
        width={1920}
        height={1080}
      />
      <Composition
        id="ProductVideo-Female"
        component={ProductVideoFemale}
        durationInFrames={2416}
        fps={30}
        width={1920}
        height={1080}
      />
      <Composition
        id="ProductVideo-BRoll"
        component={ProductVideoBRoll}
        durationInFrames={2250}
        fps={30}
        width={1920}
        height={1080}
      />
    </>
  )
}
```

Each composition is a separate render target. Run `npx remotion render src/index.ts ProductVideo-V1` to export a specific variant.

## Segment-Driven Data Architecture

Define video content as typed segment data. Components read segments and compute their state from `useCurrentFrame()`:

```typescript
// src/data/segments.ts

export interface Segment {
  id: number
  startFrame: number
  endFrame: number
  imageUrl: string
  kenBurns: {
    direction: 'zoomIn' | 'zoomOut' | 'panLeft' | 'panRight' | 'panUp' | 'diagonal'
    focusX: number  // 0-100, percentage
    focusY: number  // 0-100, percentage
  }
  editorial: {
    lineOne: string
    lineTwo: string
    accentColor: string
  }
  stats: Array<{
    label: string
    value: string
    unit: string
    animationType: 'countUp' | 'snapIn'
  }>
  transition: 'flash' | 'hardCut'
  narration: string  // transcript for this segment
}

export const SEGMENTS: Segment[] = [
  {
    id: 1,
    startFrame: 0,
    endFrame: 450,
    imageUrl: 'https://cdn.example.com/hero-exterior.jpg',
    kenBurns: { direction: 'zoomIn', focusX: 50, focusY: 40 },
    editorial: {
      lineOne: 'Redesigned from the ground up',
      lineTwo: 'The 2026 CR-V',
      accentColor: '#e11d48',
    },
    stats: [
      { label: 'Horsepower', value: '190', unit: 'hp', animationType: 'countUp' },
      { label: 'MPG Combined', value: '32', unit: 'mpg', animationType: 'snapIn' },
    ],
    transition: 'flash',
    narration: 'Honda has completely redesigned the CR-V for 2026...',
  },
  // ... more segments
]

// Timing constants
export const FPS = 30
export const MOSAIC_HOLD = 45     // 1.5s grid visible before zoom
export const FIRST_ZOOM_START = 45
export const FIRST_ZOOM_COMPLETE = 75
```

## Spring Animation Presets

Reusable spring configs for consistent feel across all animated elements:

```typescript
// src/constants/springs.ts

export const PUNCH = { damping: 12, stiffness: 200 }   // snappy entrance
export const SNAP = { damping: 200 }                    // near-instant, no bounce
export const BOUNCE = { damping: 8 }                     // playful overshoot
export const HEAVY = { damping: 15, stiffness: 80, mass: 2 }  // slow, weighty

// Usage in components:
import { spring, useCurrentFrame, useVideoConfig } from 'remotion'

const frame = useCurrentFrame()
const { fps } = useVideoConfig()

const scale = spring({
  frame: frame - entryFrame,
  fps,
  config: PUNCH,
})
```

## Ken Burns Effect

Animate images with cinematic camera movements:

```tsx
// src/components/KenBurns.tsx
import { useCurrentFrame, interpolate, Easing, Img } from 'remotion'

interface KenBurnsProps {
  src: string
  direction: 'zoomIn' | 'zoomOut' | 'panLeft' | 'panRight' | 'panUp' | 'diagonal'
  focusX: number
  focusY: number
  startFrame: number
  endFrame: number
}

export const KenBurns: React.FC<KenBurnsProps> = ({
  src, direction, focusX, focusY, startFrame, endFrame,
}) => {
  const frame = useCurrentFrame()
  const progress = interpolate(
    frame,
    [startFrame, endFrame],
    [0, 1],
    { extrapolateLeft: 'clamp', extrapolateRight: 'clamp', easing: Easing.inOut(Easing.ease) }
  )

  const getTransform = () => {
    switch (direction) {
      case 'zoomIn':
        return `scale(${1 + progress * 0.15})`
      case 'zoomOut':
        return `scale(${1.15 - progress * 0.15})`
      case 'panLeft':
        return `translateX(${-progress * 8}%) scale(1.1)`
      case 'panRight':
        return `translateX(${progress * 8}%) scale(1.1)`
      case 'panUp':
        return `translateY(${-progress * 8}%) scale(1.1)`
      case 'diagonal':
        return `translate(${progress * 5}%, ${-progress * 3}%) scale(${1 + progress * 0.1})`
    }
  }

  return (
    <Img
      src={src}
      style={{
        width: '100%',
        height: '100%',
        objectFit: 'cover',
        objectPosition: `${focusX}% ${focusY}%`,
        transform: getTransform(),
        willChange: 'transform',
      }}
    />
  )
}
```

## Frame-Locked Voiceover Sync

Map voiceover audio timing to exact frame numbers:

```typescript
// scripts/transcribe-audio.mjs
// Uses OpenRouter to generate timestamps from audio

// Frame calculation: timestamp (seconds) × FPS = frame number
// Example: "Interior at 0:48" = 48 * 30 = frame 1440

// In composition:
import { Audio, Sequence, useCurrentFrame } from 'remotion'

export const VideoWithVoiceover: React.FC = () => {
  return (
    <AbsoluteFill>
      {/* Voiceover audio */}
      <Audio src={staticFile('voiceover.mp3')} volume={1} />

      {/* Background music at lower volume */}
      <Audio src={staticFile('background-music.mp3')} volume={0.15} />

      {/* Segments synced to voiceover timing */}
      <Sequence from={0} durationInFrames={450}>
        {/* Intro segment — synced to first 15s of voiceover */}
      </Sequence>

      <Sequence from={450} durationInFrames={360}>
        {/* Exterior segment — voiceover discusses exterior at 0:15 */}
      </Sequence>
    </AbsoluteFill>
  )
}
```

## Cinematic Overlays

Dark vignettes and color grades applied as overlay layers:

```tsx
// Vignette overlay
<AbsoluteFill
  style={{
    background: 'radial-gradient(ellipse at center, transparent 50%, rgba(0,0,0,0.6) 100%)',
    pointerEvents: 'none',
  }}
/>

// Dark overlay during transitions (controlled by frame)
<AbsoluteFill
  style={{
    backgroundColor: `rgba(0, 0, 0, ${interpolate(frame, [startFrame, startFrame + 12], [0, 0.4], { extrapolateRight: 'clamp' })})`,
    pointerEvents: 'none',
  }}
/>
```

## Segment Transition Component

```tsx
// src/components/SegmentTransition.tsx
import { useCurrentFrame, interpolate, AbsoluteFill } from 'remotion'

interface Props {
  type: 'flash' | 'hardCut'
  triggerFrame: number
}

export const SegmentTransition: React.FC<Props> = ({ type, triggerFrame }) => {
  const frame = useCurrentFrame()

  if (type === 'hardCut') return null

  // 4-frame white flash
  const opacity = interpolate(
    frame,
    [triggerFrame, triggerFrame + 2, triggerFrame + 4],
    [0, 1, 0],
    { extrapolateLeft: 'clamp', extrapolateRight: 'clamp' }
  )

  if (opacity <= 0) return null

  return (
    <AbsoluteFill style={{ backgroundColor: 'white', opacity }} />
  )
}
```

## Time Range Guard Pattern

Every animated component validates its frame window before rendering:

```tsx
const frame = useCurrentFrame()

// Don't render outside this component's time window
if (frame < startFrame || frame >= endFrame) return null

// Calculate local progress within this segment
const localFrame = frame - startFrame
const duration = endFrame - startFrame
const progress = localFrame / duration
```

## Word-by-Word Text Reveal

```tsx
// src/components/FloatingType.tsx
import { useCurrentFrame, spring, useVideoConfig } from 'remotion'

interface Props {
  lineOne: string
  lineTwo: string
  accentColor: string
  startFrame: number
  endFrame: number
}

const WORD_STAGGER = 8  // frames between each word

export const FloatingType: React.FC<Props> = ({
  lineOne, lineTwo, accentColor, startFrame, endFrame,
}) => {
  const frame = useCurrentFrame()
  const { fps } = useVideoConfig()

  if (frame < startFrame || frame >= endFrame) return null

  const localFrame = frame - startFrame
  const wordsOne = lineOne.split(' ')

  return (
    <div>
      {/* Line one: word-by-word reveal */}
      <div style={{ fontWeight: 300 }}>
        {wordsOne.map((word, i) => {
          const wordDelay = i * WORD_STAGGER
          const opacity = spring({
            frame: localFrame - wordDelay,
            fps,
            config: { damping: 20 },
          })
          return (
            <span key={i} style={{ opacity, display: 'inline-block', marginRight: 8 }}>
              {word}
            </span>
          )
        })}
      </div>

      {/* Line two: bold with accent color, delayed entrance */}
      <div style={{
        fontWeight: 700,
        color: accentColor,
        opacity: spring({ frame: localFrame - 15, fps, config: BOUNCE }),
        transform: `translateY(${interpolate(
          spring({ frame: localFrame - 15, fps, config: BOUNCE }),
          [0, 1], [20, 0]
        )}px)`,
      }}>
        {lineTwo}
      </div>
    </div>
  )
}
```

## Remotion Config

```typescript
// remotion.config.ts
import { Config } from '@remotion/cli/config'
import { enableTailwind } from '@remotion/tailwind-v4'

Config.setVideoImageFormat('jpeg')
Config.setOverwriteOutput(true)
Config.overrideWebpackConfig(enableTailwind)
```

## Rendering Output

```bash
# Render specific composition
npx remotion render src/index.ts ProductVideo-V1 out/ProductVideo-V1.mp4

# Render all compositions
npx remotion render src/index.ts --all

# Output: H.264 video + AAC audio (MP4), 1920x1080 @ 30fps
```
