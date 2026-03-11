---
name: video-creative-direction
description: Creative direction patterns for YouTube and TikTok promotional videos. Covers video structure, pacing, visual storytelling, platform-specific best practices, and Remotion composition patterns for product demos and walkthroughs.
---

# Video Creative Direction

## When to Use

Use this skill whenever you need to:
- Structure a promotional video for YouTube or TikTok
- Design scene compositions and transitions for product demos
- Choose the right pacing, visual style, and storytelling arc
- Build Remotion compositions that match platform best practices

## Platform Specifications

| Property | YouTube | TikTok |
|----------|---------|--------|
| Aspect ratio | 16:9 | 9:16 |
| Resolution | 1920x1080 | 1080x1920 |
| FPS | 30 | 30 |
| Ideal duration | 60s - 5min | 15s - 60s |
| Hook window | First 5s | First 2s |
| CTA placement | Last 10-15s | Last 5s |
| Text safe zone | Full frame | Center 80% (avoid top/bottom system UI) |
| Caption style | Optional | Essential (most viewers watch muted) |

## Video Structure Templates

### YouTube Walkthrough (2-3 minutes)

```
0:00-0:05  HOOK          Bold statement or question that creates curiosity
0:05-0:15  PROBLEM       Show the pain point (frustration, complexity, time waste)
0:15-0:25  SOLUTION      Introduce the product — logo reveal, tagline
0:25-1:30  WALKTHROUGH   Show 3-5 key features in action
1:30-1:45  SOCIAL PROOF  Stats, testimonials, logos, awards
1:45-2:00  CTA           "Try it free at..." with URL on screen
```

### TikTok Promo (30 seconds)

```
0:00-0:02  HOOK          Attention-grabbing visual or text ("POV: you just found...")
0:02-0:05  PROBLEM       Quick pain point (one sentence)
0:05-0:08  SOLUTION      Product intro (logo + one line)
0:08-0:25  DEMO          Fast-paced feature showcase (2-3 screens, quick cuts)
0:25-0:30  CTA           "Link in bio" or "Try it free"
```

### TikTok Micro-Promo (15 seconds)

```
0:00-0:02  HOOK          Text overlay: "This app just changed everything"
0:02-0:10  DEMO          Single killer feature, fast walkthrough
0:10-0:15  CTA           Product name + "Link in bio"
```

## Visual Style Guide

### Browser Frame Mockup

For product walkthroughs, wrap screenshots in a stylized browser frame:

```tsx
import { useCurrentFrame, useVideoConfig, interpolate, spring } from 'remotion'

export const BrowserFrame: React.FC<{
  screenshotSrc: string
  url: string
}> = ({ screenshotSrc, url }) => {
  const frame = useCurrentFrame()
  const { fps } = useVideoConfig()

  const scale = spring({ frame, fps, config: { damping: 15, stiffness: 120 } })
  const opacity = interpolate(frame, [0, 15], [0, 1], { extrapolateRight: 'clamp' })

  return (
    <div
      style={{
        opacity,
        transform: `scale(${scale})`,
        borderRadius: 12,
        overflow: 'hidden',
        boxShadow: '0 25px 50px rgba(0,0,0,0.25)',
        border: '1px solid rgba(255,255,255,0.1)',
      }}
    >
      {/* Browser chrome */}
      <div
        style={{
          height: 40,
          background: '#1e1e2e',
          display: 'flex',
          alignItems: 'center',
          padding: '0 16px',
          gap: 8,
        }}
      >
        <div style={{ width: 12, height: 12, borderRadius: '50%', background: '#ff5f57' }} />
        <div style={{ width: 12, height: 12, borderRadius: '50%', background: '#febc2e' }} />
        <div style={{ width: 12, height: 12, borderRadius: '50%', background: '#28c840' }} />
        <div
          style={{
            marginLeft: 16,
            flex: 1,
            height: 24,
            borderRadius: 6,
            background: '#2d2d3f',
            display: 'flex',
            alignItems: 'center',
            padding: '0 12px',
            color: '#888',
            fontSize: 12,
          }}
        >
          {url}
        </div>
      </div>

      {/* Screenshot content */}
      <img src={screenshotSrc} style={{ width: '100%', display: 'block' }} />
    </div>
  )
}
```

### Phone Mockup (TikTok)

For TikTok videos, show the product inside a phone frame:

