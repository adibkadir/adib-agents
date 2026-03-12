---
name: news-script-generation
description: LLM-powered news narration script generation using OpenRouter. Supports multiple tones (breaking news, explainer, opinion, recap), includes scene markers and timing cues, generates multi-language script variants, and produces voiceover audio via ElevenLabs with word-level caption timing via Whisper.cpp.
---

# News Script Generation

## When to Use

Use this skill whenever you need to:
- Generate a narration script from news research data (extracted facts, images, quotes)
- Support multiple tones: breaking news, explainer, opinion, recap
- Translate scripts to multiple languages while preserving structure
- Generate voiceover audio with ElevenLabs multilingual TTS
- Produce word-level caption timing with Whisper.cpp for animated subtitles

## Pipeline Overview

```
Research Data (extracted-facts.json)
  │
  ▼
Tone Selection + Duration Target
  │
  ▼
OpenRouter Script Generation (primary language)
  │
  ▼
OpenRouter Translation (per additional language)
  │
  ▼
User Review & Approval
  │
  ▼
ElevenLabs TTS Generation (per language)
  │
  ▼
Audio Duration Measurement
  │
  ▼
Whisper.cpp Transcription → Word-Level Captions (per language)
  │
  ▼
Scene Timing Calculation
```

## Step 1: Script Generation with OpenRouter

Generate the narration script using the LLM, informed by the research data:

```typescript
import OpenAI from 'openai'

const openrouter = new OpenAI({
  baseURL: 'https://openrouter.ai/api/v1',
  apiKey: process.env.OPENROUTER_API_KEY,
})

type NewsTone = 'breaking-news' | 'explainer' | 'opinion' | 'recap'

const TONE_PROMPTS: Record<NewsTone, string> = {
  'breaking-news':
    'Urgent, authoritative, fast-paced. Lead with the most important fact. ' +
    'Short punchy sentences. Create a sense of immediacy.',
  'explainer':
    'Clear, educational, methodical. Break down complex topics for a general audience. ' +
    'Use analogies and simple language. Build understanding step by step.',
  'opinion':
    'Conversational, opinionated, engaging. Take a clear stance on the issue. ' +
    'Use rhetorical questions. Challenge the audience to think.',
  'recap':
    'Concise, comprehensive, neutral. Cover all key developments without editorializing. ' +
    'Chronological or importance-ordered. Factual and balanced.',
}

interface ScriptRequest {
  researchData: {
    topicSummary: string[]
    sources: { name: string; url: string; headline: string }[]
    availableImages: { path: string; caption?: string }[]
    bestQuotes: { text: string; attribution: string; source: string }[]
  }
  tone: NewsTone
  duration: number // Target duration in seconds
  language: string // Primary language code
}

async function generateScript(request: ScriptRequest): Promise<string> {
  const wordsPerSecond = 2.5
  const targetWordCount = Math.round(request.duration * wordsPerSecond)

  const imageList = request.researchData.availableImages
    .map((img) => `- ${img.path}${img.caption ? ` (${img.caption})` : ''}`)
    .join('\n')

  const quoteList = request.researchData.bestQuotes
    .map((q) => `- "${q.text}" — ${q.attribution} (${q.source})`)
    .join('\n')

  const response = await openrouter.chat.completions.create({
    model: 'anthropic/claude-sonnet-4-20250514',
    messages: [
      {
        role: 'system',
        content: `You are a professional news video script writer for TikTok/Reels vertical videos.

Write narration scripts that are:
- ${TONE_PROMPTS[request.tone]}
- Paced for narration at ~2.5 words/second
- Structured with [SCENE: name] markers for video editing
- Each scene references an [IMAGE: filename] from the available images
- Include [PAUSE: Xs] markers for dramatic beats
- Front-loaded with a hook in the first 3 seconds
- End with a brief wrap-up and source attribution

Scene types to use: hook, context, detail, quote, wrap-up, outro
Target word count: ~${targetWordCount} words for ${request.duration}s video.

Available images:
${imageList}

Available quotes:
${quoteList}

IMPORTANT: Every [SCENE:] block must have an [IMAGE:] tag referencing one of the available images.
Source names: ${request.researchData.sources.map((s) => s.name).join(', ')}`,
      },
      {
        role: 'user',
        content: `Write a ${request.duration}-second TikTok news narration script.

Key facts to cover:
${request.researchData.topicSummary.map((f) => `- ${f}`).join('\n')}

Source headlines:
${request.researchData.sources.map((s) => `- ${s.headline} (${s.name})`).join('\n')}

