# Architecture Overview - Documentation Notes

## Overview

ElevenLabs-Bernie implements a **distributed microservices architecture** for AI-powered audio generation. The system separates concerns across four main layers: Frontend, Job Queue, AI Services, and Data Storage.

## Architecture Layers

### 1. Client Layer

**Technology**: Web Browser with React

The user interacts with the system through a modern web interface built with React 18 and Next.js 15. The UI provides:
- Text input for TTS generation
- Drag-and-drop audio upload for voice conversion
- Prompt input for sound effect generation
- Audio playback controls
- Generation history

### 2. Frontend Layer (Next.js 15)

**Technology Stack**:
- Framework: Next.js 15.0.1 with App Router
- Language: TypeScript 5.5.3
- Styling: Tailwind CSS 3.4.3
- State: Zustand 5.0.3
- Forms: React Hook Form + Zod
- Auth: Auth.js (NextAuth 5.0 beta)
- ORM: Prisma 5.14.0

**Key Responsibilities**:
- Server-side rendering and routing
- User authentication and session management
- Server Actions for form submissions
- Event publishing to Inngest
- Database operations via Prisma

**Design Patterns**:
- **App Router**: File-based routing with layouts
- **Server Actions**: Direct database mutations without API routes
- **Zustand Stores**: Atomic state management for UI
- **Protected Routes**: Middleware-based auth checks

### 3. Job Queue Layer (Inngest)

**Technology**: Inngest 3.32.9

**Purpose**: Decouples user requests from long-running AI generation tasks

**Key Features**:
- Event-driven architecture (`generate.request` events)
- Automatic retries (2 attempts)
- Rate limiting (3 requests/min per user)
- Step-based execution for reliability
- Background processing without blocking UI

**Job Lifecycle**:
1. Server Action publishes event
2. Inngest receives and queues event
3. `aiGenerationFunction` processes job
4. Calls appropriate AI service
5. Updates database with result
6. Deducts user credits

### 4. AI Services Layer (FastAPI)

**Technology**: FastAPI with Uvicorn

Three independent microservices, each with GPU support:

#### StyleTTS2 Service (Port 8000)
- **Model**: Diffusion-based TTS with style control
- **Input**: Text + target voice
- **Output**: Generated speech audio
- **Features**: Text chunking, silence insertion, style transfer

#### Seed-VC Service (Port 8001)
- **Model**: DiT + CAMPPlus + BigVGAN
- **Input**: Source audio S3 key + target voice
- **Output**: Converted voice audio
- **Features**: Speaker embedding extraction, F0 conditioning

#### Make-An-Audio Service (Port 8002)
- **Model**: Latent Diffusion + CLAP + BigVGAN
- **Input**: Text prompt
- **Output**: Generated audio (10 seconds)
- **Features**: CLAP text encoding, DDIM sampling

### 5. Data & Storage Layer

#### Database (SQLite/MySQL/PostgreSQL)

**Tables**:
- `User`: Authentication, profile, credits
- `Session`: NextAuth session tokens
- `Account`: OAuth provider accounts
- `GeneratedAudioClip`: Audio history and metadata
- `VerificationToken`: Email verification

**Design Decisions**:
- SQLite for development simplicity
- Prisma for type-safe database access
- Flexible provider support for production scaling

#### AWS S3

**Bucket**: `elevenlabs-clone`

**Prefixes**:
- `styletts2-output/`: TTS generated audio
- `seedvc-outputs/`: Voice conversion audio
- `make-an-audio-outputs/`: SFX generated audio

**Access Pattern**:
- Upload: Direct from AI services via boto3
- Download: Presigned URLs (1-hour expiry)

## Key Architectural Decisions

### 1. Microservices vs Monolith

**Decision**: Separate AI models into independent services

**Rationale**:
- Each model has different GPU memory requirements
- Independent scaling based on demand
- Isolated failures (one model down doesn't affect others)
- Different deployment cycles for model updates

### 2. Job Queue Architecture

**Decision**: Use Inngest for async job processing

**Rationale**:
- AI generation takes 5-30 seconds
- Users shouldn't wait for blocking requests
- Built-in retry logic for transient failures
- Rate limiting to prevent abuse
- Step-based execution for reliability

### 3. State Management

**Decision**: Zustand for client state, database for persistent state

**Rationale**:
- Zustand: Lightweight, TypeScript-friendly, no boilerplate
- Database: Source of truth for credits, history
- No Redux complexity for this use case

### 4. Authentication

**Decision**: NextAuth with Credentials provider

**Rationale**:
- Simple email/password for MVP
- Extensible to OAuth providers later
- JWT sessions for stateless auth
- Prisma adapter for database storage

### 5. Storage

**Decision**: AWS S3 for audio files

**Rationale**:
- Scalable object storage
- Presigned URLs for secure access
- No file system management on servers
- CDN-ready for production

## Request Flow Example

### Text-to-Speech Generation

```
1. User enters text and selects voice in UI
2. Form submission triggers Server Action
3. Server Action creates GeneratedAudioClip record
4. Server Action publishes "generate.request" event
5. Inngest receives event and calls aiGenerationFunction
6. Function fetches clip metadata from database
7. Function calls StyleTTS2 API with text/voice
8. StyleTTS2 generates audio and uploads to S3
9. Function updates clip record with S3 key
10. Function deducts 50 credits from user
11. UI polls for completion via generationStatus
12. UI fetches presigned URL and plays audio
```

## Scalability Considerations

### Current Limitations

1. **Single Instance**: Each AI service runs as single container
2. **SQLite**: Development database doesn't scale horizontally
3. **Rate Limiting**: 3 req/min per user is conservative

### Scaling Strategies

1. **Horizontal AI Service Scaling**: Add load balancer + multiple GPU instances
2. **Database Migration**: Switch to managed PostgreSQL/MySQL
3. **CDN**: CloudFront for audio file delivery
4. **Queue Scaling**: Inngest handles scaling automatically
5. **Caching**: Redis for session storage, voice configs

## Security Architecture

### Authentication
- Password hashing: bcryptjs (10 rounds)
- Session tokens: JWT with secret
- Protected routes: Middleware checks

### API Security
- Backend API keys: Bearer token authentication
- S3 access: IAM roles and presigned URLs
- CORS: Configured per service

### Data Protection
- Passwords hashed before storage
- Sensitive config in environment variables
- Private S3 bucket with controlled access

## Related Diagrams

- [Repository Structure](./repo-structure.mermaid.md) - Directory organization
- [Component Map](./component-map.mermaid.md) - Module relationships
- [Data Flow](./data-flow.mermaid.md) - Request/response cycles
- [Authentication](./authentication-authorization.mermaid.md) - Security flows
- [Deployment](./deployment-infrastructure.mermaid.md) - Infrastructure setup
