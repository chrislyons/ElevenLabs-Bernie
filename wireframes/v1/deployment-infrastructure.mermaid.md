%% ElevenLabs-Bernie Deployment Infrastructure
%% How the code runs in development and production
%% Version: v1

graph TB
    subgraph Development["Development Environment"]
        direction TB

        subgraph DevMachine["Developer Machine"]
            DEV_FE["Next.js Dev Server<br/>npm run dev<br/>Port 3000"]
            DEV_STY["StyleTTS2 API<br/>uvicorn --reload<br/>Port 8000"]
            DEV_VC["Seed-VC API<br/>uvicorn --reload<br/>Port 8001"]
            DEV_MAA["Make-An-Audio API<br/>uvicorn --reload<br/>Port 8002"]
            DEV_INN["Inngest Dev Server<br/>npx inngest-cli dev<br/>Port 8288"]
            DEV_DB["SQLite Database<br/>prisma/dev.db"]
        end
    end

    subgraph Production["Production Environment - AWS"]
        direction TB

        subgraph LoadBalancer["Load Balancer"]
            ALB["Application Load Balancer<br/>HTTPS Termination"]
        end

        subgraph EC2Instances["EC2 Instances"]
            subgraph FrontendInstance["Frontend Instance"]
                FE_DOCKER["Docker Container<br/>Next.js Production<br/>Port 3000"]
            end

            subgraph GPUInstance["GPU Instance (g4dn.xlarge+)"]
                GPU_STY["StyleTTS2 Container<br/>CUDA Enabled<br/>Port 8000"]
                GPU_VC["Seed-VC Container<br/>CUDA Enabled<br/>Port 8001"]
                GPU_MAA["Make-An-Audio Container<br/>CUDA Enabled<br/>Port 8002"]
            end
        end

        subgraph ManagedServices["Managed Services"]
            RDS["RDS MySQL/PostgreSQL<br/>Database"]
            S3["S3 Bucket<br/>elevenlabs-clone<br/>Audio Storage"]
            ECR["ECR Registry<br/>Docker Images"]
            SECRETS["Secrets Manager<br/>API Keys & Credentials"]
        end

        subgraph ExternalServices["External Services"]
            INNGEST_CLOUD["Inngest Cloud<br/>Job Queue"]
            HF["HuggingFace Hub<br/>Model Downloads"]
        end
    end

    subgraph CI_CD["CI/CD Pipeline"]
        direction LR
        GH["GitHub Repository"]
        GHA["GitHub Actions"]
        BUILD["Build & Test"]
        DEPLOY["Deploy to AWS"]
    end

    %% Development Connections
    DEV_FE --> DEV_DB
    DEV_FE --> DEV_INN
    DEV_INN --> DEV_STY
    DEV_INN --> DEV_VC
    DEV_INN --> DEV_MAA

    %% Production Connections
    ALB --> FE_DOCKER
    FE_DOCKER --> RDS
    FE_DOCKER --> INNGEST_CLOUD
    INNGEST_CLOUD --> GPU_STY
    INNGEST_CLOUD --> GPU_VC
    INNGEST_CLOUD --> GPU_MAA
    GPU_STY --> S3
    GPU_VC --> S3
    GPU_MAA --> S3
    GPU_VC --> HF

    %% CI/CD Flow
    GH --> GHA
    GHA --> BUILD
    BUILD --> ECR
    BUILD --> DEPLOY
    DEPLOY --> FrontendInstance
    DEPLOY --> GPUInstance

    style Development fill:#e3f2fd
    style Production fill:#e8f5e9
    style CI_CD fill:#fff3e0
