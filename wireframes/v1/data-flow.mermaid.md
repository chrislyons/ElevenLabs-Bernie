%% ElevenLabs-Bernie Data Flow
%% How data moves through the system for each generation type
%% Version: v1

sequenceDiagram
    autonumber
    participant User as User Browser
    participant UI as React UI
    participant SA as Server Action
    participant DB as Prisma/Database
    participant INN as Inngest Queue
    participant JOB as aiGenerationFunction
    participant API as AI Service API
    participant MODEL as AI Model
    participant S3 as AWS S3

    Note over User,S3: Text-to-Speech Generation Flow

    User->>UI: Enter text & select voice
    UI->>SA: generateTextToSpeech(text, voice)
    SA->>DB: Create GeneratedAudioClip
    DB-->>SA: Return audioId
    SA->>INN: send("generate.request", {clipId, service: "styletts2"})
    SA-->>UI: Return {audioId}
    UI->>UI: Start polling generationStatus()

    INN->>JOB: Trigger aiGenerationFunction
    JOB->>DB: Fetch clip metadata
    DB-->>JOB: Return clip {text, voice}
    JOB->>API: POST /generate {text, target_voice}
    API->>MODEL: StyleTTS2Inference.inference()
    MODEL->>MODEL: Text chunking (max 125 chars)
    MODEL->>MODEL: Compute style from voice reference
    MODEL->>MODEL: Diffusion sampling
    MODEL->>MODEL: Vocoder to waveform
    MODEL->>S3: Upload WAV to styletts2-output/
    S3-->>MODEL: Return s3Key
    MODEL-->>API: Return {audio_url, s3_key}
    API-->>JOB: Return result
    JOB->>DB: Update clip with s3Key
    JOB->>DB: Deduct 50 credits from user

    UI->>SA: generationStatus(audioId)
    SA->>DB: Query GeneratedAudioClip
    DB-->>SA: Return clip with s3Key
    SA->>S3: Generate presigned URL
    S3-->>SA: Return URL (1hr expiry)
    SA-->>UI: Return {status: "completed", audioUrl}
    UI->>User: Play audio from presigned URL

    Note over User,S3: Voice Conversion Flow (Speech-to-Speech)

    User->>UI: Drop audio file & select target voice
    UI->>SA: generateUploadUrl()
    SA->>S3: Create presigned PUT URL
    S3-->>SA: Return {uploadUrl, s3Key}
    SA-->>UI: Return upload credentials

    UI->>S3: PUT audio file directly
    S3-->>UI: Upload complete

    UI->>SA: generateSpeechToSpeech(s3Key, voice)
    SA->>DB: Create GeneratedAudioClip (seedvc)
    SA->>INN: send("generate.request", {clipId, service: "seedvc"})
    SA-->>UI: Return {audioId}

    INN->>JOB: Trigger aiGenerationFunction
    JOB->>DB: Fetch clip {originalVoiceS3Key, voice}
    JOB->>API: POST /convert {source_audio_key, target_voice}
    API->>S3: Download source audio
    S3-->>API: Return audio bytes
    API->>MODEL: VoiceConversion.convert()
    MODEL->>MODEL: Extract speaker embedding (CAMPPlus)
    MODEL->>MODEL: DiT diffusion conversion
    MODEL->>MODEL: BigVGAN vocoder
    MODEL->>S3: Upload to seedvc-outputs/
    S3-->>MODEL: Return s3Key
    MODEL-->>API: Return {audio_url, s3_key}
    API-->>JOB: Return result
    JOB->>DB: Update clip, deduct credits

    UI->>SA: Poll generationStatus()
    SA-->>UI: Return {status: "completed", audioUrl}
    UI->>User: Play converted audio

    Note over User,S3: Sound Effect Generation Flow

    User->>UI: Enter prompt
    UI->>SA: generateSoundEffect(prompt)
    SA->>DB: Create GeneratedAudioClip (make-an-audio)
    SA->>INN: send("generate.request", {clipId, service: "make-an-audio"})
    SA-->>UI: Return {audioId}

    INN->>JOB: Trigger aiGenerationFunction
    JOB->>DB: Fetch clip {text}
    JOB->>API: POST /generate {prompt}
    API->>MODEL: GenWav.generate()
    MODEL->>MODEL: CLAP text encoding
    MODEL->>MODEL: DDIM sampling (100 steps)
    MODEL->>MODEL: BigVGAN vocoder
    MODEL->>S3: Upload to make-an-audio-outputs/
    S3-->>MODEL: Return s3Key
    MODEL-->>API: Return {audio_url, s3_key}
    API-->>JOB: Return result
    JOB->>DB: Update clip, deduct credits

    UI->>SA: Poll generationStatus()
    SA-->>UI: Return {status: "completed", audioUrl}
    UI->>User: Play generated sound effect
