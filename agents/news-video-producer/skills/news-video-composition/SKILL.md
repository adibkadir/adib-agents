---
name: news-video-composition
description: Remotion-based video composition for TikTok-format (9:16, 1080x1920) news clips. Includes news-specific components (headline cards, image panels, lower thirds, tickers, fact cards), Ken Burns and panning effects, word-by-word animated subtitles, multi-language rendering, and audio mixing. Use when composing the final video from research, scripts, voiceover, and images.
---

# News Video Composition

## When to Use

Use this skill whenever you need to:
- Compose a vertical (9:16) news video from narration audio, images, and caption data
- Build news-specific visual components (headlines, tickers, lower thirds, fact cards)
- Animate images with Ken Burns, panning, and zoom effects
- Render word-by-word animated subtitles in multiple languages
- Mix voiceover audio with background music
- Render separate MP4 files per language

## Composition Setup

```
Resolution:  1080 x 1920 (9:16 vertical)
FPS:         30
Codec:       H.264 (CRF 18)
Audio:       Voiceover layer + optional background music
Subtitles:   Word-by-word highlighting (bottom-center, TikTok safe zone)
```

## Composition Registry

Register one composition per language in `src/Root.tsx`:

```tsx
import { Composition } from 'remotion'
import { NewsVideo, type NewsVideoProps } from './NewsVideo'
import { calculateNewsMetadata } from './lib/timing'
import { staticFile } from 'remotion'

export const RemotionRoot: React.FC = () => (
  <>
    <Composition
      id="NewsClip-en"
      component={NewsVideo}
      width={1080}
      height={1920}
      fps={30}
      durationInFrames={60 * 30} // Placeholder, overridden by calculateMetadata
      defaultProps={{
        language: 'en',
        voiceoverSrc: staticFile('voiceover/narration-en.mp3'),
        captionsSrc: staticFile('captions/captions-en.json'),
        musicSrc: staticFile('music/background.mp3'),
        scenes: [], // Populated from script parsing
      }}
      calculateMetadata={calculateNewsMetadata}
    />

    <Composition
      id="NewsClip-ar"
      component={NewsVideo}
      width={1080}
      height={1920}
      fps={30}
      durationInFrames={60 * 30}
      defaultProps={{
        language: 'ar',
        voiceoverSrc: staticFile('voiceover/narration-ar.mp3'),
        captionsSrc: staticFile('captions/captions-ar.json'),
        musicSrc: staticFile('music/background.mp3'),
        scenes: [],
      }}
      calculateMetadata={calculateNewsMetadata}
    />
  </>
)
```

## Dynamic Duration Calculation

```typescript
import { type CalculateMetadataFunction } from 'remotion'
import { getAudioDurationInSeconds } from '@remotion/media-utils'

export interface NewsVideoProps {
  language: string
  voiceoverSrc: string
  captionsSrc: string
  musicSrc?: string
  scenes: SceneConfig[]
}

interface SceneConfig {
  type: 'hook' | 'context' | 'detail' | 'quote' | 'wrap-up' | 'outro'
  text: string
  image: string
  durationInFrames: number
}

export const calculateNewsMetadata: CalculateMetadataFunction<NewsVideoProps> = async ({
  props,
}) => {
  const voiceoverDuration = await getAudioDurationInSeconds(props.voiceoverSrc)

  const introPadding = 1.5
  const outroPadding = 3
  const totalDuration = introPadding + voiceoverDuration + outroPadding

  return {
    durationInFrames: Math.ceil(totalDuration * 30),
    fps: 30,
    width: 1080,
    height: 1920,
  }
}
```

## Main Composition

