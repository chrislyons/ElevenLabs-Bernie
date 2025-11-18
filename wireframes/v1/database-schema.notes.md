# Database Schema - Documentation Notes

## Overview

ElevenLabs-Bernie uses Prisma ORM with support for SQLite (development) and MySQL/PostgreSQL (production). The schema supports user authentication, session management, and audio generation history.

## Database Configuration

### Connection

**File**: `src/server/db.ts`

```typescript
import { PrismaClient } from "@prisma/client";

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined;
};

export const db =
  globalForPrisma.prisma ??
  new PrismaClient({
    log: env.NODE_ENV === "development" ? ["query", "error", "warn"] : ["error"],
  });

if (env.NODE_ENV !== "production") globalForPrisma.prisma = db;
```

### Environment Variable

```bash
# SQLite (development)
DATABASE_URL="file:./dev.db"

# MySQL
DATABASE_URL="mysql://user:password@localhost:3306/elevenlabs"

# PostgreSQL
DATABASE_URL="postgresql://user:password@localhost:5432/elevenlabs"
```

## Table Definitions

### User

**Purpose**: Core user accounts with authentication and credits

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | String | PK, CUID | Unique identifier |
| name | String? | - | Display name |
| email | String | Unique | Login email |
| emailVerified | DateTime? | - | Verification timestamp |
| image | String? | - | Profile image URL |
| password | String | - | Bcrypt hash (10 rounds) |
| credits | Int | Default: 10000 | Generation credits |

**Relationships**:
- One-to-many with `Account`
- One-to-many with `Session`
- One-to-many with `Post`
- One-to-many with `GeneratedAudioClip`

**Indexes**:
- Primary: `id`
- Unique: `email`

---

### Account

**Purpose**: OAuth provider accounts (for future OAuth support)

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | String | PK, CUID | Unique identifier |
| userId | String | FK → User | Account owner |
| type | String | - | Account type |
| provider | String | - | OAuth provider |
| providerAccountId | String | - | Provider's ID |
| refresh_token | String? | - | OAuth refresh token |
| access_token | String? | - | OAuth access token |
| expires_at | Int? | - | Token expiration |
| token_type | String? | - | Token type |
| scope | String? | - | OAuth scopes |
| id_token | String? | - | OIDC ID token |
| session_state | String? | - | Session state |

**Indexes**:
- Primary: `id`
- Unique: `(provider, providerAccountId)`

---

### Session

**Purpose**: NextAuth session management

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | String | PK, CUID | Unique identifier |
| sessionToken | String | Unique | Session token |
| userId | String | FK → User | Session owner |
| expires | DateTime | - | Expiration time |

**Indexes**:
- Primary: `id`
- Unique: `sessionToken`

---

### VerificationToken

**Purpose**: Email verification tokens

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| identifier | String | PK (part) | Email or phone |
| token | String | Unique | Verification token |
| expires | DateTime | - | Token expiration |

**Indexes**:
- Unique: `(identifier, token)`

---

### Post

**Purpose**: Demo content (T3 Stack template)

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | Int | PK, Auto-inc | Unique identifier |
| name | String | - | Post title |
| createdAt | DateTime | Default: now() | Creation time |
| updatedAt | DateTime | @updatedAt | Last update |
| createdById | String | FK → User | Author |

**Indexes**:
- Primary: `id`
- Index: `createdById`
- Index: `name`

---

### GeneratedAudioClip

**Purpose**: Audio generation history and results

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | String | PK, CUID | Unique identifier |
| userId | String | FK → User | Generator |
| service | String | - | 'styletts2', 'seedvc', 'make-an-audio' |
| text | String? | - | Input text (TTS/SFX) |
| voice | String? | - | Target voice ID |
| originalVoiceS3Key | String? | - | Source audio (voice conversion) |
| s3Key | String? | - | Generated audio S3 key |
| failed | Boolean | Default: false | Generation failed |
| createdAt | DateTime | Default: now() | Creation time |

**Indexes**:
- Primary: `id`
- Index: `userId`

## Common Queries

### User Operations

**Create User** (Registration):
```typescript
const user = await db.user.create({
  data: {
    email: "user@example.com",
    password: await bcrypt.hash(password, 10),
    name: "User Name",
    credits: 10000
  }
});
```

**Find User** (Authentication):
```typescript
const user = await db.user.findUnique({
  where: { email: "user@example.com" }
});
```