Output the script with [SCENE: name], [IMAGE: filename], and [PAUSE: Xs] markers.`,
      },
    ],
    temperature: 0.7,
    max_tokens: 2000,
  })

  return response.choices[0].message.content!
}
```

### Script Format

The LLM should produce scripts in this format:

```markdown
[SCENE: hook]
[IMAGE: article-01-hero.jpg]
A massive breakthrough in AI regulation just changed the game for every tech company on the planet.

[PAUSE: 0.5s]

[SCENE: context]
[IMAGE: article-02-hero.jpg]
The European Union officially passed the world's first comprehensive AI safety framework today, setting strict rules for how artificial intelligence can be developed and deployed.

[SCENE: detail]
[IMAGE: article-01-hero.jpg]
Key provisions include mandatory transparency for AI systems used in hiring, lending, and law enforcement. Companies using high-risk AI must now conduct impact assessments and register their systems in a public database.

[SCENE: quote]
[IMAGE: article-03-hero.jpg]
EU Commissioner Thierry Breton called it "a watershed moment for technology governance" — a statement that signals Europe's intent to lead global AI policy.

[PAUSE: 0.5s]

[SCENE: wrap-up]
[IMAGE: article-02-hero.jpg]
As this framework takes effect, tech companies worldwide are scrambling to comply. The question now: will the rest of the world follow Europe's lead?

Sources: Reuters, The Guardian, TechCrunch

[SCENE: outro]
Follow for more AI news updates.
```

## Step 2: Multi-Language Translation

Translate the script to each target language while preserving markers:

```typescript
async function translateScript(
  englishScript: string,
  targetLanguage: string,
  languageName: string
): Promise<string> {
  const response = await openrouter.chat.completions.create({
    model: 'anthropic/claude-sonnet-4-20250514',
    messages: [
      {
        role: 'system',
        content: `You are a professional translator specializing in broadcast news scripts.

Translate the following news narration script from English to ${languageName}.

CRITICAL RULES:
- Keep ALL [SCENE: name] markers exactly as-is (in English)
- Keep ALL [IMAGE: filename] markers exactly as-is (in English)
- Keep ALL [PAUSE: Xs] markers exactly as-is
- Translate ONLY the narration text
- Maintain the same tone, pacing, and emotional intensity
- Adapt idioms naturally — don't translate literally
- Keep source names (Reuters, The Guardian, etc.) as-is
- For RTL languages (Arabic, Hebrew): the text will be rendered RTL automatically`,
      },
      {
        role: 'user',
        content: `Translate this news script to ${languageName}:\n\n${englishScript}`,
      },
    ],
    temperature: 0.3,
    max_tokens: 2000,
  })

  return response.choices[0].message.content!
}

// Generate scripts for all languages
async function generateAllScripts(
  request: ScriptRequest,
  languages: { code: string; name: string }[]
): Promise<Map<string, string>> {
  const scripts = new Map<string, string>()

  // Generate primary language script
  const primaryScript = await generateScript(request)
  scripts.set(request.language, primaryScript)

  // Translate to additional languages
  for (const lang of languages) {
    if (lang.code === request.language) continue
    const translated = await translateScript(primaryScript, lang.code, lang.name)
    scripts.set(lang.code, translated)
    console.log(`Translated script to ${lang.name}`)
  }

  return scripts
}
```

## Step 3: Voice Selection

Map languages to appropriate ElevenLabs voices:

```typescript
interface VoiceConfig {
  voiceId: string
  name: string
  model: string
}

const LANGUAGE_VOICES: Record<string, VoiceConfig> = {
  en: { voiceId: 'pNInz6obpgDQGcFmaJgB', name: 'Adam', model: 'eleven_multilingual_v2' },
  ar: { voiceId: 'onwK4e9ZLuTAKqWW03F9', name: 'Daniel', model: 'eleven_multilingual_v2' },
  fr: { voiceId: 'XB0fDUnXU5powFXDhCwa', name: 'Charlotte', model: 'eleven_multilingual_v2' },
  es: { voiceId: 'EXAVITQu4vr4xnSDxMaL', name: 'Sarah', model: 'eleven_multilingual_v2' },
  de: { voiceId: 'TX3LPaxmHKxFdv7VOQHJ', name: 'Liam', model: 'eleven_multilingual_v2' },
  pt: { voiceId: 'pNInz6obpgDQGcFmaJgB', name: 'Adam', model: 'eleven_multilingual_v2' },
  zh: { voiceId: 'onwK4e9ZLuTAKqWW03F9', name: 'Daniel', model: 'eleven_multilingual_v2' },
  ja: { voiceId: 'EXAVITQu4vr4xnSDxMaL', name: 'Sarah', model: 'eleven_multilingual_v2' },
  hi: { voiceId: 'TX3LPaxmHKxFdv7VOQHJ', name: 'Liam', model: 'eleven_multilingual_v2' },
}

function getVoice(language: string, userPreference?: string): VoiceConfig {
  if (userPreference) {
    return {
      voiceId: userPreference,
      name: 'Custom',
      model: 'eleven_multilingual_v2',
    }
  }
  return LANGUAGE_VOICES[language] ?? LANGUAGE_VOICES['en']
}
```