```tsx
import {
  AbsoluteFill,
  Audio,
  Sequence,
  useCurrentFrame,
  useVideoConfig,
  interpolate,
} from 'remotion'
import { TransitionSeries, linearTiming } from '@remotion/transitions'
import { fade } from '@remotion/transitions/fade'
import { slide } from '@remotion/transitions/slide'
import { wipe } from '@remotion/transitions/wipe'
import { SubtitleTrack } from './components/SubtitleTrack'
import { ProgressIndicator } from './components/ProgressIndicator'

export const NewsVideo: React.FC<NewsVideoProps> = (props) => {
  const { fps, durationInFrames } = useVideoConfig()
  const frame = useCurrentFrame()

  const voiceoverStartFrame = Math.round(1.5 * fps) // 1.5s intro

  // Background music volume: 15%, fade in/out over 2 seconds
  const musicVolume = interpolate(
    frame,
    [0, 2 * fps, durationInFrames - 2 * fps, durationInFrames],
    [0, 0.15, 0.15, 0],
    { extrapolateLeft: 'clamp', extrapolateRight: 'clamp' }
  )

  return (
    <AbsoluteFill style={{ backgroundColor: '#0a0a0f' }}>
      {/* Scene transitions */}
      <TransitionSeries>
        {props.scenes.map((scene, i) => (
          <React.Fragment key={i}>
            {i > 0 && (
              <TransitionSeries.Transition
                presentation={scene.type === 'quote' ? wipe({ direction: 'from-left' }) : fade()}
                timing={linearTiming({ durationInFrames: Math.round(fps * 0.5) })}
              />
            )}
            <TransitionSeries.Sequence durationInFrames={scene.durationInFrames}>
              <SceneRenderer scene={scene} language={props.language} />
            </TransitionSeries.Sequence>
          </React.Fragment>
        ))}
      </TransitionSeries>

      {/* Subtitles — overlaid on all scenes */}
      <AbsoluteFill style={{ zIndex: 10 }}>
        <Sequence from={voiceoverStartFrame}>
          <SubtitleTrack
            captionsSrc={props.captionsSrc}
            language={props.language}
          />
        </Sequence>
      </AbsoluteFill>

      {/* Progress indicator at top */}
      <AbsoluteFill style={{ zIndex: 11 }}>
        <ProgressIndicator />
      </AbsoluteFill>

      {/* Audio layers */}
      <Sequence from={voiceoverStartFrame}>
        <Audio src={props.voiceoverSrc} volume={1.0} />
      </Sequence>

      {props.musicSrc && (
        <Audio src={props.musicSrc} volume={musicVolume} loop />
      )}
    </AbsoluteFill>
  )
}
```

## Components

### HeadlineCard — Animated News Headline

```tsx
import { useCurrentFrame, useVideoConfig, spring, interpolate } from 'remotion'

export const HeadlineCard: React.FC<{
  text: string
  category?: string
  direction?: 'ltr' | 'rtl'
}> = ({ text, category, direction = 'ltr' }) => {
  const frame = useCurrentFrame()
  const { fps } = useVideoConfig()

  const progress = spring({ frame, fps, config: { damping: 15, stiffness: 100 } })
  const y = interpolate(progress, [0, 1], [60, 0])
  const opacity = interpolate(progress, [0, 1], [0, 1])

  // Category badge animation (delayed)
  const badgeOpacity = interpolate(frame, [10, 20], [0, 1], {
    extrapolateLeft: 'clamp',
    extrapolateRight: 'clamp',
  })

  return (
    <div
      style={{
        position: 'absolute',
        bottom: 400,
        left: 40,
        right: 40,
        transform: `translateY(${y}px)`,
        opacity,
        direction,
      }}
    >
      {category && (
        <div
          style={{
            opacity: badgeOpacity,
            display: 'inline-block',
            background: '#dc2626',
            color: '#fff',
            padding: '6px 16px',
            borderRadius: 4,
            fontSize: 20,
            fontWeight: 700,
            letterSpacing: 2,
            textTransform: 'uppercase',
            marginBottom: 12,
          }}
        >
          {category}
        </div>
      )}

      <div
        style={{
          fontSize: 52,
          fontWeight: 800,
          color: '#ffffff',
          lineHeight: 1.2,
          textShadow: '0 2px 20px rgba(0,0,0,0.8)',
          letterSpacing: -1,
        }}
      >
        {text}
      </div>
    </div>
  )
}
```

### ImagePanel — Full-Bleed Image with Ken Burns

