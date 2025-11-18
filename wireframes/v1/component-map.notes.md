# Component Map - Documentation Notes

## Overview

This document details all components in the ElevenLabs-Bernie system, their responsibilities, dependencies, and public APIs. Components are organized by layer: Frontend UI, State Management, Server-Side, and AI Services.

## Frontend UI Components

### Page Components

Located in `src/app/app/`

#### TextToSpeechPage
**File**: `speech-synthesis/text-to-speech/page.tsx`

**Responsibility**: Main interface for StyleTTS2 text-to-speech generation

**Child Components**:
- `TextToSpeechEditor` - Text input and settings
- `VoiceSelector` - Voice selection dropdown
- `HistoryPanel` - Previous generations
- `GenerateButton` - Trigger generation

**Dependencies**: VoiceStore, AudioStore

---

#### SpeechToSpeechPage
**File**: `speech-synthesis/speech-to-speech/page.tsx`

**Responsibility**: Interface for Seed-VC voice conversion

**Child Components**:
- `VoiceChanger` - Drag-and-drop audio upload
- `VoiceSelector` - Target voice selection
- `HistoryPanel` - Conversion history

**Dependencies**: VoiceStore, AudioStore

---

#### SoundEffectsPage
**File**: `sound-effects/generate/page.tsx`

**Responsibility**: Interface for Make-An-Audio generation

**Child Components**:
- Text prompt input
- `GenerateButton` - Trigger generation
- `HistoryPanel` - Generated sounds

**Dependencies**: AudioStore

### Shared UI Components

Located in `src/components/client/`

#### TextToSpeechEditor
**Responsibility**: Text input form with TTS settings

**Props**:
- `onSubmit: (text: string, voice: string) => void`
- `initialText?: string`

**Internal State**:
- `text: string` - Input text
- `voice: string` - Selected voice
- `settings: TTSSettings` - Generation parameters

**Methods**:
- `handleGenerate()` - Trigger generation via Server Action

---

#### VoiceChanger
**Responsibility**: Audio upload and voice conversion trigger

**Props**:
- `onUpload: (file: File) => void`
- `onConvert: (s3Key: string, voice: string) => void`

**Internal State**:
- `audioFile: File | null` - Uploaded audio
- `targetVoice: string` - Selected target voice
- `uploading: boolean` - Upload status

**Methods**:
- `handleDrop(files: File[])` - Handle drag-and-drop
- `handleConvert()` - Trigger conversion

---

#### HistoryPanel
**Responsibility**: Display generated audio history for a service

**Props**:
- `service: 'styletts2' | 'seedvc' | 'make-an-audio'`
- `clips: AudioClip[]`

**Internal State**:
- `selectedClip: AudioClip | null`

**Methods**:
- `loadHistory()` - Fetch clips from database
- `playClip(clip: AudioClip)` - Set clip in AudioStore

---

#### PlayBar
**Responsibility**: Global audio playback controls

**Props**:
- `audioUrl: string` - Presigned S3 URL
- `title: string` - Audio title/text

**Internal State**:
- `isPlaying: boolean`
- `progress: number` - Current position (seconds)
- `duration: number` - Total length

**Methods**:
- `play()` - Start playback
- `pause()` - Pause playback
- `seek(position: number)` - Jump to position
- `download()` - Download audio file

---

#### Sidebar
**Responsibility**: Navigation between services

**Props**:
- `currentRoute: string`

**Internal State**:
- `collapsed: boolean`

**Methods**:
- `navigate(route: string)` - Change page

---

#### VoiceSelector
**Responsibility**: Voice selection for each service

**Props**:
- `service: 'styletts2' | 'seedvc' | 'make-an-audio'`
- `onSelect: (voice: string) => void`

**Internal State**:
- `selectedVoice: string`
- `voices: Voice[]` - Available voices

**Methods**:
- `loadVoices()` - Fetch voices from API

---

#### GenerateButton
**Responsibility**: Generation trigger with credits display

**Props**:
- `onClick: () => void`
- `credits: number`

**Internal State**:
- `loading: boolean`

**Methods**:
- `handleClick()` - Trigger with loading state

## State Management

Located in `src/stores/`

### AudioStore
**File**: `audio-store.ts`

**Responsibility**: Manage global audio playback state

**State**:
```typescript
{
  currentAudio: AudioClip | null;
  isPlaying: boolean;
  progress: number;
  duration: number;
}
```

**Actions**:
- `setAudio(clip: AudioClip)` - Set current audio
- `setPlaying(playing: boolean)` - Update play state
- `setProgress(seconds: number)` - Update position

---

### VoiceStore
**File**: `voice-store.ts`

**Responsibility**: Track selected voice per service

**State**:
```typescript
{
  styletts2Voice: string;
  seedvcVoice: string;
  makeAnAudioVoice: string;
}
```

**Actions**:
- `setVoice(service: string, voice: string)` - Update selection
- `getVoice(service: string)` - Get current voice

---

### UIStore
**File**: `ui-store.ts`

**Responsibility**: UI state like sidebar, modals

**State**:
```typescript
{
  sidebarCollapsed: boolean;
  modalOpen: boolean;
}
```

**Actions**:
- `toggleSidebar()` - Toggle sidebar state
- `openModal()` / `closeModal()` - Modal control

## Server Actions

Located in `src/actions/generate-speech.ts`

