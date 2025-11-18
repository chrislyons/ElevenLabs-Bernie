%% ElevenLabs-Bernie Component Map
%% Detailed breakdown of all modules, their responsibilities, and dependencies
%% Version: v1

classDiagram
    direction TB

    %% Frontend Page Components
    class TextToSpeechPage {
        +route: /app/speech-synthesis/text-to-speech
        +components: TextToSpeechEditor, HistoryPanel
        +actions: generateTextToSpeech()
    }

    class SpeechToSpeechPage {
        +route: /app/speech-synthesis/speech-to-speech
        +components: VoiceChanger, HistoryPanel
        +actions: generateSpeechToSpeech()
    }

    class SoundEffectsPage {
        +route: /app/sound-effects/generate
        +components: SoundEffectsEditor, HistoryPanel
        +actions: generateSoundEffect()
    }

    %% Frontend UI Components
    class TextToSpeechEditor {
        +props: onSubmit, initialText
        +state: text, voice, settings
        +methods: handleGenerate()
    }

    class VoiceChanger {
        +props: onUpload, onConvert
        +state: audioFile, targetVoice
        +methods: handleDrop(), handleConvert()
    }

    class HistoryPanel {
        +props: service, clips
        +state: selectedClip
        +methods: loadHistory(), playClip()
    }

    class PlayBar {
        +props: audioUrl, title
        +state: isPlaying, progress, duration
        +methods: play(), pause(), seek(), download()
    }

    class Sidebar {
        +props: currentRoute
        +state: collapsed
        +methods: navigate()
    }

    class VoiceSelector {
        +props: service, onSelect
        +state: selectedVoice, voices
        +methods: loadVoices()
    }

    class GenerateButton {
        +props: onClick, credits
        +state: loading
        +methods: handleClick()
    }

    %% State Stores
    class AudioStore {
        +currentAudio: AudioClip
        +isPlaying: boolean
        +progress: number
        +duration: number
        +setAudio()
        +setPlaying()
        +setProgress()
    }

    class VoiceStore {
        +styletts2Voice: string
        +seedvcVoice: string
        +makeAnAudioVoice: string
        +setVoice()
        +getVoice()
    }

    class UIStore {
        +sidebarCollapsed: boolean
        +modalOpen: boolean
        +toggleSidebar()
        +openModal()
    }

    %% Server Actions
    class ServerActions {
        +generateTextToSpeech(text, voice)
        +generateSpeechToSpeech(s3Key, voice)
        +generateSoundEffect(prompt)
        +generationStatus(clipId)
        +generateUploadUrl()
    }

    %% Inngest Functions
    class InngestClient {
        +send(event)
    }

    class aiGenerationFunction {
        +event: generate.request
        +steps: fetchClip, callAPI, saveResult, deductCredits
        +retry: 2
        +throttle: 3/min
    }

    %% Database Models
    class User {
        +id: string
        +email: string
        +password: string
        +name: string
        +credits: number
        +generatedAudioClips: AudioClip[]
    }

    class GeneratedAudioClip {
        +id: string
        +userId: string
        +service: string
        +text: string
        +voice: string
        +s3Key: string
        +failed: boolean
        +createdAt: DateTime
    }

    %% AI Service APIs
    class StyleTTS2API {
        +POST /generate
        +GET /voices
        +GET /health
        -inference: StyleTTS2Inference
    }

    class SeedVCAPI {
        +POST /convert
        +GET /voices
        +GET /health
        -inference: VoiceConversion
    }

    class MakeAnAudioAPI {
        +POST /generate
        +GET /health
        -sampler: GenWav
    }

    %% AI Model Components
    class StyleTTS2Inference {
        +model: StyleTTS2Model
        +textAligner: ASR
        +pitchExtractor: JDC
        +plbert: PLBERT
        +inference(text, voice)
        +computeStyle(audio)
    }

    class VoiceConversion {
        +dit: DiTModel
        +campplus: CAMPPlus
        +bigvgan: BigVGAN
        +convert(sourceAudio, targetVoice)
    }

    class GenWav {
        +diffusion: LatentDiffusion
        +clap: CLAPEncoder
        +bigvgan: BigVGAN
        +generate(prompt)
    }

    %% Relationships - Pages to Components
    TextToSpeechPage --> TextToSpeechEditor
    TextToSpeechPage --> HistoryPanel
    TextToSpeechPage --> VoiceSelector
    TextToSpeechPage --> GenerateButton

    SpeechToSpeechPage --> VoiceChanger
    SpeechToSpeechPage --> HistoryPanel
    SpeechToSpeechPage --> VoiceSelector

    SoundEffectsPage --> HistoryPanel
    SoundEffectsPage --> GenerateButton

    %% Relationships - Components to Stores
    TextToSpeechEditor --> VoiceStore
    VoiceChanger --> VoiceStore
    VoiceSelector --> VoiceStore
    HistoryPanel --> AudioStore
    PlayBar --> AudioStore
    Sidebar --> UIStore

    %% Relationships - Components to Actions
    TextToSpeechEditor --> ServerActions
    VoiceChanger --> ServerActions
    GenerateButton --> ServerActions
    HistoryPanel --> ServerActions

    %% Relationships - Actions to Inngest
    ServerActions --> InngestClient
    InngestClient --> aiGenerationFunction

    %% Relationships - Inngest to APIs
    aiGenerationFunction --> StyleTTS2API
    aiGenerationFunction --> SeedVCAPI
    aiGenerationFunction --> MakeAnAudioAPI

    %% Relationships - APIs to Models
    StyleTTS2API --> StyleTTS2Inference
    SeedVCAPI --> VoiceConversion
    MakeAnAudioAPI --> GenWav

    %% Relationships - Actions to Database
    ServerActions --> User
    ServerActions --> GeneratedAudioClip
    aiGenerationFunction --> GeneratedAudioClip
    aiGenerationFunction --> User