```tsx
import { useCurrentFrame, useVideoConfig, interpolate } from 'remotion'
import { Img } from 'remotion'

type KenBurnsDirection =
  | 'zoom-in'
  | 'zoom-out'
  | 'pan-left'
  | 'pan-right'
  | 'pan-up'
  | 'pan-down'

export const ImagePanel: React.FC<{
  src: string
  direction?: KenBurnsDirection
  overlayOpacity?: number
}> = ({ src, direction = 'zoom-in', overlayOpacity = 0.4 }) => {
  const frame = useCurrentFrame()
  const { durationInFrames } = useVideoConfig()
  const progress = frame / durationInFrames

  const transforms: Record<KenBurnsDirection, string> = {
    'zoom-in': `scale(${1 + progress * 0.08})`,
    'zoom-out': `scale(${1.08 - progress * 0.08})`,
    'pan-left': `translateX(${-progress * 4}%) scale(1.05)`,
    'pan-right': `translateX(${progress * 4}%) scale(1.05)`,
    'pan-up': `translateY(${-progress * 4}%) scale(1.05)`,
    'pan-down': `translateY(${progress * 4}%) scale(1.05)`,
  }

  return (
    <div style={{ position: 'absolute', inset: 0, overflow: 'hidden' }}>
      <Img
        src={src}
        style={{
          width: '100%',
          height: '100%',
          objectFit: 'cover',
          transform: transforms[direction],
        }}
      />

      {/* Dark gradient overlay for text readability */}
      <div
        style={{
          position: 'absolute',
          inset: 0,
          background: `linear-gradient(
            180deg,
            rgba(0,0,0,${overlayOpacity * 0.3}) 0%,
            rgba(0,0,0,${overlayOpacity * 0.1}) 30%,
            rgba(0,0,0,${overlayOpacity * 0.5}) 60%,
            rgba(0,0,0,${overlayOpacity}) 100%
          )`,
        }}
      />
    </div>
  )
}
```

### LowerThird — Source Attribution Bar

```tsx
import { useCurrentFrame, useVideoConfig, interpolate } from 'remotion'

export const LowerThird: React.FC<{
  source: string
  date?: string
}> = ({ source, date }) => {
  const frame = useCurrentFrame()
  const { fps } = useVideoConfig()

  const slideIn = interpolate(frame, [10, 25], [-400, 0], {
    extrapolateLeft: 'clamp',
    extrapolateRight: 'clamp',
  })

  const opacity = interpolate(frame, [10, 20], [0, 1], {
    extrapolateLeft: 'clamp',
    extrapolateRight: 'clamp',
  })

  return (
    <div
      style={{
        position: 'absolute',
        bottom: 280,
        left: 0,
        right: 0,
        transform: `translateX(${slideIn}px)`,
        opacity,
      }}
    >
      <div
        style={{
          display: 'inline-flex',
          alignItems: 'center',
          gap: 12,
          background: 'rgba(220, 38, 38, 0.9)',
          padding: '10px 24px',
          borderRadius: '0 6px 6px 0',
        }}
      >
        <span style={{ color: '#fff', fontSize: 20, fontWeight: 700 }}>
          {source}
        </span>
        {date && (
          <span style={{ color: 'rgba(255,255,255,0.7)', fontSize: 16 }}>
            {date}
          </span>
        )}
      </div>
    </div>
  )
}
```

### SubtitleTrack — Word-by-Word Animated Subtitles

```tsx
import { useCurrentFrame, useVideoConfig } from 'remotion'
import { loadFont } from '@remotion/google-fonts/Inter'
import { loadFont as loadNotoArabic } from '@remotion/google-fonts/NotoSansArabic'

const { fontFamily: interFont } = loadFont()

interface CaptionWord {
  text: string
  startMs: number
  endMs: number
}

export const SubtitleTrack: React.FC<{
  captionsSrc: string
  language: string
}> = ({ captionsSrc, language }) => {
  const frame = useCurrentFrame()
  const { fps } = useVideoConfig()
  const currentTimeMs = (frame / fps) * 1000

  const isRtl = ['ar', 'he', 'fa', 'ur'].includes(language)

  // Load appropriate font
  if (isRtl) {
    const { fontFamily: arabicFont } = loadNotoArabic()
    // Use arabicFont
  }

  // Group words into lines (max 6 words per line for readability)
  const currentWords = getVisibleWords(captionsSrc, currentTimeMs)

  return (
    <div
      style={{
        position: 'absolute',
        bottom: 180, // TikTok safe zone
        left: 40,
        right: 40,
        display: 'flex',
        flexWrap: 'wrap',
        justifyContent: 'center',
        gap: 8,
        direction: isRtl ? 'rtl' : 'ltr',
      }}
    >
      {currentWords.map((word, i) => {
        const isActive =
          currentTimeMs >= word.startMs && currentTimeMs <= word.endMs

        return (
          <span
            key={`${word.startMs}-${i}`}
            style={{
              fontSize: 36,
              fontWeight: 800,
              fontFamily: isRtl ? 'Noto Sans Arabic' : interFont,
              color: isActive ? '#fbbf24' : '#ffffff',
              textShadow: '0 2px 8px rgba(0,0,0,0.9), 0 0 20px rgba(0,0,0,0.5)',
              transform: isActive ? 'scale(1.1)' : 'scale(1)',
              transition: 'none', // No CSS transitions in Remotion!
            }}
          >
            {word.text}
          </span>
        )
      })}
    </div>
  )
}

// Show a sliding window of words around the current time
function getVisibleWords(
  captionsSrc: string,
  currentTimeMs: number,
  windowSize: number = 8
): CaptionWord[] {
  // In practice, load and parse the JSON caption data
  // Return the window of words surrounding the current playback position
  // This ensures ~6-8 words are visible at a time
  return []
}
```