### generateTextToSpeech
```typescript
async function generateTextToSpeech(
  text: string,
  voice: string
): Promise<{ audioId: string }>
```

**Steps**:
1. Create `GeneratedAudioClip` record
2. Send `generate.request` event to Inngest
3. Return `audioId` for polling

---

### generateSpeechToSpeech
```typescript
async function generateSpeechToSpeech(
  s3Key: string,
  voice: string
): Promise<{ audioId: string }>
```

**Steps**:
1. Create `GeneratedAudioClip` with `service: 'seedvc'`
2. Store `originalVoiceS3Key` for source audio
3. Send Inngest event
4. Return `audioId`

---

### generateSoundEffect
```typescript
async function generateSoundEffect(
  prompt: string
): Promise<{ audioId: string }>
```

**Steps**:
1. Create `GeneratedAudioClip` with `service: 'make-an-audio'`
2. Send Inngest event
3. Return `audioId`

---

### generationStatus
```typescript
async function generationStatus(
  clipId: string
): Promise<{ status: string; audioUrl?: string }>
```

**Returns**:
- `status: 'pending' | 'completed' | 'failed'`
- `audioUrl`: Presigned S3 URL when completed

---

### generateUploadUrl
```typescript
async function generateUploadUrl(): Promise<{
  uploadUrl: string;
  s3Key: string;
}>
```

**Returns**: Presigned PUT URL for audio upload

## Inngest Functions

Located in `src/inngest/`

### InngestClient
**File**: `client.ts`

**Methods**:
- `send(event: { name: string; data: object })` - Publish event

---

### aiGenerationFunction
**File**: `functions.ts`

**Event**: `generate.request`

**Configuration**:
- Retry: 2 attempts
- Throttle: 3 requests per minute per user

**Steps**:
1. **fetchClip** - Get clip metadata from database
2. **callAPI** - HTTP POST to appropriate AI service
3. **saveResult** - Update clip with S3 key
4. **deductCredits** - Subtract 50 credits from user

## Database Models

Located in `prisma/schema.prisma`

### User
**Fields**:
- `id: String` (CUID)
- `email: String` (unique)
- `password: String` (bcrypt hash)
- `name: String?`
- `credits: Int` (default: 10000)

**Relations**:
- `generatedAudioClips: GeneratedAudioClip[]`

---

### GeneratedAudioClip
**Fields**:
- `id: String` (CUID)
- `userId: String` (foreign key)
- `service: String` ('styletts2' | 'seedvc' | 'make-an-audio')
- `text: String?` (for TTS)
- `voice: String?`
- `originalVoiceS3Key: String?` (for voice conversion source)
- `s3Key: String?` (generated output)
- `failed: Boolean` (default: false)
- `createdAt: DateTime`

## AI Service APIs

### StyleTTS2API
**File**: `StyleTTS2/api.py`

**Endpoints**:

```python
POST /generate
Body: { text: str, target_voice: str }
Response: { audio_url: str, s3_key: str }

GET /voices
Response: [{ id: str, name: str }]

GET /health
Response: { status: str }
```

---

### SeedVCAPI
**File**: `seed-vc/api.py`

**Endpoints**:

```python
POST /convert
Body: { source_audio_key: str, target_voice: str }
Response: { audio_url: str, s3_key: str }

GET /voices
Response: [{ id: str, name: str }]

GET /health
Response: { status: str }
```

---

### MakeAnAudioAPI
**File**: `Make-An-Audio/api.py`

**Endpoints**:

```python
POST /generate
Body: { prompt: str }
Response: { audio_url: str, s3_key: str }

GET /health
Response: { status: str }
```

## AI Model Components

### StyleTTS2Inference
**File**: `StyleTTS2/libri_inference.py`

**Dependencies**:
- ASR (text aligner)
- JDC (pitch extractor)
- PLBERT (phonetic embeddings)
- Diffusion sampler

**Methods**:
- `inference(text, voice)` - Generate speech
- `computeStyle(audio)` - Extract style from reference

---

### VoiceConversion
**File**: `seed-vc/inference.py`

**Dependencies**:
- DiT model (diffusion transformer)
- CAMPPlus (speaker embedding)
- BigVGAN (vocoder)
- Optional: RMVPE (F0 extractor)

**Methods**:
- `convert(sourceAudio, targetVoice)` - Convert voice

---

### GenWav
**File**: `Make-An-Audio/gen_wav.py`

**Dependencies**:
- LatentDiffusion model
- CLAP encoder (text conditioning)
- BigVGAN (vocoder)

**Methods**:
- `generate(prompt)` - Generate audio from text

## Shared Utilities

### AudioManager
**File**: `src/utils/audio-manager.ts`

**Purpose**: Audio playback utilities

### S3Client
**Usage**: boto3 in Python, @aws-sdk/client-s3 in TypeScript

**Operations**: Upload, download, presigned URLs

## Component Dependencies Graph

```
Pages
  └── UI Components
        ├── Zustand Stores
        └── Server Actions
              └── Inngest
                    └── AI Service APIs
                          └── Model Components
```

## Related Diagrams

- [Architecture Overview](./architecture-overview.mermaid.md) - High-level system design
- [Data Flow](./data-flow.mermaid.md) - Request/response cycles
- [Entry Points](./entry-points.mermaid.md) - All interaction methods
