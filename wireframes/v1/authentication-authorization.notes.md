# Authentication & Authorization - Documentation Notes

## Overview

ElevenLabs-Bernie implements a multi-layered security model with NextAuth for user authentication, middleware for route protection, and API keys for backend service authentication.

## Authentication Architecture

### Technology Stack

| Component | Technology | Purpose |
|-----------|------------|---------|
| Auth Library | Auth.js (NextAuth 5.0 beta) | Session management |
| Password Hashing | bcryptjs | Secure password storage |
| Session Strategy | JWT | Stateless authentication |
| ORM | Prisma | Database operations |
| Validation | Zod | Input validation |

### Configuration Files

- **Main Config**: `src/server/auth/config.ts`
- **Auth Handler**: `src/app/api/auth/[...nextauth]/route.ts`
- **Middleware**: `src/middleware.ts`

## User Registration

### Flow

1. User submits registration form
2. Server validates input with Zod schema
3. Check if email already exists
4. Hash password with bcrypt (10 rounds)
5. Create user with 10,000 initial credits
6. Redirect to sign-in page

### Code Location

**File**: `src/app/app/sign-up/page.tsx` or similar

### Password Requirements

- Minimum length: Enforced by Zod schema
- Hashing: bcryptjs with cost factor 10
- Storage: Only hash stored in database

### Security Considerations

- Passwords never logged or transmitted in plain text
- Same error message for "email exists" and "invalid input"
- Rate limiting should be added for production

## User Login

### Flow

1. User submits email and password
2. NextAuth receives credentials
3. Query database for user by email
4. Compare submitted password with stored hash
5. Generate JWT token if match
6. Set HTTP-only session cookie
7. Redirect to protected area

### NextAuth Credentials Provider

**File**: `src/server/auth/config.ts`

```typescript
CredentialsProvider({
  name: "credentials",
  credentials: {
    email: { label: "Email", type: "email" },
    password: { label: "Password", type: "password" }
  },
  async authorize(credentials) {
    const user = await db.user.findUnique({
      where: { email: credentials.email }
    });

    if (!user || !user.password) return null;

    const valid = await bcrypt.compare(
      credentials.password,
      user.password
    );

    if (!valid) return null;

    return {
      id: user.id,
      email: user.email,
      name: user.name,
      image: user.image
    };
  }
})
```

### JWT Token Contents

```typescript
{
  sub: user.id,      // User ID
  name: user.name,
  email: user.email,
  image: user.image,
  iat: timestamp,    // Issued at
  exp: timestamp     // Expiration
}
```

### Session Cookie

- Name: `next-auth.session-token`
- HttpOnly: Yes
- Secure: Yes (production)
- SameSite: Lax
- Expiration: 30 days (default)

## Route Protection

### Middleware Configuration

**File**: `src/middleware.ts`

```typescript
export { default } from "next-auth/middleware";

export const config = {
  matcher: ["/app/:path*"]
};
```

### Protected Routes

All routes under `/app/*` require authentication:

- `/app/speech-synthesis/text-to-speech`
- `/app/speech-synthesis/speech-to-speech`
- `/app/sound-effects/generate`
- `/app/sound-effects/history`

### Public Routes

- `/` - Landing page
- `/app/sign-in` - Login page
- `/app/sign-up` - Registration page
- `/api/auth/*` - Auth endpoints

### Redirect Behavior

- Unauthenticated access to protected route → `/app/sign-in`
- After login → Original requested URL or default
- After logout → Landing page

## Server Action Authorization

### Getting Session in Server Actions

```typescript
import { auth } from "~/server/auth";

export async function generateTextToSpeech(text: string, voice: string) {
  const session = await auth();

  if (!session?.user) {
    throw new Error("Unauthorized");
  }

  // Proceed with generation
  const userId = session.user.id;
  // ...
}
```

### Authorization Checks

1. **Session Validity**: Is user logged in?
2. **Credit Balance**: Does user have enough credits?
3. **Rate Limiting**: Is user within limits? (Inngest throttle)

## API Key Authentication

### Backend Services

All three AI services (StyleTTS2, Seed-VC, Make-An-Audio) use Bearer token authentication:

**Request**:
```http
POST /generate HTTP/1.1
Host: localhost:8000
Authorization: Bearer your-api-key-here
Content-Type: application/json
```

