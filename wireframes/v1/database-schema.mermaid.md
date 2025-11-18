%% ElevenLabs-Bernie Database Schema
%% Entity Relationship Diagram for Prisma models
%% Version: v1

erDiagram
    User {
        string id PK "CUID primary key"
        string name "Optional display name"
        string email UK "Unique email address"
        datetime emailVerified "Email verification timestamp"
        string image "Profile image URL"
        string password "Bcrypt hashed password"
        int credits "Generation credits (default: 10000)"
    }

    Account {
        string id PK "CUID primary key"
        string userId FK "References User"
        string type "OAuth account type"
        string provider "OAuth provider name"
        string providerAccountId "Provider's account ID"
        string refresh_token "OAuth refresh token"
        string access_token "OAuth access token"
        int expires_at "Token expiration timestamp"
        string token_type "Token type (Bearer)"
        string scope "OAuth scopes"
        string id_token "OIDC ID token"
        string session_state "Session state"
    }

    Session {
        string id PK "CUID primary key"
        string sessionToken UK "Unique session token"
        string userId FK "References User"
        datetime expires "Session expiration"
    }

    VerificationToken {
        string identifier PK "Email or phone"
        string token UK "Unique verification token"
        datetime expires "Token expiration"
    }

    Post {
        int id PK "Auto-increment ID"
        string name "Post title"
        datetime createdAt "Creation timestamp"
        datetime updatedAt "Last update timestamp"
        string createdById FK "References User"
    }

    GeneratedAudioClip {
        string id PK "CUID primary key"
        string userId FK "References User"
        string service "styletts2|seedvc|make-an-audio"
        string text "Input text for TTS"
        string voice "Target voice ID"
        string originalVoiceS3Key "Source audio for conversion"
        string s3Key "Generated audio S3 key"
        boolean failed "Generation failure flag"
        datetime createdAt "Creation timestamp"
    }

    %% Relationships
    User ||--o{ Account : "has many"
    User ||--o{ Session : "has many"
    User ||--o{ Post : "has many"
    User ||--o{ GeneratedAudioClip : "has many"

    Account }o--|| User : "belongs to"
    Session }o--|| User : "belongs to"
    Post }o--|| User : "created by"
    GeneratedAudioClip }o--|| User : "created by"
