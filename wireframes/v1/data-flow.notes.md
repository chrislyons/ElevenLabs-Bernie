# Data Flow - Documentation Notes

## Overview

This document describes how data moves through the ElevenLabs-Bernie system during audio generation workflows. There are three main flows: Text-to-Speech, Voice Conversion, and Sound Effect generation.

## Common Data Flow Pattern

All three generation types follow the same high-level pattern:

```
User Input → Server Action → Database Record → Inngest Event
    → Job Handler → AI Service → S3 Upload → Database Update
    → Client Polling → Presigned URL → Audio Playback
```

## Text-to-Speech Flow

### 1. User Input
**Location**: `src/components/client/speech-synthesis/TextToSpeechEditor.tsx`

**Data**:
- `text: string` - User's input text
- `voice: string` - Selected voice ID ("andreas", "woman")

### 2. Server Action Call
**Location**: `src/actions/generate-speech.ts`

**Function**: `generateTextToSpeech(text, voice)`

**Operations**:
1. Get authenticated user session
2. Create `GeneratedAudioClip` record:
   ```typescript
   {
     userId: session.user.id,
     service: "styletts2",
     text: text,
     voice: voice,
     s3Key: null,
     failed: false
   }
   ```
3. Publish Inngest event:
   ```typescript
   {
     name: "generate.request",
     data: { clipId, service: "styletts2" }
   }
   ```
4. Return `{ audioId }` to client

### 3. Job Queue Processing
**Location**: `src/inngest/functions.ts`

**Function**: `aiGenerationFunction`

**Steps**:

1. **fetchClip** - Query database for clip metadata
2. **callAPI** - HTTP POST to StyleTTS2:
   ```typescript
   POST ${STYLETTS2_API_ROUTE}/generate
   Body: { text, target_voice }
   Headers: { Authorization: Bearer ${BACKEND_API_KEY} }
   ```
3. **saveResult** - Update clip with returned `s3Key`
4. **deductCredits** - Subtract 50 from user credits

### 4. AI Model Processing
**Location**: `StyleTTS2/api.py`, `StyleTTS2/libri_inference.py`

**Processing Steps**:
1. **Text Chunking**: Split text into max 125-character chunks
2. **Style Computation**: Extract style from reference voice audio
3. **Inference**:
   - Text → Phonemes → PLBERT embeddings
   - Duration prediction
   - Pitch extraction (JDC)
   - Diffusion sampling for mel-spectrogram
   - Style injection
4. **Vocoder**: Mel-spectrogram → Waveform
5. **Silence Insertion**: Add pauses between chunks
6. **S3 Upload**: Upload WAV to `styletts2-output/<uuid>.wav`

**Output**:
```python
{
  "audio_url": "https://s3.amazonaws.com/...",
  "s3_key": "styletts2-output/<uuid>.wav"
}
```

### 5. Client Polling
**Location**: `src/actions/generate-speech.ts`

**Function**: `generationStatus(clipId)`

**Operations**:
1. Query `GeneratedAudioClip` by ID
2. Check if `s3Key` is populated
3. Generate presigned URL (1-hour expiry)
4. Return status and URL

**Polling Pattern**: Client polls every 2-3 seconds until complete

### 6. Audio Playback
**Location**: `src/components/client/playbar.tsx`

**Data Flow**:
1. Receive presigned S3 URL
2. Load into HTML5 Audio element
3. Update AudioStore with playback state
4. Stream audio from S3

## Voice Conversion Flow

### Differences from TTS

1. **Source Audio Upload**:
   - Client requests presigned PUT URL
   - Direct upload to S3 from browser
   - Returns `s3Key` for source audio

2. **Service Parameter**:
   - `service: "seedvc"`
   - Uses `originalVoiceS3Key` field

3. **API Endpoint**:
   - `POST /convert` instead of `/generate`
   - Body: `{ source_audio_key, target_voice }`

4. **Model Processing**:
   - Downloads source audio from S3
   - Extracts speaker embedding with CAMPPlus
   - Converts using DiT diffusion model
   - Optional F0 (pitch) conditioning
   - BigVGAN vocoder

### Upload Flow Detail

```
1. generateUploadUrl()
   → S3 presigned PUT URL

2. Browser PUT to S3
   → Audio stored at generated s3Key

3. generateSpeechToSpeech(s3Key, voice)
   → Creates clip with originalVoiceS3Key = s3Key
```

## Sound Effect Generation Flow

### Differences from TTS

1. **Input Data**:
   - `prompt: string` instead of `text`
   - No voice selection

2. **Service Parameter**:
   - `service: "make-an-audio"`

3. **API Endpoint**:
   - `POST /generate`
   - Body: `{ prompt }`

4. **Model Processing**:
   - CLAP text encoder for conditioning
   - Latent diffusion (100 DDIM steps)
   - Fixed 10-second output duration
   - BigVGAN vocoder

## Data Transformations

### Text Processing

```
Raw Text → Chunks (≤125 chars) → Phonemes → Embeddings → Duration → Mel → Waveform
```

### Voice Conversion

```
Source Audio → Mel-spectrogram → Speaker Embedding → Converted Mel → Waveform
```

### Audio Generation

```
Text Prompt → CLAP Embedding → Latent Noise → Denoising → Mel → Waveform
```

## State Changes

### Database State Transitions

```
GeneratedAudioClip:
  Created: s3Key=null, failed=false
  Success: s3Key=<value>, failed=false
  Failure: s3Key=null, failed=true
```

### User Credits

```
Initial: 10,000 credits
Per generation: -50 credits
Deducted: After successful S3 upload
```

### Client State (Zustand)

```
AudioStore:
  Idle: currentAudio=null
  Loading: currentAudio={id, status: 'pending'}
  Playing: currentAudio={id, url, status: 'playing'}
```

## Error Handling

### Server Action Errors

- Auth failure → 401 response
- Database error → 500 response
- Inngest error → Event not queued

### Job Processing Errors

- API timeout → Retry (2 attempts)
- Model error → Mark clip as failed
- S3 error → Retry upload

### Client-Side Handling

- Poll timeout → Show error state
- Playback error → Retry with new presigned URL

## Caching

### Model Caching

- Models loaded once on startup
- Kept in memory during service lifetime
- HuggingFace models cached in `checkpoints/hf_cache/`

### URL Caching

- Presigned URLs expire after 1 hour
- Client should re-fetch if expired

## Rate Limiting

### Inngest Throttle

```typescript
throttle: {
  key: `user-${userId}`,
  count: 3,
  period: "1m"
}
```

- 3 requests per minute per user
- Applies across all services

## Performance Considerations

### Bottlenecks

1. **Model Inference**: 5-30 seconds depending on text length
2. **S3 Upload**: ~1-2 seconds for typical audio
3. **Database Queries**: Minimal latency with SQLite

### Optimization Opportunities

1. **Batch Processing**: Queue multiple chunks for TTS
2. **Caching**: Cache common voice embeddings
3. **CDN**: Use CloudFront for audio delivery
4. **Concurrent Upload**: Start upload before model completes

## Related Diagrams

- [Architecture Overview](./architecture-overview.mermaid.md) - System design
- [Component Map](./component-map.mermaid.md) - Module relationships
- [Entry Points](./entry-points.mermaid.md) - Interaction methods
- [Database Schema](./database-schema.mermaid.md) - Data models