### BreakingBanner — Animated Category Banner

```tsx
import { useCurrentFrame, useVideoConfig, spring, interpolate } from 'remotion'

export const BreakingBanner: React.FC<{
  text: string
  color?: string
}> = ({ text, color = '#dc2626' }) => {
  const frame = useCurrentFrame()
  const { fps } = useVideoConfig()

  const slideDown = spring({
    frame,
    fps,
    config: { damping: 15, stiffness: 120 },
  })
  const y = interpolate(slideDown, [0, 1], [-80, 0])

  // Subtle pulse
  const pulse = interpolate(
    frame % (fps * 2),
    [0, fps, fps * 2],
    [1, 1.02, 1],
    { extrapolateRight: 'clamp' }
  )

  return (
    <div
      style={{
        position: 'absolute',
        top: 80,
        left: 0,
        right: 0,
        display: 'flex',
        justifyContent: 'center',
        transform: `translateY(${y}px) scale(${pulse})`,
      }}
    >
      <div
        style={{
          background: color,
          padding: '10px 32px',
          borderRadius: 6,
          boxShadow: `0 4px 20px ${color}66`,
        }}
      >
        <span
          style={{
            color: '#fff',
            fontSize: 22,
            fontWeight: 800,
            letterSpacing: 3,
            textTransform: 'uppercase',
          }}
        >
          {text}
        </span>
      </div>
    </div>
  )
}
```

### FactCard — Animated Statistics

```tsx
import { useCurrentFrame, useVideoConfig, spring, interpolate } from 'remotion'

export const FactCard: React.FC<{
  number: number
  label: string
  prefix?: string
  suffix?: string
}> = ({ number, label, prefix = '', suffix = '' }) => {
  const frame = useCurrentFrame()
  const { fps } = useVideoConfig()

  const scaleProgress = spring({ frame, fps, config: { damping: 12, stiffness: 80 } })
  const scale = interpolate(scaleProgress, [0, 1], [0.8, 1])
  const opacity = interpolate(scaleProgress, [0, 1], [0, 1])

  // Counter animation
  const countProgress = interpolate(frame, [10, 40], [0, 1], {
    extrapolateLeft: 'clamp',
    extrapolateRight: 'clamp',
  })
  const displayNumber = Math.round(number * countProgress)

  return (
    <div
      style={{
        position: 'absolute',
        inset: 0,
        display: 'flex',
        flexDirection: 'column',
        alignItems: 'center',
        justifyContent: 'center',
        transform: `scale(${scale})`,
        opacity,
      }}
    >
      <div
        style={{
          fontSize: 96,
          fontWeight: 900,
          color: '#fbbf24',
          letterSpacing: -3,
          textShadow: '0 4px 30px rgba(251, 191, 36, 0.3)',
        }}
      >
        {prefix}{displayNumber.toLocaleString()}{suffix}
      </div>
      <div
        style={{
          fontSize: 28,
          fontWeight: 600,
          color: '#94a3b8',
          marginTop: 8,
          textTransform: 'uppercase',
          letterSpacing: 2,
        }}
      >
        {label}
      </div>
    </div>
  )
}
```

