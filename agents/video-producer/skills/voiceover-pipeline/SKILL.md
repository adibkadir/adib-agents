---
name: voiceover-pipeline
description: End-to-end voiceover production pipeline using OpenRouter LLM for script generation and ElevenLabs TTS for professional narration audio. Handles script writing, voice selection, audio generation, and duration-based composition timing.
---

# Voiceover Pipeline

## When to Use

Use this skill whenever you need to:
- Generate a voiceover script from a creative brief and product information
- Convert a script to professional narration audio via ElevenLabs
- Calculate video composition duration from generated audio length
- Select and manage voice profiles for different video tones

## Pipeline Overview

```
Product Info (Firecrawl) + Creative Brief
  ↓
LLM Script Generation (OpenRouter)
  ↓
Script Review & Approval
  ↓
ElevenLabs TTS Generation
  ↓
Audio Duration Measurement
  ↓
Scene Timing Calculation
```

## Step 1: Script Generation with OpenRouter

Use the LLM to write a voiceover script informed by the product's actual content:

```typescript
import OpenAI from 'openai'

const openrouter = new OpenAI({
  baseURL: 'https://openrouter.ai/api/v1',
  apiKey: process.env.OPENROUTER_API_KEY,
})

interface ScriptRequest {
  productInfo: string     // Firecrawl-extracted content
  brief: {
    goal: string          // "Show the booking flow for cruise vacations"
    platform: 'youtube' | 'tiktok'
    duration: number      // Target duration in seconds
    tone: string          // "professional", "playful", "urgent", "conversational"
    audience: string      // "SaaS founders", "travel enthusiasts"
    keyPoints: string[]   // Must-mention features or benefits
  }
}

async function generateScript(request: ScriptRequest): Promise<string> {
  const wordsPerSecond = 2.5 // Average narration pace
  const targetWordCount = Math.round(request.brief.duration * wordsPerSecond)

  const response = await openrouter.chat.completions.create({
    model: 'anthropic/claude-sonnet-4-20250514',
    messages: [
      {
        role: 'system',
        content: `You are a professional video script writer for ${request.brief.platform} promotional videos.

Write voiceover scripts that are:
- Conversational and benefit-focused (not feature-listy)
- Paced for narration at ~2.5 words/second
- Structured with [SCENE: name] markers for video editing
- Include [PAUSE: Xs] markers for dramatic beats
- Front-loaded with a hook in the first 3 seconds
- End with a clear call-to-action

Target word count: ~${targetWordCount} words for ${request.brief.duration}s video.
Tone: ${request.brief.tone}
Audience: ${request.brief.audience}`,
      },
      {
        role: 'user',
        content: `Write a ${request.brief.duration}-second ${request.brief.platform} voiceover script.

Goal: ${request.brief.goal}

Key points to cover:
${request.brief.keyPoints.map((p) => `- ${p}`).join('\n')}

Product information:
${request.productInfo}

Output the script with [SCENE: name] markers and [PAUSE: Xs] markers.`,
      },
    ],
    temperature: 0.8,
    max_tokens: 2000,
  })

  return response.choices[0].message.content!
}
```

### Script Format

The LLM should produce scripts in this format:

```markdown
[SCENE: hook]
What if planning your dream cruise took 30 seconds instead of 30 hours?

[PAUSE: 1s]

[SCENE: problem]
Comparing prices across a dozen cruise lines, reading thousands of reviews,
trying to figure out which cabin is actually worth it...

[SCENE: solution-intro]
Meet Croozie — the only platform that brings every cruise line, every ship,
and every real review into one place.

[SCENE: walkthrough-search]
Just tell it where you want to go, and boom — every sailing, sorted by price,
filtered by your dates.

[SCENE: walkthrough-compare]
Tap any cruise to see real reviews from actual passengers,
side-by-side cabin comparisons, and port-by-port itineraries.

[SCENE: walkthrough-book]
Found the one? Book directly through the cruise line with the best price
Croozie found for you.

[PAUSE: 0.5s]

[SCENE: cta]
Start planning your next cruise at croozie dot com.
It's free, it's fast, and your dream vacation is closer than you think.
```

## Step 2: Voice Selection

