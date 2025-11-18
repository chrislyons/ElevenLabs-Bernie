# Entry Points - Documentation Notes

## Overview

This document catalogs all ways to interact with the ElevenLabs-Bernie codebase, including web application routes, API endpoints, CLI commands, and development utilities.

## Web Application Entry Points

### Development Server

```bash
cd elevenlabs-clone-frontend
npm run dev
```

- **Port**: 3000
- **Mode**: Development with hot reload
- **Features**: Turbo mode enabled, source maps
- **URL**: http://localhost:3000

### Production Server

```bash
cd elevenlabs-clone-frontend
npm run build    # Build optimized bundle
npm start        # Start production server
```

- **Port**: 3000 (or $PORT)
- **Mode**: Production optimized
- **Output**: `.next/` directory

### Application Routes

#### Public Routes (No Auth Required)

| Route | File | Description |
|-------|------|-------------|
| `/` | `src/app/page.tsx` | Landing page |
| `/app/sign-in` | `src/app/app/sign-in/page.tsx` | Login form |
| `/app/sign-up` | `src/app/app/sign-up/page.tsx` | Registration form |

#### Protected Routes (Auth Required)

| Route | File | Description |
|-------|------|-------------|
| `/app/speech-synthesis/text-to-speech` | `src/app/app/speech-synthesis/text-to-speech/page.tsx` | StyleTTS2 TTS interface |
| `/app/speech-synthesis/speech-to-speech` | `src/app/app/speech-synthesis/speech-to-speech/page.tsx` | Seed-VC voice conversion |
| `/app/sound-effects/generate` | `src/app/app/sound-effects/generate/page.tsx` | Make-An-Audio generator |
| `/app/sound-effects/history` | `src/app/app/sound-effects/history/page.tsx` | Generation history |

### API Routes

#### Authentication

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/auth/signin` | GET/POST | NextAuth sign in |
| `/api/auth/signout` | POST | NextAuth sign out |
| `/api/auth/session` | GET | Get current session |
| `/api/auth/csrf` | GET | Get CSRF token |
| `/api/auth/callback/credentials` | POST | Credentials callback |

#### Job Queue

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/inngest` | POST | Inngest webhook handler |

#### Development Mocks

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/mock/[...slug]` | * | Mock API responses |

## AI Service Entry Points

### StyleTTS2 API (Port 8000)

**Start Command**:
```bash
cd StyleTTS2
uvicorn api:app --host 0.0.0.0 --port 8000
```

**With Docker**:
```bash
docker build -f Dockerfile.api -t styletts2-api .
docker run -p 8000:8000 --gpus all styletts2-api
```

#### Endpoints

| Endpoint | Method | Body | Response |
|----------|--------|------|----------|
| `/generate` | POST | `{text, target_voice}` | `{audio_url, s3_key}` |
| `/voices` | GET | - | `[{id, name}]` |
| `/health` | GET | - | `{status}` |

**Authentication**: Bearer token in `Authorization` header

**Example**:
```bash
curl -X POST http://localhost:8000/generate \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"text": "Hello world", "target_voice": "andreas"}'
```

---

### Seed-VC API (Port 8001)

**Start Command**:
```bash
cd seed-vc
uvicorn api:app --host 0.0.0.0 --port 8000
```

**Docker Compose Port**: 8001

#### Endpoints

| Endpoint | Method | Body | Response |
|----------|--------|------|----------|
| `/convert` | POST | `{source_audio_key, target_voice}` | `{audio_url, s3_key}` |
| `/voices` | GET | - | `[{id, name}]` |
| `/health` | GET | - | `{status}` |

**Example**:
```bash
curl -X POST http://localhost:8001/convert \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"source_audio_key": "uploads/audio.wav", "target_voice": "andreas"}'
```

---

### Make-An-Audio API (Port 8002)

**Start Command**:
```bash
cd Make-An-Audio
uvicorn api:app --host 0.0.0.0 --port 8000
```

**Docker Compose Port**: 8002

#### Endpoints

| Endpoint | Method | Body | Response |
|----------|--------|------|----------|
| `/generate` | POST | `{prompt}` | `{audio_url, s3_key}` |
| `/health` | GET | - | `{status}` |

**Example**:
```bash
curl -X POST http://localhost:8002/generate \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"prompt": "dog barking in the distance"}'
```

## Training Entry Points

### StyleTTS2 Training

**First Stage** (duration and acoustic models):
```bash
cd StyleTTS2
python train_first.py --config_path ./Configs/config.yml
```

**Second Stage** (SLM joint training):
```bash
python train_second.py --config_path ./Configs/config.yml
```

**Fine-tuning** (single speaker):
```bash
python train_finetune.py --config_path ./Configs/config_ft.yml
```

**Multi-speaker** (LibriTTS):
```bash
python train_first.py --config_path ./Configs/config_libritts.yml
```

---

### Seed-VC Training

```bash
cd seed-vc
python train.py
```

Configure via YAML files in `configs/presets/`

---

### Make-An-Audio Training

**VAE Training**:
```bash
cd Make-An-Audio
python main.py --base configs/train/vae.yaml -t --gpus 0,1,2,3,4,5,6,7
```

**Diffusion Training**:
```bash
python main.py --base configs/train/diffusion.yaml -t --gpus 0,1,2,3,4,5,6,7
```

## Docker Entry Points

### Docker Compose (All Services)

```bash
docker-compose up
```

**Services Started**:
- `elevenlabs-frontend`: Port 3000
- `styletts2-api`: Port 8000
- `seedvc-api`: Port 8001
- `make-an-audio-api`: Port 8002

### Individual Container Builds

**Frontend**:
```bash
cd elevenlabs-clone-frontend
docker build -t elevenlabs-frontend .
docker run -p 3000:3000 elevenlabs-frontend
```

**StyleTTS2**:
```bash
cd StyleTTS2
docker build -f Dockerfile.api -t styletts2-api .
docker run -p 8000:8000 --gpus all styletts2-api
```

**Seed-VC**:
```bash
cd seed-vc
docker build -f Dockerfile.api -t seedvc-api .
docker run -p 8001:8000 --gpus all seedvc-api
```

**Make-An-Audio**:
```bash
cd Make-An-Audio
docker build -f Dockerfile.api -t make-an-audio-api .
docker run -p 8002:8000 --gpus all make-an-audio-api
```

## Utility Entry Points

### Prisma Database Tools

```bash
cd elevenlabs-clone-frontend