**Validation** (Python):
```python
from fastapi import HTTPException, Depends
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

security = HTTPBearer()

def verify_token(credentials: HTTPAuthorizationCredentials = Depends(security)):
    if credentials.credentials != os.environ.get("API_KEY"):
        raise HTTPException(status_code=401, detail="Invalid token")
    return credentials.credentials

@app.post("/generate")
async def generate(request: Request, token: str = Depends(verify_token)):
    # Process request
```

### Environment Variables

```bash
# Frontend
BACKEND_API_KEY=your-shared-api-key

# Backend Services
API_KEY=your-shared-api-key
```

### Security Notes

- Same API key used across all services
- Key should be 32+ characters
- Rotate keys periodically
- Never commit keys to version control

## Credit System

### Overview

| Aspect | Value |
|--------|-------|
| Initial Credits | 10,000 |
| Cost per Generation | 50 |
| Deduction Timing | After successful upload |

### Credit Check Flow

1. User initiates generation
2. Server Action checks credit balance
3. Job processes in queue
4. Credits deducted after S3 upload success
5. UI reflects new balance

### Implementation

**Deduction** (in Inngest function):
```typescript
await db.user.update({
  where: { id: userId },
  data: { credits: { decrement: 50 } }
});
```

**Balance Check** (Server Action):
```typescript
const user = await db.user.findUnique({
  where: { id: session.user.id },
  select: { credits: true }
});

if (user.credits < 50) {
  throw new Error("Insufficient credits");
}
```

### Future Enhancements

1. **Subscription Plans**: Different credit allowances
2. **Credit Purchase**: Stripe integration
3. **Usage History**: Track all credit changes
4. **Rollback**: Refund credits on failure

## Rate Limiting

### Inngest Throttle

```typescript
export const aiGenerationFunction = inngest.createFunction(
  {
    id: "ai-generation",
    throttle: {
      key: "event.data.userId",
      count: 3,
      period: "1m"
    },
    retries: 2
  },
  { event: "generate.request" },
  async ({ event, step }) => {
    // Process generation
  }
);
```

- **Limit**: 3 requests per minute per user
- **Scope**: All services combined
- **Behavior**: Queued requests wait

### Additional Rate Limiting (Recommended)

1. **Login Attempts**: Limit failed logins per IP
2. **Registration**: Limit signups per IP
3. **API Endpoints**: Per-user request limits

## Session Management

### Getting Session (Client)

```typescript
import { useSession } from "next-auth/react";

function Component() {
  const { data: session, status } = useSession();

  if (status === "loading") return <Loading />;
  if (!session) return <SignIn />;

  return <App user={session.user} />;
}
```

### Getting Session (Server)

```typescript
import { auth } from "~/server/auth";

export default async function Page() {
  const session = await auth();

  if (!session) {
    redirect("/app/sign-in");
  }

  return <ProtectedContent />;
}
```

### Session Refresh

- JWT tokens refresh automatically
- Client-side session updates on focus
- Force refresh: `update()` from `useSession`

## Security Best Practices

### Currently Implemented

- Password hashing with bcrypt
- HTTP-only session cookies
- CSRF protection (NextAuth)
- Environment variable secrets
- API key authentication

### Recommended Additions

1. **HTTPS Enforcement**: Redirect HTTP to HTTPS
2. **Security Headers**: CSP, HSTS, etc.
3. **Input Sanitization**: Prevent XSS
4. **SQL Injection Prevention**: Prisma provides this
5. **Audit Logging**: Track auth events
6. **2FA Support**: TOTP or SMS
7. **Account Lockout**: After failed attempts

## OAuth Support (Future)

The schema supports OAuth providers:

```typescript
GoogleProvider({
  clientId: process.env.GOOGLE_CLIENT_ID,
  clientSecret: process.env.GOOGLE_CLIENT_SECRET
})
```

This would create entries in the `Account` table for linked providers.

## Troubleshooting

### Common Issues

1. **"Invalid credentials"**: Check password hash in database
2. **Session not persisting**: Check `AUTH_SECRET` environment variable
3. **Redirect loops**: Check middleware matcher patterns
4. **API 401 errors**: Verify `BACKEND_API_KEY` matches

### Debug Mode

```typescript
// In auth config
export const authConfig = {
  debug: process.env.NODE_ENV === "development",
  // ...
};
```

## Related Diagrams

- [Architecture Overview](./architecture-overview.mermaid.md) - System design
- [Data Flow](./data-flow.mermaid.md) - Request cycles
- [Database Schema](./database-schema.mermaid.md) - User model