```typescript
// List available ElevenLabs voices
async function listVoices(apiKey: string) {
  const response = await fetch('https://api.elevenlabs.io/v1/voices', {
    headers: { 'xi-api-key': apiKey },
  })
  const data = await response.json()
  return data.voices.map((v: any) => ({
    id: v.voice_id,
    name: v.name,
    category: v.category,
    labels: v.labels,
    previewUrl: v.preview_url,
  }))
}

// Recommended voices by tone
const VOICE_PRESETS: Record<string, { voiceId: string; name: string }> = {
  professional: { voiceId: 'pNInz6obpgDQGcFmaJgB', name: 'Adam' },
  conversational: { voiceId: 'EXAVITQu4vr4xnSDxMaL', name: 'Sarah' },
  energetic: { voiceId: 'TX3LPaxmHKxFdv7VOQHJ', name: 'Liam' },
  warm: { voiceId: 'XB0fDUnXU5powFXDhCwa', name: 'Charlotte' },
}
```

## Step 3: ElevenLabs TTS Generation

```typescript
import { createWriteStream } from 'fs'
import { pipeline } from 'stream/promises'

interface VoiceoverConfig {
  script: string
  voiceId: string
  apiKey: string
  outputPath: string
  model?: string
}

async function generateVoiceover(config: VoiceoverConfig): Promise<{
  audioPath: string
  durationSeconds: number
}> {
  // Strip scene markers and pauses from script for TTS
  const cleanScript = config.script
    .replace(/\[SCENE:[^\]]+\]/g, '')
    .replace(/\[PAUSE:[^\]]+\]/g, '...')  // Convert pauses to natural breaks
    .replace(/\n{2,}/g, '\n')
    .trim()

  const response = await fetch(
    `https://api.elevenlabs.io/v1/text-to-speech/${config.voiceId}`,
    {
      method: 'POST',
      headers: {
        'xi-api-key': config.apiKey,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        text: cleanScript,
        model_id: config.model ?? 'eleven_multilingual_v2',
        output_format: 'mp3_44100_128',
        voice_settings: {
          stability: 0.6,
          similarity_boost: 0.75,
          style: 0.2,
          speed: 1.0,
          use_speaker_boost: true,
        },
      }),
    }
  )

  if (!response.ok) {
    const error = await response.json()
    throw new Error(`ElevenLabs error: ${JSON.stringify(error)}`)
  }

  // Write audio to file
  const fileStream = createWriteStream(config.outputPath)
  await pipeline(response.body as any, fileStream)

  // Measure the audio duration
  const durationSeconds = await getAudioDuration(config.outputPath)

  console.log(
    `Generated voiceover: ${config.outputPath} (${durationSeconds.toFixed(1)}s)`
  )

  return { audioPath: config.outputPath, durationSeconds }
}
```

## Step 4: Audio Duration Measurement

Use Remotion's built-in audio utilities for frame-accurate duration:

```typescript
import { getAudioDurationInSeconds } from '@remotion/media-utils'

// In a Remotion context (calculateMetadata):
export const calculateMetadata: CalculateMetadataFunction<VideoProps> = async ({ props }) => {
  const voiceoverDuration = await getAudioDurationInSeconds(props.voiceoverSrc)

  // Add padding for intro/outro
  const introDuration = 2  // seconds
  const outroDuration = 3  // seconds
  const totalDuration = introDuration + voiceoverDuration + outroDuration

  return {
    durationInFrames: Math.ceil(totalDuration * props.fps),
    fps: props.fps,
    width: props.width,
    height: props.height,
  }
}
```

## Step 5: Scene Timing from Script

Parse the script markers to calculate per-scene timing:

```typescript
interface SceneTiming {
  scene: string
  text: string
  startFrame: number
  durationInFrames: number
}

