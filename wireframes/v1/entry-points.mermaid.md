%% ElevenLabs-Bernie Entry Points
%% All ways to interact with and run the codebase
%% Version: v1

graph TB
    subgraph WebApplication["Web Application Entry Points"]
        direction TB

        subgraph DevServer["Development Server"]
            NPM_DEV["npm run dev<br/>Port 3000<br/>Turbo mode"]
        end

        subgraph ProdServer["Production Server"]
            NPM_BUILD["npm run build"]
            NPM_START["npm start<br/>Port 3000"]
        end

        subgraph PublicRoutes["Public Routes"]
            LANDING["/ (Landing Page)<br/>page.tsx"]
            SIGNIN["/app/sign-in<br/>Authentication"]
            SIGNUP["/app/sign-up<br/>Registration"]
        end

        subgraph ProtectedRoutes["Protected Routes (Auth Required)"]
            TTS["/app/speech-synthesis/text-to-speech<br/>StyleTTS2 UI"]
            STS["/app/speech-synthesis/speech-to-speech<br/>Seed-VC UI"]
            SFX_GEN["/app/sound-effects/generate<br/>Make-An-Audio UI"]
            SFX_HIST["/app/sound-effects/history<br/>Generation History"]
        end

        subgraph APIRoutes["API Routes"]
            AUTH_API["/api/auth/[...nextauth]<br/>NextAuth endpoints"]
            INNGEST_API["/api/inngest<br/>Job queue webhook"]
            MOCK_API["/api/mock/[...slug]<br/>Development mocks"]
        end
    end

    subgraph AIServices["AI Service Entry Points"]
        direction TB

        subgraph StyleTTS2API["StyleTTS2 API :8000"]
            STY_START["uvicorn api:app<br/>--host 0.0.0.0 --port 8000"]
            STY_GENERATE["POST /generate<br/>{text, target_voice}"]
            STY_VOICES["GET /voices<br/>List available voices"]
            STY_HEALTH["GET /health<br/>Service status"]
        end

        subgraph SeedVCAPI["Seed-VC API :8001"]
            VC_START["uvicorn api:app<br/>--host 0.0.0.0 --port 8000"]
            VC_CONVERT["POST /convert<br/>{source_audio_key, target_voice}"]
            VC_VOICES["GET /voices<br/>List available voices"]
            VC_HEALTH["GET /health<br/>Service status"]
        end

        subgraph MakeAnAudioAPI["Make-An-Audio API :8002"]
            MAA_START["uvicorn api:app<br/>--host 0.0.0.0 --port 8000"]
            MAA_GENERATE["POST /generate<br/>{prompt}"]
            MAA_HEALTH["GET /health<br/>Service status"]
        end
    end

    subgraph Training["Training Entry Points"]
        direction TB

        subgraph StyleTTS2Train["StyleTTS2 Training"]
            STY_FIRST["python train_first.py<br/>--config_path ./Configs/config.yml"]
            STY_SECOND["python train_second.py<br/>--config_path ./Configs/config.yml"]
            STY_FT["python train_finetune.py<br/>--config_path ./Configs/config_ft.yml"]
        end

        subgraph SeedVCTrain["Seed-VC Training"]
            VC_TRAIN["python train.py<br/>Custom voice training"]
        end

        subgraph MakeAnAudioTrain["Make-An-Audio Training"]
            MAA_VAE["python main.py<br/>--base configs/train/vae.yaml<br/>-t --gpus 0,1,2,3,4,5,6,7"]
            MAA_DIFF["python main.py<br/>--base configs/train/diffusion.yaml<br/>-t --gpus 0,1,2,3,4,5,6,7"]
        end
    end

    subgraph Docker["Docker Entry Points"]
        direction TB

        COMPOSE["docker-compose up<br/>Start all services"]

        subgraph Containers["Individual Containers"]
            FE_CONT["elevenlabs-frontend<br/>Port 3000"]
            STY_CONT["styletts2-api<br/>Port 8000"]
            VC_CONT["seedvc-api<br/>Port 8001"]
            MAA_CONT["make-an-audio-api<br/>Port 8002"]
        end
    end

    subgraph Utility["Utility Entry Points"]
        direction TB

        PRISMA_GEN["npx prisma generate<br/>Generate Prisma client"]
        PRISMA_PUSH["npx prisma db push<br/>Sync schema to DB"]
        PRISMA_STUDIO["npx prisma studio<br/>Database GUI"]
        INNGEST_DEV["npx inngest-cli dev<br/>Local Inngest dashboard"]
    end

    subgraph GradioApps["Gradio UI Apps (Development)"]
        direction TB

        GRADIO_VC["python app_vc.py<br/>Voice conversion demo"]
        GRADIO_SVC["python app_svc.py<br/>Singing voice demo"]
        GRADIO_MAIN["python app.py<br/>Combined demo"]
    end

    %% Relationships
    NPM_DEV --> PublicRoutes
    NPM_START --> PublicRoutes
    PublicRoutes --> ProtectedRoutes
    ProtectedRoutes --> APIRoutes

    COMPOSE --> FE_CONT
    COMPOSE --> STY_CONT
    COMPOSE --> VC_CONT
    COMPOSE --> MAA_CONT

    STY_START --> STY_GENERATE
    STY_START --> STY_VOICES
    STY_START --> STY_HEALTH

    VC_START --> VC_CONVERT
    VC_START --> VC_VOICES
    VC_START --> VC_HEALTH

    MAA_START --> MAA_GENERATE
    MAA_START --> MAA_HEALTH

    style WebApplication fill:#e3f2fd
    style AIServices fill:#e8f5e9
    style Training fill:#fff3e0
    style Docker fill:#f3e5f5
    style Utility fill:#fce4ec
    style GradioApps fill:#f5f5f5
