# Repository Structure - Documentation Notes

## Overview

The ElevenLabs-Bernie repository is a **monorepo** containing a Next.js frontend application and three independent AI model backend services. This architecture enables separate development, testing, and deployment of each AI model while maintaining a unified user interface.

## Directory Organization

### Root Level Structure

```
ElevenLabs-Bernie/
├── elevenlabs-clone-frontend/   # Main web application
├── StyleTTS2/                   # Text-to-Speech service
├── seed-vc/                     # Voice Conversion service
├── Make-An-Audio/               # Sound Effects Generation
├── StyleTTS2FineTune/           # Fine-tuning utilities
├── docker-compose.yml           # Container orchestration
├── README.md                    # Project documentation
└── LICENSE                      # License file
```

## Key Directories Explained

### Frontend (`elevenlabs-clone-frontend/`)

The Next.js 15 application using the T3 Stack pattern:

| Directory | Purpose | Key Files |
|-----------|---------|-----------|
| `src/app/` | Next.js App Router pages | `page.tsx`, API routes |
| `src/app/api/` | Backend API endpoints | `auth/[...nextauth]`, `inngest` |
| `src/app/app/` | Protected application pages | TTS, voice conversion, SFX pages |
| `src/components/client/` | React UI components | `sidebar.tsx`, `playbar.tsx` |
| `src/actions/` | Server Actions | `generate-speech.ts` |
| `src/inngest/` | Job queue configuration | `client.ts`, `functions.ts` |
| `src/server/` | Server-side utilities | `auth/`, `db.ts` |
| `src/stores/` | Zustand state management | `audio-store.ts`, `voice-store.ts` |
| `prisma/` | Database schema | `schema.prisma` |

### StyleTTS2 (`StyleTTS2/`)

Text-to-Speech generation service:

| Directory | Purpose | Key Files |
|-----------|---------|-----------|
| Root | API and inference | `api.py`, `libri_inference.py`, `models.py` |
| `Modules/diffusion/` | Diffusion model components | `sampler.py`, `modules.py` |
| `Utils/ASR/` | Text alignment | Pre-trained aligner for 24kHz |
| `Utils/JDC/` | Pitch extraction | F0 extractor |
| `Utils/PLBERT/` | Phonetic embeddings | PL-BERT model |
| `Configs/` | Model configurations | `config.yml`, `config_ft.yml` |
| `Models/` | Pre-trained weights | `LJSpeech/`, `LibriTTS/` |

### Seed-VC (`seed-vc/`)

Voice conversion (voice cloning) service:

| Directory | Purpose | Key Files |
|-----------|---------|-----------|
| Root | API and inference | `api.py`, `inference.py`, `train.py` |
| `modules/bigvgan/` | Vocoder | High-quality audio synthesis |
| `modules/openvoice/` | Voice encoder | Style/voice embedding |
| `modules/campplus/` | Speaker embedding | DTDNN speaker identification |
| `configs/presets/` | Model presets | `config_dit_*.yml` |

### Make-An-Audio (`Make-An-Audio/`)

Text-to-audio generation service:

| Directory | Purpose | Key Files |
|-----------|---------|-----------|
| Root | API and generation | `api.py`, `gen_wav.py`, `main.py` |
| `ldm/models/diffusion/` | Latent diffusion | Model architecture |
| `ldm/modules/encoders/` | Text encoding | CLAP encoder |
| `vocoder/bigvgan/` | Audio synthesis | BigVGAN vocoder |
| `configs/train/` | Training configs | `vae.yaml`, `diffusion.yaml` |
| `configs/text_to_audio/` | Generation configs | Inference parameters |

## Configuration Files Reference

### Frontend Configuration

| File | Purpose |
|------|---------|
| `package.json` | Dependencies and scripts |
| `tsconfig.json` | TypeScript configuration |
| `tailwind.config.ts` | Tailwind CSS settings |
| `next.config.js` | Next.js configuration |
| `src/env.js` | Environment variable validation (Zod) |
| `prisma/schema.prisma` | Database schema |

### Backend Configuration

| Service | Config Files |
|---------|--------------|
| StyleTTS2 | `Configs/config.yml`, `config_ft.yml`, `config_libritts.yml` |
| Seed-VC | `configs/presets/config_dit_mel_seed_uvit_*.yml` |
| Make-An-Audio | `configs/text_to_audio/*.yaml`, `configs/train/*.yaml` |

## Code Organization Patterns

### Frontend Patterns

1. **App Router Structure**: Pages in `src/app/` with route groups
2. **Client Components**: All interactive UI in `src/components/client/`
3. **Server Actions**: Form submissions in `src/actions/`
4. **State Management**: Zustand stores for global state
5. **Type Safety**: TypeScript with Zod validation

### Backend Patterns

1. **FastAPI Servers**: Each model has `api.py` as entry point
2. **Inference Separation**: `inference.py` or similar for model logic
3. **Modular Architecture**: Shared components in `modules/` or `Utils/`
4. **Configuration Files**: YAML configs for model parameters
5. **Docker Support**: `Dockerfile.api` and `Dockerfile.train` per service

## Where to Find Things

### Adding New Features

| Task | Location |
|------|----------|
| New page | `src/app/app/{feature}/page.tsx` |
| New component | `src/components/client/{feature}/` |
| New API endpoint | `src/app/api/{endpoint}/route.ts` |
| New server action | `src/actions/{action}.ts` |
| New state store | `src/stores/{store}-store.ts` |
| Database change | `prisma/schema.prisma` |

### Modifying AI Models

| Task | Location |
|------|----------|
| TTS API changes | `StyleTTS2/api.py` |
| TTS model changes | `StyleTTS2/models.py`, `libri_inference.py` |
| Voice conversion API | `seed-vc/api.py` |
| Voice conversion model | `seed-vc/inference.py` |
| SFX generation API | `Make-An-Audio/api.py` |
| SFX generation model | `Make-An-Audio/gen_wav.py` |

### DevOps Changes

| Task | Location |
|------|----------|
| Container setup | `docker-compose.yml` |
| Service Dockerfile | `{service}/Dockerfile.api` |
| Training Dockerfile | `{service}/Dockerfile.train` |
| Dependencies | `{service}/requirements.txt` or `package.json` |

## Technical Debt & Complexity

### Areas of Complexity

1. **Multiple Inference Engines**: Each AI service has different model loading patterns
2. **Shared Module Dependencies**: `modules/bigvgan/` appears in multiple services
3. **Configuration Sprawl**: Multiple YAML config files across services
4. **Temporary File Management**: `/tmp` cleanup logic in each API

### Potential Improvements

1. Extract shared modules to a common library
2. Standardize config file structure across services
3. Implement shared logging and monitoring
4. Add health check endpoints consistently

## Related Diagrams

- [Architecture Overview](./architecture-overview.mermaid.md) - High-level system design
- [Component Map](./component-map.mermaid.md) - Detailed component relationships
- [Data Flow](./data-flow.mermaid.md) - Request/response cycles
- [Entry Points](./entry-points.mermaid.md) - All interaction methods