# Generate Prisma client
npx prisma generate

# Push schema to database
npx prisma db push

# Open database GUI
npx prisma studio

# Run migrations
npx prisma migrate dev

# Reset database
npx prisma migrate reset
```

### Inngest Development

```bash
cd elevenlabs-clone-frontend
npx inngest-cli dev
```

Opens local Inngest dashboard at http://localhost:8288

### Type Checking & Linting

```bash
cd elevenlabs-clone-frontend

# Type check
npm run typecheck

# Lint
npm run lint

# Format
npm run format
```

## Gradio Demo Apps (Development)

Standalone demo UIs for testing models without the full stack:

### Seed-VC Demos

```bash
cd seed-vc

# Voice conversion demo
python app_vc.py

# Singing voice conversion
python app_svc.py

# Combined demo
python app.py
```

These launch Gradio web UIs for direct model testing.

## Environment-Specific Configurations

### Development

```bash
# Frontend
cp elevenlabs-clone-frontend/.env.example elevenlabs-clone-frontend/.env
# Edit .env with development values

# Start frontend
cd elevenlabs-clone-frontend && npm run dev

# Start AI services (in separate terminals)
cd StyleTTS2 && uvicorn api:app --reload --port 8000
cd seed-vc && uvicorn api:app --reload --port 8001
cd Make-An-Audio && uvicorn api:app --reload --port 8002
```

### Production

```bash
# Using Docker Compose
docker-compose -f docker-compose.yml up -d

# Or manual deployment
npm run build && npm start
```

## Healthcheck Endpoints

Monitor service health:

```bash
# Frontend
curl http://localhost:3000/api/health

# StyleTTS2
curl http://localhost:8000/health

# Seed-VC
curl http://localhost:8001/health

# Make-An-Audio
curl http://localhost:8002/health
```

## Common Workflows

### Starting Full Development Stack

```bash
# Terminal 1: Frontend
cd elevenlabs-clone-frontend
npm run dev

# Terminal 2: StyleTTS2
cd StyleTTS2
uvicorn api:app --host 0.0.0.0 --port 8000

# Terminal 3: Seed-VC
cd seed-vc
uvicorn api:app --host 0.0.0.0 --port 8001

# Terminal 4: Make-An-Audio
cd Make-An-Audio
uvicorn api:app --host 0.0.0.0 --port 8002

# Terminal 5: Inngest
cd elevenlabs-clone-frontend
npx inngest-cli dev
```

### Database Reset

```bash
cd elevenlabs-clone-frontend
npx prisma migrate reset
npx prisma db push
```

## Related Diagrams

- [Architecture Overview](./architecture-overview.mermaid.md) - System design
- [Data Flow](./data-flow.mermaid.md) - Request cycles
- [Deployment Infrastructure](./deployment-infrastructure.mermaid.md) - How it runs