### NewsTicker — Scrolling Ticker Bar

```tsx
import { useCurrentFrame, useVideoConfig } from 'remotion'

export const NewsTicker: React.FC<{
  items: string[]
  speed?: number
}> = ({ items, speed = 2 }) => {
  const frame = useCurrentFrame()
  const { width } = useVideoConfig()

  const tickerText = items.join('  \u2022  ') // Bullet separator
  const textWidth = tickerText.length * 14 // Approximate width
  const x = -(frame * speed) % (textWidth + width)

  return (
    <div
      style={{
        position: 'absolute',
        bottom: 120,
        left: 0,
        right: 0,
        height: 40,
        background: 'rgba(220, 38, 38, 0.85)',
        display: 'flex',
        alignItems: 'center',
        overflow: 'hidden',
      }}
    >
      <div
        style={{
          whiteSpace: 'nowrap',
          transform: `translateX(${x}px)`,
          color: '#fff',
          fontSize: 18,
          fontWeight: 600,
          letterSpacing: 0.5,
        }}
      >
        {tickerText}
      </div>
    </div>
  )
}
```

### ProgressIndicator — Story Progress Bar

```tsx
import { useCurrentFrame, useVideoConfig } from 'remotion'

export const ProgressIndicator: React.FC = () => {
  const frame = useCurrentFrame()
  const { durationInFrames } = useVideoConfig()

  const progress = (frame / durationInFrames) * 100

  return (
    <div
      style={{
        position: 'absolute',
        top: 0,
        left: 0,
        right: 0,
        height: 4,
        background: 'rgba(255,255,255,0.1)',
        zIndex: 20,
      }}
    >
      <div
        style={{
          height: '100%',
          width: `${progress}%`,
          background: 'linear-gradient(90deg, #dc2626, #f59e0b)',
          borderRadius: '0 2px 2px 0',
        }}
      />
    </div>
  )
}
```

## Layout Components

### SplitLayout — Image Top, Text Bottom

```tsx
import { AbsoluteFill } from 'remotion'

export const SplitLayout: React.FC<{
  imageContent: React.ReactNode
  textContent: React.ReactNode
  splitRatio?: number // 0-1, default 0.5
}> = ({ imageContent, textContent, splitRatio = 0.5 }) => (
  <AbsoluteFill>
    <div style={{ position: 'absolute', top: 0, left: 0, right: 0, height: `${splitRatio * 100}%` }}>
      {imageContent}
    </div>
    <div
      style={{
        position: 'absolute',
        bottom: 0,
        left: 0,
        right: 0,
        height: `${(1 - splitRatio) * 100}%`,
        background: '#0a0a0f',
        display: 'flex',
        flexDirection: 'column',
        justifyContent: 'center',
        padding: '0 40px',
      }}
    >
      {textContent}
    </div>
  </AbsoluteFill>
)
```

### OverlayLayout — Text on Full-Bleed Image

```tsx
import { AbsoluteFill } from 'remotion'

export const OverlayLayout: React.FC<{
  children: React.ReactNode
}> = ({ children }) => (
  <AbsoluteFill>
    {children}
    {/* Gradient overlay for text readability */}
    <div
      style={{
        position: 'absolute',
        inset: 0,
        background: `linear-gradient(
          180deg,
          rgba(0,0,0,0.2) 0%,
          rgba(0,0,0,0.05) 30%,
          rgba(0,0,0,0.4) 60%,
          rgba(0,0,0,0.85) 100%
        )`,
      }}
    />
  </AbsoluteFill>
)
```

### CardLayout — Centered Card on Gradient

```tsx
import { AbsoluteFill, useCurrentFrame, spring, useVideoConfig, interpolate } from 'remotion'

export const CardLayout: React.FC<{
  children: React.ReactNode
  gradientFrom?: string
  gradientTo?: string
}> = ({ children, gradientFrom = '#1a1a2e', gradientTo = '#0a0a0f' }) => {
  const frame = useCurrentFrame()
  const { fps } = useVideoConfig()

  const scale = spring({ frame, fps, config: { damping: 14, stiffness: 90 } })
  const cardScale = interpolate(scale, [0, 1], [0.9, 1])
  const opacity = interpolate(scale, [0, 1], [0, 1])

  return (
    <AbsoluteFill
      style={{
        background: `linear-gradient(180deg, ${gradientFrom}, ${gradientTo})`,
        display: 'flex',
        alignItems: 'center',
        justifyContent: 'center',
        padding: 40,
      }}
    >
      <div
        style={{
          transform: `scale(${cardScale})`,
          opacity,
          background: 'rgba(255,255,255,0.05)',
          borderRadius: 16,
          padding: 48,
          border: '1px solid rgba(255,255,255,0.1)',
          width: '100%',
          maxWidth: 900,
        }}
      >
        {children}
      </div>
    </AbsoluteFill>
  )
}
```

