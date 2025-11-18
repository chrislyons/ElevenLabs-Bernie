%% ElevenLabs-Bernie Repository Structure
%% Complete directory tree visualization
%% Version: v1

graph TB
    Root["ElevenLabs-Bernie<br/>Root Repository"]

    subgraph Frontend["elevenlabs-clone-frontend/"]
        FE_SRC["src/"]
        FE_PRISMA["prisma/"]
        FE_PKG["package.json"]

        subgraph AppRouter["src/app/"]
            APP_API["api/<br/>NextAuth & Inngest"]
            APP_PAGES["app/<br/>Protected Pages"]
            APP_ROOT["page.tsx<br/>Landing Page"]
        end

        subgraph Components["src/components/client/"]
            COMP_TTS["speech-synthesis/<br/>TTS & Voice Change UI"]
            COMP_SFX["sound-effects/<br/>SFX Generator UI"]
            COMP_SHARED["sidebar.tsx<br/>playbar.tsx<br/>voice-selector.tsx"]
        end

        subgraph Actions["src/actions/"]
            ACT_GEN["generate-speech.ts<br/>Server Actions"]
        end

        subgraph Inngest["src/inngest/"]
            INN_CLIENT["client.ts"]
            INN_FUNCS["functions.ts<br/>Job Handlers"]
        end

        subgraph Server["src/server/"]
            SRV_AUTH["auth/<br/>NextAuth Config"]
            SRV_DB["db.ts<br/>Prisma Client"]
        end

        subgraph Stores["src/stores/"]
            STORE_AUDIO["audio-store.ts"]
            STORE_VOICE["voice-store.ts"]
            STORE_UI["ui-store.ts"]
        end

        subgraph PrismaDir["prisma/"]
            SCHEMA["schema.prisma<br/>Database Models"]
        end
    end

    subgraph StyleTTS2["StyleTTS2/"]
        STY_API["api.py<br/>FastAPI Server"]
        STY_INF["libri_inference.py<br/>TTS Inference"]
        STY_MOD["models.py<br/>Neural Networks"]

        subgraph StyModules["Modules/"]
            STY_DIFF["diffusion/<br/>Diffusion Sampler"]
        end

        subgraph StyUtils["Utils/"]
            STY_ASR["ASR/<br/>Text Aligner"]
            STY_JDC["JDC/<br/>Pitch Extractor"]
            STY_PLB["PLBERT/<br/>Embeddings"]
        end

        subgraph StyConfigs["Configs/"]
            STY_CFG["config.yml<br/>config_ft.yml<br/>config_libritts.yml"]
        end

        subgraph StyModels["Models/"]
            STY_LJ["LJSpeech/<br/>Single Speaker"]
            STY_LIB["LibriTTS/<br/>Multi Speaker"]
        end
    end

    subgraph SeedVC["seed-vc/"]
        VC_API["api.py<br/>FastAPI Server"]
        VC_INF["inference.py<br/>Voice Conversion"]
        VC_TRAIN["train.py"]

        subgraph VCModules["modules/"]
            VC_BIG["bigvgan/<br/>Vocoder"]
            VC_OPEN["openvoice/<br/>Encoder"]
            VC_CAMP["campplus/<br/>Speaker Embed"]
        end

        subgraph VCConfigs["configs/"]
            VC_PRESET["presets/<br/>config_dit_*.yml"]
        end
    end

    subgraph MakeAnAudio["Make-An-Audio/"]
        MAA_API["api.py<br/>FastAPI Server"]
        MAA_GEN["gen_wav.py<br/>Audio Generation"]
        MAA_MAIN["main.py<br/>Training"]

        subgraph LDM["ldm/"]
            LDM_DIFF["models/diffusion/<br/>Diffusion Architecture"]
            LDM_ENC["modules/encoders/<br/>CLAP Text Encoder"]
        end

        subgraph MAAVocoder["vocoder/"]
            MAA_BIG["bigvgan/<br/>Audio Synthesis"]
        end

        subgraph MAAConfigs["configs/"]
            MAA_TRAIN["train/<br/>vae.yaml<br/>diffusion.yaml"]
            MAA_TXT["text_to_audio/<br/>txt2audio_args.yaml"]
        end
    end

    subgraph DevOps["DevOps & Config"]
        DOCKER["docker-compose.yml<br/>Multi-container"]
        README["README.md"]
        LICENSE["LICENSE"]
    end

    Root --> Frontend
    Root --> StyleTTS2
    Root --> SeedVC
    Root --> MakeAnAudio
    Root --> DevOps

    FE_SRC --> AppRouter
    FE_SRC --> Components
    FE_SRC --> Actions
    FE_SRC --> Inngest
    FE_SRC --> Server
    FE_SRC --> Stores

    style Root fill:#e1f5fe
    style Frontend fill:#f3e5f5
    style StyleTTS2 fill:#fff3e0
    style SeedVC fill:#e8f5e9
    style MakeAnAudio fill:#fce4ec
    style DevOps fill:#f5f5f5