function calculateSceneTimings(
  script: string,
  totalDurationFrames: number,
  fps: number
): SceneTiming[] {
  const scenes: { scene: string; text: string; pauseAfter: number }[] = []
  let currentScene = ''
  let currentText = ''

  for (const line of script.split('\n')) {
    const sceneMatch = line.match(/\[SCENE:\s*(.+?)\]/)
    const pauseMatch = line.match(/\[PAUSE:\s*([\d.]+)s\]/)

    if (sceneMatch) {
      if (currentScene) {
        scenes.push({ scene: currentScene, text: currentText.trim(), pauseAfter: 0 })
      }
      currentScene = sceneMatch[1]
      currentText = ''
    } else if (pauseMatch) {
      if (currentScene) {
        scenes.push({
          scene: currentScene,
          text: currentText.trim(),
          pauseAfter: parseFloat(pauseMatch[1]),
        })
        currentScene = ''
        currentText = ''
      }
    } else {
      currentText += line + ' '
    }
  }

  // Push last scene
  if (currentScene) {
    scenes.push({ scene: currentScene, text: currentText.trim(), pauseAfter: 0 })
  }

  // Calculate word counts and proportional timing
  const totalWords = scenes.reduce((sum, s) => sum + s.text.split(/\s+/).length, 0)
  const totalPauseFrames = scenes.reduce((sum, s) => sum + s.pauseAfter * fps, 0)
  const availableFrames = totalDurationFrames - totalPauseFrames

  let currentFrame = 0
  return scenes.map((s) => {
    const wordCount = s.text.split(/\s+/).length
    const proportion = wordCount / totalWords
    const durationInFrames = Math.round(proportion * availableFrames + s.pauseAfter * fps)

    const timing: SceneTiming = {
      scene: s.scene,
      text: s.text,
      startFrame: currentFrame,
      durationInFrames,
    }
    currentFrame += durationInFrames
    return timing
  })
}
```

## Long Script Handling

For scripts over 5000 characters, segment and use context parameters:

```typescript
async function generateLongVoiceover(
  segments: string[],
  config: Omit<VoiceoverConfig, 'script' | 'outputPath'>
) {
  const audioParts: string[] = []

  for (let i = 0; i < segments.length; i++) {
    const partPath = `${config.outputPath.replace('.mp3', '')}-part${i}.mp3`

    await fetch(`https://api.elevenlabs.io/v1/text-to-speech/${config.voiceId}`, {
      method: 'POST',
      headers: {
        'xi-api-key': config.apiKey,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        text: segments[i],
        model_id: 'eleven_multilingual_v2',
        output_format: 'mp3_44100_128',
        // Provide surrounding context for natural prosody across segments
        previous_text: i > 0 ? segments[i - 1].slice(-200) : undefined,
        next_text: i < segments.length - 1 ? segments[i + 1].slice(0, 200) : undefined,
      }),
    })

    audioParts.push(partPath)
  }

  // Concatenate parts using FFmpeg (bundled with Remotion)
  // bunx remotion ffmpeg -i "concat:part0.mp3|part1.mp3" -c copy output.mp3
  return audioParts
}
```

## Background Music Integration

```typescript
import { readdirSync } from 'fs'
import path from 'path'

interface MusicSelection {
  trackPath: string
  trackName: string
  durationSeconds: number
}

async function selectBackgroundMusic(
  musicFolderPath: string,
  targetDuration: number
): Promise<MusicSelection> {
  // List available tracks
  const tracks = readdirSync(musicFolderPath)
    .filter((f) => f.endsWith('.mp3') || f.endsWith('.wav'))
    .map((f) => path.join(musicFolderPath, f))

  if (tracks.length === 0) {
    throw new Error(`No audio files found in ${musicFolderPath}`)
  }

  // Measure each track's duration and pick the best fit
  const measured = await Promise.all(
    tracks.map(async (trackPath) => ({
      trackPath,
      trackName: path.basename(trackPath),
      durationSeconds: await getAudioDuration(trackPath),
    }))
  )

  // Prefer tracks longer than the video (so we can trim, not loop)
  const longerTracks = measured.filter((t) => t.durationSeconds >= targetDuration)

  if (longerTracks.length > 0) {
    // Pick the shortest one that's still long enough (least wasted audio)
    return longerTracks.sort((a, b) => a.durationSeconds - b.durationSeconds)[0]
  }

  // If no track is long enough, pick the longest and plan to loop
  return measured.sort((a, b) => b.durationSeconds - a.durationSeconds)[0]
}
```

## Audio Mixing in Remotion

```tsx
import { Audio, Sequence, useCurrentFrame, useVideoConfig, interpolate } from 'remotion'

export const AudioMix: React.FC<{
  voiceoverSrc: string
  musicSrc: string
}> = ({ voiceoverSrc, musicSrc }) => {
  const frame = useCurrentFrame()
  const { fps, durationInFrames } = useVideoConfig()

  // Fade music in over first 2 seconds, fade out over last 2 seconds
  const musicVolume = interpolate(
    frame,
    [0, 2 * fps, durationInFrames - 2 * fps, durationInFrames],
    [0, 0.15, 0.15, 0],
    { extrapolateLeft: 'clamp', extrapolateRight: 'clamp' }
  )

  return (
    <>
      {/* Voiceover — starts after intro (e.g., 2 seconds in) */}
      <Sequence from={2 * fps}>
        <Audio src={voiceoverSrc} volume={1.0} />
      </Sequence>

      {/* Background music — full duration, low volume */}
      <Audio src={musicSrc} volume={musicVolume} loop />
    </>
  )
}
```