```tsx
export const PhoneMockup: React.FC<{
  screenshotSrc: string
}> = ({ screenshotSrc }) => {
  const frame = useCurrentFrame()
  const { fps } = useVideoConfig()

  const slideUp = spring({ frame, fps, config: { damping: 15, stiffness: 80 } })
  const y = interpolate(slideUp, [0, 1], [100, 0])

  return (
    <div
      style={{
        transform: `translateY(${y}px)`,
        width: 380,
        height: 780,
        borderRadius: 40,
        overflow: 'hidden',
        border: '8px solid #1a1a2e',
        boxShadow: '0 30px 60px rgba(0,0,0,0.4)',
        position: 'relative',
      }}
    >
      {/* Notch */}
      <div
        style={{
          position: 'absolute',
          top: 0,
          left: '50%',
          transform: 'translateX(-50%)',
          width: 150,
          height: 30,
          background: '#1a1a2e',
          borderRadius: '0 0 20px 20px',
          zIndex: 10,
        }}
      />

      <img
        src={screenshotSrc}
        style={{ width: '100%', height: '100%', objectFit: 'cover' }}
      />
    </div>
  )
}
```

## Transition Patterns

### Scene Transitions with TransitionSeries

```tsx
import { TransitionSeries, linearTiming } from '@remotion/transitions'
import { fade } from '@remotion/transitions/fade'
import { slide } from '@remotion/transitions/slide'
import { wipe } from '@remotion/transitions/wipe'

// Use different transitions for different emotional beats:
// - fade: calm, professional transitions (solution intro, CTA)
// - slide: energetic, forward motion (feature walkthroughs)
// - wipe: dramatic reveals (problem → solution)

export const VideoComposition: React.FC<Props> = (props) => {
  const { fps } = useVideoConfig()

  return (
    <TransitionSeries>
      <TransitionSeries.Sequence durationInFrames={5 * fps}>
        <HookScene />
      </TransitionSeries.Sequence>

      <TransitionSeries.Transition
        presentation={wipe({ direction: 'from-left' })}
        timing={linearTiming({ durationInFrames: fps / 2 })}
      />

      <TransitionSeries.Sequence durationInFrames={10 * fps}>
        <ProblemScene />
      </TransitionSeries.Sequence>

      <TransitionSeries.Transition
        presentation={fade()}
        timing={linearTiming({ durationInFrames: fps })}
      />

      <TransitionSeries.Sequence durationInFrames={15 * fps}>
        <SolutionScene />
      </TransitionSeries.Sequence>

      <TransitionSeries.Transition
        presentation={slide({ direction: 'from-right' })}
        timing={linearTiming({ durationInFrames: fps / 2 })}
      />

      <TransitionSeries.Sequence durationInFrames={5 * fps}>
        <CTAScene />
      </TransitionSeries.Sequence>
    </TransitionSeries>
  )
}
```

## Animation Patterns

### Text Reveal (Headlines, CTAs)

```tsx
export const TextReveal: React.FC<{ text: string; delay?: number }> = ({
  text,
  delay = 0,
}) => {
  const frame = useCurrentFrame()
  const { fps } = useVideoConfig()

  const adjustedFrame = frame - delay
  const progress = spring({
    frame: Math.max(0, adjustedFrame),
    fps,
    config: { damping: 15, stiffness: 100 },
  })

  const y = interpolate(progress, [0, 1], [40, 0])
  const opacity = interpolate(progress, [0, 1], [0, 1])

  return (
    <div style={{ transform: `translateY(${y}px)`, opacity }}>
      {text}
    </div>
  )
}
```

### Screenshot Zoom (Ken Burns)

```tsx
export const KenBurns: React.FC<{
  src: string
  direction: 'zoom-in' | 'zoom-out' | 'pan-left' | 'pan-right'
}> = ({ src, direction }) => {
  const frame = useCurrentFrame()
  const { durationInFrames } = useVideoConfig()
  const progress = frame / durationInFrames

  const transforms: Record<string, string> = {
    'zoom-in': `scale(${1 + progress * 0.1})`,
    'zoom-out': `scale(${1.1 - progress * 0.1})`,
    'pan-left': `translateX(${-progress * 5}%) scale(1.05)`,
    'pan-right': `translateX(${progress * 5}%) scale(1.05)`,
  }

  return (
    <div style={{ overflow: 'hidden', width: '100%', height: '100%' }}>
      <img
        src={src}
        style={{
          width: '100%',
          height: '100%',
          objectFit: 'cover',
          transform: transforms[direction],
        }}
      />
    </div>
  )
}
```

### Feature Callout (Highlight a UI element)

```tsx
export const FeatureCallout: React.FC<{
  screenshotSrc: string
  highlight: { x: number; y: number; width: number; height: number }
  label: string
}> = ({ screenshotSrc, highlight, label }) => {
  const frame = useCurrentFrame()
  const { fps } = useVideoConfig()

  const ringScale = spring({ frame, fps, config: { damping: 12, stiffness: 100 } })
  const labelOpacity = interpolate(frame, [15, 25], [0, 1], { extrapolateRight: 'clamp' })

  return (
    <div style={{ position: 'relative' }}>
      <img src={screenshotSrc} style={{ width: '100%' }} />

      {/* Highlight ring */}
      <div
        style={{
          position: 'absolute',
          left: highlight.x,
          top: highlight.y,
          width: highlight.width,
          height: highlight.height,
          border: '3px solid #f59e0b',
          borderRadius: 8,
          transform: `scale(${ringScale})`,
          boxShadow: '0 0 20px rgba(245, 158, 11, 0.4)',
        }}
      />

      {/* Label */}
      <div
        style={{
          position: 'absolute',
          left: highlight.x + highlight.width + 20,
          top: highlight.y,
          opacity: labelOpacity,
          background: '#f59e0b',
          color: '#000',
          padding: '8px 16px',
          borderRadius: 8,
          fontWeight: 'bold',
          fontSize: 18,
        }}
      >
        {label}
      </div>
    </div>
  )
}
```