## Step 4: ElevenLabs TTS Generation

Generate voiceover audio for each language:

```typescript
import { createWriteStream } from 'fs'
import { pipeline } from 'stream/promises'

interface VoiceoverResult {
  language: string
  audioPath: string
  durationSeconds: number
}

async function generateVoiceover(
  script: string,
  language: string,
  outputPath: string,
  apiKey: string,
  voiceOverride?: string
): Promise<VoiceoverResult> {
  const voice = getVoice(language, voiceOverride)

  // Strip scene/image markers from script for TTS
  const cleanScript = script
    .replace(/\[SCENE:[^\]]+\]/g, '')
    .replace(/\[IMAGE:[^\]]+\]/g, '')
    .replace(/\[PAUSE:[^\]]+\]/g, '...')
    .replace(/\n{2,}/g, '\n')
    .trim()

  const response = await fetch(
    `https://api.elevenlabs.io/v1/text-to-speech/${voice.voiceId}`,
    {
      method: 'POST',
      headers: {
        'xi-api-key': apiKey,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        text: cleanScript,
        model_id: voice.model,
        output_format: 'mp3_44100_128',
        voice_settings: {
          stability: 0.65,
          similarity_boost: 0.75,
          style: 0.15,
          speed: 1.0,
          use_speaker_boost: true,
        },
      }),
    }
  )

  if (!response.ok) {
    const error = await response.json()
    throw new Error(`ElevenLabs error (${language}): ${JSON.stringify(error)}`)
  }

  const fileStream = createWriteStream(outputPath)
  await pipeline(response.body as any, fileStream)

  const durationSeconds = await getAudioDuration(outputPath)
  console.log(`Generated voiceover [${language}]: ${outputPath} (${durationSeconds.toFixed(1)}s)`)

  return { language, audioPath: outputPath, durationSeconds }
}

// Generate voiceovers for all languages
async function generateAllVoiceovers(
  scripts: Map<string, string>,
  outputDir: string,
  apiKey: string,
  voiceOverrides?: Record<string, string>
): Promise<VoiceoverResult[]> {
  const results: VoiceoverResult[] = []

  for (const [lang, script] of scripts) {
    const outputPath = `${outputDir}/public/voiceover/narration-${lang}.mp3`
    const result = await generateVoiceover(
      script,
      lang,
      outputPath,
      apiKey,
      voiceOverrides?.[lang]
    )
    results.push(result)
  }

  return results
}
```

## Step 5: Word-Level Caption Transcription

Use Whisper.cpp to transcribe each voiceover for word-by-word subtitle timing:

```typescript
import {
  installWhisperCpp,
  transcribe,
  downloadWhisperModel,
} from '@remotion/install-whisper-cpp'
import path from 'path'

interface CaptionWord {
  text: string
  startMs: number
  endMs: number
  confidence: number
}

interface CaptionData {
  language: string
  words: CaptionWord[]
  durationMs: number
}

async function transcribeCaptions(
  audioPath: string,
  language: string,
  outputPath: string
): Promise<CaptionData> {
  // Install Whisper.cpp if not present
  const whisperPath = path.join(process.cwd(), '.whisper')
  await installWhisperCpp({ to: whisperPath, version: '1.5.5' })
  await downloadWhisperModel({ model: 'medium', folder: whisperPath })

  // Transcribe with word-level timestamps
  const result = await transcribe({
    inputPath: audioPath,
    whisperPath,
    model: 'medium',
    tokenLevelTimestamps: true,
    language: language === 'zh' ? 'zh' : language, // Map language codes
  })

  // Convert to our caption format
  const words: CaptionWord[] = result.transcription.flatMap((segment) =>
    (segment.tokens ?? []).map((token) => ({
      text: token.text.trim(),
      startMs: token.offsets.from,
      endMs: token.offsets.to,
      confidence: token.p,
    }))
  ).filter((w) => w.text.length > 0)

  const captionData: CaptionData = {
    language,
    words,
    durationMs: words.length > 0 ? words[words.length - 1].endMs : 0,
  }

  // Save caption data
  const { writeFile } = await import('fs/promises')
  await writeFile(outputPath, JSON.stringify(captionData, null, 2))

  console.log(
    `Transcribed captions [${language}]: ${words.length} words, ` +
    `${(captionData.durationMs / 1000).toFixed(1)}s`
  )

  return captionData
}
```

## Step 6: Scene Timing Calculation

Parse the script markers to calculate per-scene timing and map images:

```typescript
interface SceneTiming {
  scene: string
  text: string
  image: string
  startFrame: number
  durationInFrames: number
}