## Color Palette

```typescript
export const NEWS_PALETTE = {
  background: '#0a0a0f',
  surface: '#1a1a2e',
  breaking: '#dc2626',         // Red — breaking news, urgent
  accent: '#f59e0b',           // Amber — highlights, active subtitle word
  info: '#3b82f6',             // Blue — explainer, informational
  opinion: '#8b5cf6',          // Purple — opinion pieces
  text: '#ffffff',
  textMuted: '#94a3b8',
  subtitleHighlight: '#fbbf24', // Active word in subtitles
  gradientOverlay: 'linear-gradient(180deg, transparent 0%, rgba(0,0,0,0.8) 100%)',
} as const

// Map tone to primary color
export const TONE_COLORS: Record<string, string> = {
  'breaking-news': NEWS_PALETTE.breaking,
  'explainer': NEWS_PALETTE.info,
  'opinion': NEWS_PALETTE.opinion,
  'recap': NEWS_PALETTE.accent,
}
```

## Typography

```typescript
import { loadFont } from '@remotion/google-fonts/Inter'
import { loadFont as loadNotoArabic } from '@remotion/google-fonts/NotoSansArabic'
import { loadFont as loadNotoSC } from '@remotion/google-fonts/NotoSansSC'
import { loadFont as loadNotoJP } from '@remotion/google-fonts/NotoSansJP'

const { fontFamily: interFont } = loadFont()

export const TYPOGRAPHY = {
  headline: { fontFamily: interFont, fontWeight: 800, fontSize: 52, letterSpacing: -1, lineHeight: 1.2 },
  subheadline: { fontFamily: interFont, fontWeight: 600, fontSize: 32 },
  body: { fontFamily: interFont, fontWeight: 400, fontSize: 24, lineHeight: 1.5 },
  caption: { fontFamily: interFont, fontWeight: 500, fontSize: 18 },
  subtitle: { fontFamily: interFont, fontWeight: 800, fontSize: 36 },
  stat: { fontFamily: interFont, fontWeight: 900, fontSize: 96, letterSpacing: -3 },
  ticker: { fontFamily: interFont, fontWeight: 600, fontSize: 18 },
} as const

// Language-specific font loading
export const LANGUAGE_FONTS: Record<string, () => { fontFamily: string }> = {
  en: loadFont,
  ar: loadNotoArabic,
  zh: loadNotoSC,
  ja: loadNotoJP,
}

export function getFontFamily(language: string): string {
  const loader = LANGUAGE_FONTS[language] ?? loadFont
  const { fontFamily } = loader()
  return fontFamily
}
```

## RTL Language Support

For Arabic, Hebrew, Farsi, and Urdu:

```typescript
export function getTextDirection(language: string): 'ltr' | 'rtl' {
  return ['ar', 'he', 'fa', 'ur'].includes(language) ? 'rtl' : 'ltr'
}

// Apply to all text-containing components:
// style={{ direction: getTextDirection(language), textAlign: direction === 'rtl' ? 'right' : 'left' }}
```

## Rendering

```bash
# Preview in Remotion Studio
npx remotion studio

# Render English version
npx remotion render NewsClip-en out/news-en.mp4 --codec h264 --crf 18

# Render Arabic version
npx remotion render NewsClip-ar out/news-ar.mp4 --codec h264 --crf 18

# Render all languages
for lang in en ar fr es; do
  npx remotion render "NewsClip-$lang" "out/news-$lang.mp4" --codec h264 --crf 18
done

# Render with specific concurrency for faster encoding
npx remotion render NewsClip-en out/news-en.mp4 --codec h264 --crf 18 --concurrency 4
```
