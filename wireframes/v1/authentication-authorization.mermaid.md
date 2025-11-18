%% ElevenLabs-Bernie Authentication & Authorization
%% Security flows and permission model
%% Version: v1

sequenceDiagram
    autonumber
    participant User as User Browser
    participant UI as Next.js App
    participant MW as Middleware
    participant Auth as NextAuth
    participant DB as Prisma/Database
    participant API as AI Service API

    Note over User,API: User Registration Flow

    User->>UI: Submit registration form
    UI->>Auth: signUp(email, password, name)
    Auth->>Auth: Validate input (Zod)
    Auth->>Auth: Hash password (bcrypt, 10 rounds)
    Auth->>DB: Create User record
    DB-->>Auth: Return user ID
    Auth-->>UI: Registration success
    UI->>User: Redirect to sign-in

    Note over User,API: User Login Flow

    User->>UI: Submit login form
    UI->>Auth: signIn("credentials", {email, password})
    Auth->>DB: Find user by email
    DB-->>Auth: Return user with hashed password
    Auth->>Auth: Compare passwords (bcrypt)
    alt Password matches
        Auth->>Auth: Generate JWT token
        Auth->>Auth: Set session cookie
        Auth-->>UI: Login success
        UI->>User: Redirect to app
    else Password fails
        Auth-->>UI: Invalid credentials
        UI->>User: Show error
    end

    Note over User,API: Protected Route Access

    User->>MW: Request /app/speech-synthesis/*
    MW->>MW: Check session cookie
    alt Has valid session
        MW->>UI: Allow request
        UI->>User: Render page
    else No session
        MW->>User: Redirect to /app/sign-in
    end

    Note over User,API: Server Action Authorization

    User->>UI: Click generate button
    UI->>Auth: getServerSession()
    Auth->>Auth: Verify JWT token
    alt Valid session
        Auth-->>UI: Return session with user
        UI->>DB: Check user credits
        DB-->>UI: Return credits
        alt Has credits
            UI->>UI: Process generation
        else No credits
            UI->>User: Show "insufficient credits"
        end
    else Invalid session
        UI->>User: Redirect to sign-in
    end

    Note over User,API: API Authentication (Backend Services)

    UI->>API: POST /generate
    Note right of UI: Header: Authorization: Bearer <API_KEY>
    API->>API: Extract Bearer token
    alt Token matches API_KEY
        API->>API: Process request
        API-->>UI: Return result
    else Token invalid
        API-->>UI: 401 Unauthorized
    end

    Note over User,API: Credit Deduction Flow

    UI->>DB: User has 100 credits
    UI->>API: Generate audio
    API-->>UI: Success
    UI->>DB: Deduct 50 credits
    DB-->>UI: User now has 50 credits
