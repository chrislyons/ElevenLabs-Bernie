%% ElevenLabs-Bernie Architecture Overview
%% High-level system design showing all components and interactions
%% Version: v1

graph TB
    subgraph Client["Client Layer"]
        Browser["Web Browser<br/>React UI"]
    end

    subgraph Frontend["Frontend Layer - Next.js 15"]
        NextApp["Next.js App Router<br/>Pages & API Routes"]

        subgraph UIComponents["UI Components"]
            TTS_UI["Text-to-Speech<br/>Editor & Controls"]
            VC_UI["Voice Changer<br/>Upload & Convert"]
            SFX_UI["Sound Effects<br/>Prompt & Generate"]
            PlayBar["PlayBar<br/>Audio Playback"]
        end

        subgraph StateManagement["State Management - Zustand"]
            AudioStore["Audio Store<br/>Playback State"]
            VoiceStore["Voice Store<br/>Selection State"]
            UIStore["UI Store<br/>Interface State"]
        end

        subgraph ServerSide["Server-Side"]
            ServerActions["Server Actions<br/>generateTextToSpeech<br/>generateSpeechToSpeech<br/>generateSoundEffect"]
            AuthConfig["NextAuth<br/>Credentials Provider"]
            PrismaClient["Prisma Client<br/>Database ORM"]
        end
    end

    subgraph JobQueue["Job Queue Layer - Inngest"]
        InngestClient["Inngest Client<br/>Event Publisher"]
        InngestHandler["aiGenerationFunction<br/>Job Processor"]
    end

    subgraph AIServices["AI Services Layer - FastAPI"]
        subgraph StyleTTS2Service["StyleTTS2 Service :8000"]
            STY_API["FastAPI Server"]
            STY_INF["TTS Inference Engine<br/>Diffusion + Vocoder"]
        end

        subgraph SeedVCService["Seed-VC Service :8001"]
            VC_API["FastAPI Server"]
            VC_INF["Voice Conversion<br/>DiT + CAMPPlus + BigVGAN"]
        end

        subgraph MakeAnAudioService["Make-An-Audio Service :8002"]
            MAA_API["FastAPI Server"]
            MAA_INF["Audio Generation<br/>LDM + CLAP + BigVGAN"]
        end
    end

    subgraph DataLayer["Data & Storage Layer"]
        subgraph Database["Database - SQLite/MySQL"]
            Users["Users Table<br/>Auth & Credits"]
            AudioClips["GeneratedAudioClip<br/>History & S3 Keys"]
            Sessions["Sessions<br/>Auth Tokens"]
        end

        subgraph CloudStorage["AWS S3"]
            S3Bucket["elevenlabs-clone<br/>Audio File Storage"]
        end
    end

    subgraph External["External Services"]
        HuggingFace["HuggingFace<br/>Pre-trained Models"]
    end

    %% Client to Frontend
    Browser -->|HTTP/WSS| NextApp

    %% Frontend Internal
    NextApp --> UIComponents
    UIComponents --> StateManagement
    UIComponents --> ServerActions

    %% Server Actions to Job Queue
    ServerActions -->|Create Record| PrismaClient
    ServerActions -->|Publish Event| InngestClient
    InngestClient -->|generate.request| InngestHandler

    %% Job Queue to AI Services
    InngestHandler -->|POST /generate| STY_API
    InngestHandler -->|POST /convert| VC_API
    InngestHandler -->|POST /generate| MAA_API

    %% AI Service Internal
    STY_API --> STY_INF
    VC_API --> VC_INF
    MAA_API --> MAA_INF

    %% Model Downloads
    VC_INF -.->|Download Checkpoints| HuggingFace

    %% Storage Operations
    InngestHandler -->|Update s3Key| PrismaClient
    STY_INF -->|Upload Audio| S3Bucket
    VC_INF -->|Upload Audio| S3Bucket
    MAA_INF -->|Upload Audio| S3Bucket
    VC_INF -->|Download Source| S3Bucket

    %% Auth Flow
    NextApp --> AuthConfig
    AuthConfig --> PrismaClient

    %% Database Connections
    PrismaClient --> Users
    PrismaClient --> AudioClips
    PrismaClient --> Sessions

    %% Playback Flow
    PlayBar -->|Presigned URL| S3Bucket

    style Client fill:#e3f2fd
    style Frontend fill:#f3e5f5
    style JobQueue fill:#fff8e1
    style AIServices fill:#e8f5e9
    style DataLayer fill:#fce4ec
    style External fill:#f5f5f5