**Deduct Credits**:
```typescript
await db.user.update({
  where: { id: userId },
  data: { credits: { decrement: 50 } }
});
```

### Audio Clip Operations

**Create Clip**:
```typescript
const clip = await db.generatedAudioClip.create({
  data: {
    userId: session.user.id,
    service: "styletts2",
    text: "Hello world",
    voice: "andreas",
    s3Key: null,
    failed: false
  }
});
```

**Update with Result**:
```typescript
await db.generatedAudioClip.update({
  where: { id: clipId },
  data: { s3Key: "styletts2-output/uuid.wav" }
});
```

**Mark as Failed**:
```typescript
await db.generatedAudioClip.update({
  where: { id: clipId },
  data: { failed: true }
});
```

**Get User History**:
```typescript
const clips = await db.generatedAudioClip.findMany({
  where: {
    userId: session.user.id,
    service: "styletts2"
  },
  orderBy: { createdAt: "desc" },
  take: 50
});
```

**Check Status**:
```typescript
const clip = await db.generatedAudioClip.findUnique({
  where: { id: clipId },
  select: { s3Key: true, failed: true }
});
```

## Migration Strategy

### Development

```bash
# Push schema changes directly
npx prisma db push

# Reset database
npx prisma migrate reset
```

### Production

```bash
# Create migration
npx prisma migrate dev --name add_credits_field

# Deploy migrations
npx prisma migrate deploy
```

### Schema Changes

When adding new columns:

1. Add to `schema.prisma`
2. Run `npx prisma migrate dev`
3. Update affected queries
4. Test in development
5. Deploy migration to production

## Data Integrity

### Foreign Key Constraints

All relationships use cascading deletes:
- Deleting a User deletes all their Sessions, Accounts, Posts, and GeneratedAudioClips

### Required Fields

- `User.email` - Cannot be null
- `User.password` - Cannot be null
- `GeneratedAudioClip.userId` - Cannot be null
- `GeneratedAudioClip.service` - Cannot be null

### Default Values

- `User.credits` - 10000
- `GeneratedAudioClip.failed` - false
- `GeneratedAudioClip.createdAt` - Current timestamp

## Performance Considerations

### Indexes

**Existing**:
- User email (unique lookups)
- Session token (auth validation)
- Account provider+ID (OAuth)
- Post name and createdById

**Recommended Additions**:
```prisma
@@index([userId, service, createdAt])  // History queries
@@index([service, failed])              // Admin monitoring
```

### Query Optimization

1. **Pagination**: Always use `take` and `skip` for history
2. **Select**: Only fetch needed fields
3. **Include**: Avoid N+1 with eager loading

### Connection Pooling

For production, use connection pooling:

```bash
DATABASE_URL="postgresql://user:pass@host/db?connection_limit=5"
```

## Backup and Recovery

### SQLite Backup

```bash
cp prisma/dev.db prisma/dev.db.backup
```

### MySQL/PostgreSQL Backup

```bash
pg_dump -U user dbname > backup.sql
mysqldump -u user -p dbname > backup.sql
```

## Schema Evolution

### Current State (v1)

- User authentication with credits
- NextAuth session management
- Audio generation history

### Potential Future Changes

1. **Add Subscription Plans**
   ```prisma
   model Subscription {
     id        String   @id @default(cuid())
     userId    String
     plan      String   // 'free', 'pro', 'enterprise'
     expiresAt DateTime
     user      User     @relation(fields: [userId], references: [id])
   }
   ```

2. **Add Voice Models**
   ```prisma
   model CustomVoice {
     id        String   @id @default(cuid())
     userId    String
     name      String
     s3Key     String   // Training data
     modelKey  String   // Trained model
     status    String   // 'training', 'ready', 'failed'
     user      User     @relation(fields: [userId], references: [id])
   }
   ```

3. **Add Usage Analytics**
   ```prisma
   model UsageLog {
     id        String   @id @default(cuid())
     userId    String
     service   String
     credits   Int
     timestamp DateTime @default(now())
     user      User     @relation(fields: [userId], references: [id])
   }
   ```

## Related Diagrams

- [Architecture Overview](./architecture-overview.mermaid.md) - System design
- [Data Flow](./data-flow.mermaid.md) - How data moves
- [Authentication](./authentication-authorization.mermaid.md) - Auth flows