## Pacing Guidelines

### Words per Scene (at 2.5 words/second narration)

| Scene Type | Duration | Word Count | Visual Pacing |
|-----------|----------|-----------|---------------|
| Hook | 3-5s | 8-12 words | Fast cuts, bold text |
| Problem | 5-15s | 12-38 words | Slow, empathetic visuals |
| Solution Intro | 5-10s | 12-25 words | Logo reveal, product shot |
| Feature Demo | 10-20s per feature | 25-50 words | Screenshot + callout animation |
| Social Proof | 10-15s | 25-38 words | Stats counter, testimonial cards |
| CTA | 5-10s | 12-25 words | URL on screen, logo, action button |

### TikTok-Specific Pacing

TikTok demands faster pacing than YouTube:
- Cut every 2-3 seconds (viewers decide to swipe in < 2s)
- Text overlays on screen at all times (80% of TikTok is watched muted)
- Use motion constantly — static screenshots lose attention
- Front-load the value proposition (no slow buildups)

### YouTube-Specific Pacing

YouTube allows more deliberate pacing:
- 5-8 second shots for walkthroughs (let the viewer absorb the UI)
- Use Ken Burns on screenshots to add subtle motion
- Transitions can be longer (0.5-1s crossfades)
- Include a brief "context" beat before each feature demo

## Color and Typography

### Default Video Palette

```typescript
const PALETTE = {
  background: '#0f0f1a',        // Deep dark blue-black
  surface: '#1e1e2e',           // Slightly lighter for cards/frames
  primary: '#6366f1',           // Indigo accent
  secondary: '#f59e0b',         // Amber for highlights/callouts
  text: '#ffffff',              // White text
  textMuted: '#94a3b8',         // Slate for secondary text
  success: '#22c55e',           // Green for positive indicators
  gradient: 'linear-gradient(135deg, #6366f1, #8b5cf6)', // Purple gradient
}
```

### Typography

```typescript
// Use Google Fonts loaded via @remotion/google-fonts
import { loadFont } from '@remotion/google-fonts/Inter'

const { fontFamily } = loadFont()

const TYPOGRAPHY = {
  heading: { fontFamily, fontWeight: 800, fontSize: 64, letterSpacing: -2 },
  subheading: { fontFamily, fontWeight: 600, fontSize: 36 },
  body: { fontFamily, fontWeight: 400, fontSize: 24 },
  caption: { fontFamily, fontWeight: 500, fontSize: 18 },
  cta: { fontFamily, fontWeight: 700, fontSize: 48, letterSpacing: -1 },
}
```

## Composition Registration

```tsx
// src/Root.tsx
import { Composition } from 'remotion'

export const RemotionRoot: React.FC = () => {
  return (
    <>
      <Composition
        id="YouTubePromo"
        component={VideoComposition}
        durationInFrames={120 * 30}  // Calculated from voiceover
        fps={30}
        width={1920}
        height={1080}
        defaultProps={{
          voiceoverSrc: staticFile('voiceover/narration.mp3'),
          musicSrc: staticFile('music/background.mp3'),
          screenshots: [
            staticFile('screenshots/01-landing.png'),
            staticFile('screenshots/02-dashboard.png'),
            // ...
          ],
        }}
        calculateMetadata={calculateVideoMetadata}
      />

      <Composition
        id="TikTokPromo"
        component={TikTokComposition}
        durationInFrames={30 * 30}
        fps={30}
        width={1080}
        height={1920}
        defaultProps={{
          voiceoverSrc: staticFile('voiceover/narration-short.mp3'),
          musicSrc: staticFile('music/background.mp3'),
          screenshots: [
            staticFile('screenshots/mobile/01-landing.png'),
            // ...
          ],
        }}
        calculateMetadata={calculateVideoMetadata}
      />
    </>
  )
}
```

## Rendering

```bash
# Preview in Remotion Studio
npx remotion studio

# Render YouTube version
npx remotion render YouTubePromo out/youtube-promo.mp4

# Render TikTok version
npx remotion render TikTokPromo out/tiktok-promo.mp4

# Render with specific codec for quality
npx remotion render YouTubePromo out/youtube-promo.mp4 --codec h264 --crf 18
```