function calculateSceneTimings(
  script: string,
  totalDurationFrames: number,
  fps: number
): SceneTiming[] {
  const scenes: { scene: string; text: string; image: string; pauseAfter: number }[] = []
  let currentScene = ''
  let currentImage = ''
  let currentText = ''

  for (const line of script.split('\n')) {
    const sceneMatch = line.match(/\[SCENE:\s*(.+?)\]/)
    const imageMatch = line.match(/\[IMAGE:\s*(.+?)\]/)
    const pauseMatch = line.match(/\[PAUSE:\s*([\d.]+)s\]/)

    if (sceneMatch) {
      if (currentScene) {
        scenes.push({
          scene: currentScene,
          text: currentText.trim(),
          image: currentImage,
          pauseAfter: 0,
        })
      }
      currentScene = sceneMatch[1]
      currentText = ''
    } else if (imageMatch) {
      currentImage = imageMatch[1]
    } else if (pauseMatch) {
      if (currentScene) {
        scenes.push({
          scene: currentScene,
          text: currentText.trim(),
          image: currentImage,
          pauseAfter: parseFloat(pauseMatch[1]),
        })
        currentScene = ''
        currentText = ''
        currentImage = ''
      }
    } else {
      currentText += line + ' '
    }
  }

  // Push last scene
  if (currentScene) {
    scenes.push({
      scene: currentScene,
      text: currentText.trim(),
      image: currentImage,
      pauseAfter: 0,
    })
  }

  // Calculate proportional timing based on word count
  const totalWords = scenes.reduce(
    (sum, s) => sum + s.text.split(/\s+/).filter(Boolean).length,
    0
  )
  const totalPauseFrames = scenes.reduce((sum, s) => sum + s.pauseAfter * fps, 0)
  const availableFrames = totalDurationFrames - totalPauseFrames

  let currentFrame = 0
  return scenes.map((s) => {
    const wordCount = s.text.split(/\s+/).filter(Boolean).length
    const proportion = totalWords > 0 ? wordCount / totalWords : 1 / scenes.length
    const durationInFrames = Math.round(proportion * availableFrames + s.pauseAfter * fps)

    const timing: SceneTiming = {
      scene: s.scene,
      text: s.text,
      image: s.image,
      startFrame: currentFrame,
      durationInFrames,
    }
    currentFrame += durationInFrames
    return timing
  })
}
```

## Audio Duration Measurement

Use Remotion's media utils for frame-accurate duration:

```typescript
import { getAudioDurationInSeconds } from '@remotion/media-utils'

// In a Remotion context (calculateMetadata):
export const calculateNewsMetadata: CalculateMetadataFunction<NewsVideoProps> = async ({
  props,
}) => {
  const voiceoverDuration = await getAudioDurationInSeconds(props.voiceoverSrc)

  const introPadding = 1.5 // seconds before voiceover starts
  const outroPadding = 3 // seconds after voiceover ends
  const totalDuration = introPadding + voiceoverDuration + outroPadding

  return {
    durationInFrames: Math.ceil(totalDuration * 30),
    fps: 30,
    width: 1080,
    height: 1920,
  }
}
```

## Long Script Handling

For scripts over 5000 characters, segment and use context parameters:

```typescript
async function generateLongVoiceover(
  segments: string[],
  language: string,
  outputDir: string,
  apiKey: string,
  voiceOverride?: string
): Promise<string[]> {
  const voice = getVoice(language, voiceOverride)
  const audioParts: string[] = []

  for (let i = 0; i < segments.length; i++) {
    const partPath = `${outputDir}/narration-${language}-part${i}.mp3`

    await fetch(`https://api.elevenlabs.io/v1/text-to-speech/${voice.voiceId}`, {
      method: 'POST',
      headers: {
        'xi-api-key': apiKey,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        text: segments[i],
        model_id: voice.model,
        output_format: 'mp3_44100_128',
        // Surrounding context for natural prosody
        previous_text: i > 0 ? segments[i - 1].slice(-200) : undefined,
        next_text: i < segments.length - 1 ? segments[i + 1].slice(0, 200) : undefined,
      }),
    })

    audioParts.push(partPath)
  }

  return audioParts
}
```
