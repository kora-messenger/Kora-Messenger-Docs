# Kora Messenger — Backend Architecture Plan

> Reference document for migrating from the temporary Base44 backend to a self-hosted infrastructure on a custom domain.

---

## 1. Hosting

| Option | Cost | Best For |
|--------|------|----------|
| DigitalOcean Droplet | $6/mo | Full control, cheapest at scale |
| Hetzner Cloud | $4/mo | Cheapest VPS, EU servers |
| Railway | $5+/mo | Zero DevOps, Git-push deploys |
| Render | $7+/mo | Zero DevOps, free tier available |

**Recommendation:** Start with Railway or Render for speed, migrate to Hetzner/DigitalOcean when you need WebSockets and Redis.

**Cloudflare** sits in front of everything — free CDN, free SSL, DDoS protection, DNS management.

---

## 2. Tech Stack

- **Runtime:** Node.js 20+ with Fastify (faster than Express, built-in schema validation)
- **Database:** PostgreSQL 16 (users, messages, sessions, subscriptions)
- **Cache/Real-time:** Redis 7 (presence, pub/sub, rate limiting, verification codes)
- **Storage:** Cloudflare R2 (S3-compatible, free egress) — voice notes, images, profile pictures
- **Email:** Resend (free 3k/month) or Brevo (300/day free)
- **AI:** OpenRouter (free-tier models) orchestrated via backend
- **Payments:** Paystack (Africa), Google Pay, Apple Pay

---

## 3. Domain Structure

```
kora.com          → App frontend (web version, if built later)
api.kora.com      → Backend API + WebSocket server
mail.kora.com     → Email sending (Resend/Brevo SMTP)
cdn.kora.com      → File storage (Cloudflare R2 / S3)
```

All domain references are centralized in one config file (`lib/config/kora_config.dart`) — migrating is a single value change.

---

## 4. Database Schema (PostgreSQL)

### users
```sql
CREATE TABLE users (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email           VARCHAR(255) UNIQUE NOT NULL,
  username        VARCHAR(50) UNIQUE,
  kora_id         VARCHAR(20) UNIQUE NOT NULL,
  password_hash   TEXT,
  pin_hash        TEXT,
  full_name       VARCHAR(100),
  bio             TEXT,
  avatar_url      TEXT,
  is_premium      BOOLEAN DEFAULT FALSE,
  premium_expires_at  TIMESTAMPTZ,
  premium_source      VARCHAR(20),  -- 'paystack' | 'google' | 'apple' | 'owner_override'
  is_verified    BOOLEAN DEFAULT FALSE,
  is_suspended   BOOLEAN DEFAULT FALSE,
  suspension_reason    TEXT,
  suspension_expires_at TIMESTAMPTZ,
  profile_completed    BOOLEAN DEFAULT FALSE,
  passkeys_enabled     BOOLEAN DEFAULT FALSE,
  created_at     TIMESTAMPTZ DEFAULT NOW(),
  updated_at     TIMESTAMPTZ DEFAULT NOW()
);
```

### sessions (multi-device login)
```sql
CREATE TABLE sessions (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  device_id     VARCHAR(255) NOT NULL,
  device_name   VARCHAR(255),
  platform      VARCHAR(50),   -- 'android' | 'ios' | 'web'
  is_trusted    BOOLEAN DEFAULT FALSE,
  created_at    TIMESTAMPTZ DEFAULT NOW(),
  last_active   TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(user_id, device_id)
);
```

### passkeys
```sql
CREATE TABLE passkeys (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  credential_id TEXT NOT NULL,
  public_key    BYTEA NOT NULL,
  device_id     VARCHAR(255),
  created_at    TIMESTAMPTZ DEFAULT NOW()
);
```

### conversations
```sql
CREATE TABLE conversations (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  type        VARCHAR(20) NOT NULL,  -- 'direct' | 'group' | 'channel'
  name        VARCHAR(255),           -- for groups/channels
  description TEXT,
  avatar_url  TEXT,
  created_by  UUID REFERENCES users(id),
  created_at  TIMESTAMPTZ DEFAULT NOW()
);
```

### conversation_members
```sql
CREATE TABLE conversation_members (
  conversation_id  UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
  user_id           UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  role              VARCHAR(20) DEFAULT 'member',  -- 'owner' | 'admin' | 'member'
  joined_at         TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (conversation_id, user_id)
);
```

### messages
```sql
CREATE TABLE messages (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  conversation_id UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
  sender_id       UUID REFERENCES users(id) ON DELETE SET NULL,
  type            VARCHAR(20) DEFAULT 'text',  -- 'text' | 'voice' | 'image' | 'file' | 'system'
  text            TEXT,
  attachment_url  TEXT,
  voice_duration  VARCHAR(10),    -- "0:12"
  voice_transcript TEXT,
  reply_to_id     UUID REFERENCES messages(id) ON DELETE SET NULL,
  is_starred      BOOLEAN DEFAULT FALSE,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_messages_conversation ON messages(conversation_id, created_at DESC);
```

### message_status (read receipts)
```sql
CREATE TABLE message_status (
  message_id  UUID NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
  user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  status      VARCHAR(20) NOT NULL,  -- 'sent' | 'delivered' | 'read'
  updated_at  TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (message_id, user_id)
);
```

### verification_codes
```sql
CREATE TABLE verification_codes (
  id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email     VARCHAR(255) NOT NULL,
  code      VARCHAR(6) NOT NULL,
  type      VARCHAR(30) NOT NULL,  -- 'login' | 'registration' | 'email_change' | 'password_reset'
  expires_at TIMESTAMPTZ NOT NULL,
  used      BOOLEAN DEFAULT FALSE,
  attempts  INT DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### subscriptions
```sql
CREATE TABLE subscriptions (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  provider        VARCHAR(20) NOT NULL,  -- 'paystack' | 'google' | 'apple'
  plan            VARCHAR(20) NOT NULL,  -- 'monthly' | 'yearly'
  status          VARCHAR(20) NOT NULL,  -- 'active' | 'cancelled' | 'expired'
  starts_at       TIMESTAMPTZ,
  expires_at      TIMESTAMPTZ,
  provider_ref    TEXT,  -- Paystack subscription ref / Google purchase token / Apple receipt
  created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

### suspensions
```sql
CREATE TABLE suspensions (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id               UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  user_kora_id          VARCHAR(20),
  username              VARCHAR(50),
  reason                TEXT NOT NULL,
  detection_type        VARCHAR(50),  -- 'auto' | 'manual'
  detection_details     JSONB,
  auto_detected         BOOLEAN DEFAULT FALSE,
  expires_at            TIMESTAMPTZ,
  status                VARCHAR(20) DEFAULT 'active',  -- 'active' | 'resolved' | 'appealed'
  suspended_by          UUID REFERENCES users(id),
  owner_notified        BOOLEAN DEFAULT FALSE,
  appeal_message        TEXT,
  appeal_submitted_at   TIMESTAMPTZ,
  appeal_decision       VARCHAR(20),  -- 'upheld' | 'overturned' | 'pending'
  suspended_at          TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 5. Real-time Messaging

### WebSocket Server (Fastify + ws)
```
Client connects → WebSocket → Server authenticates via JWT
                                ↓
                        Redis presence SET (user:123 → socket_id, TTL 30s)
                                ↓
                        Heartbeat refreshes TTL every 15s
                                ↓
                        On disconnect → TTL expires → user shows "last seen"
```

### Message Delivery Flow
```
User A sends message
  → POST /api/messages
  → Server saves to PostgreSQL
  → Server publishes to Redis channel: conversation:{conv_id}
  → All server instances subscribed to that channel receive it
  → Each instance pushes to its connected WebSocket clients in that conversation
  → User B receives message in real-time
```

### Presence
- Redis key: `presence:{user_id}` → `{ socket_id, last_seen }`
- TTL: 30 seconds, refreshed by heartbeat every 15s
- On expiry: user goes offline, `last_seen` timestamp saved

---

## 6. File Uploads

```
App requests upload → GET /api/upload/presign?filename=voice_note.aac
  → Server generates pre-signed R2/S3 URL
  → App uploads file directly to R2 (bypasses server)
  → App sends message with the R2 URL
  → Server stores URL in messages table
```

- Voice notes: `.aac` / `.m4a`, max 5 minutes
- Images: `.jpg` / `.png`, max 5MB, auto-compress on client
- Profile pictures: `.jpg`, max 2MB

---

## 7. Email (Verification Codes + Notifications)

- **Resend** (free 3k/month) or **Brevo** (300/day free)
- DNS records needed on domain:
  - SPF: `v=spf1 include:resend.com ~all`
  - DKIM: provided by Resend/Brevo, add as TXT record
  - DMARC: `v=DMARC1; p=quarantine; rua=mailto:admin@kora.com`
- Sender: `noreply@kora.com` for verification, `notifications@kora.com` for alerts

---

## 8. AI Engine (OpenRouter)

```
App → POST /api/ai/chat
  → Server checks Redis rate limit (per user: 50 req/hour free, unlimited premium)
  → Server loads local knowledge base (prevents hallucinations)
  → Server calls OpenRouter (free-tier model)
  → Server returns response to app
```

Features:
- Kora AI (general assistant)
- Kora Support (helpdesk)
- Chat summary
- Writing assistant
- Voice note transcription/translation (on-device STT → OpenRouter translation)

---

## 9. Payments

### Paystack (Africa)
```
App → POST /api/payments/initialize (amount, plan)
  → Server calls Paystack API → returns authorization URL
  → User pays in app/browser
  → Paystack webhook → POST /api/payments/webhook
  → Server verifies signature → updates subscriptions table
  → Sets user.is_premium = true, premium_expires_at
```

### Google Pay / Apple Pay
- Same webhook pattern via respective platforms
- Server verifies receipts server-side

### Owner Override
- `premium_source = 'owner_override'` → permanent premium, no expiry check

---

## 10. Security

| Layer | Implementation |
|-------|---------------|
| Auth | JWT (15 min access) + Refresh token (30 days, httpOnly cookie) |
| Password hashing | argon2id |
| Rate limiting | Redis — 100 req/min per IP, 50 AI req/hour per user |
| Input validation | Fastify JSON schema on every endpoint |
| CORS | Locked to app origins only |
| SSL/TLS | Cloudflare (termination) + Let's Encrypt (origin) |
| File upload | Pre-signed URLs only, no direct server uploads |
| Verification codes | 6-digit, 10 min expiry, max 5 attempts, 1 per minute per email |

---

## 11. API Endpoints

### Auth
```
POST   /api/auth/register          → create account
POST   /api/auth/login             → email + password
POST   /api/auth/verify-code       → verify 6-digit code
POST   /api/auth/resend-code       → resend verification code
POST   /api/auth/reset-password    → request password reset
POST   /api/auth/set-password      → set new password
POST   /api/auth/refresh           → refresh access token
POST   /api/auth/logout            → revoke session
GET    /api/auth/sessions          → list active sessions
DELETE /api/auth/sessions/:id      → revoke specific session
```

### Passkeys
```
POST   /api/passkeys/register      → register new passkey
POST   /api/passkeys/authenticate  → login with passkey
DELETE /api/passkeys/:id           → remove passkey
```

### Users
```
GET    /api/users/me               → own profile
PATCH  /api/users/me               → update profile
GET    /api/users/:koraId          → public profile by Kora ID
GET    /api/users/search?q=        → search by name, Kora ID, @username
PATCH  /api/users/me/email         → change email (two-step verify)
PATCH  /api/users/me/pin           → set/change secure PIN
```

### Conversations & Messages
```
GET    /api/conversations                    → list user's conversations
POST   /api/conversations                    → create direct/group/channel
GET    /api/conversations/:id/messages        → paginated messages
POST   /api/conversations/:id/messages       → send message
DELETE /api/conversations/:id/messages/:msgId → delete message
PATCH  /api/conversations/:id/messages/:msgId → edit/star message
POST   /api/conversations/:id/read           → mark conversation as read
WS     /api/ws                               → WebSocket (messages, typing, presence)
```

### Files
```
GET    /api/upload/presign                    → get pre-signed upload URL
```

### AI
```
POST   /api/ai/chat                           → Kora AI
POST   /api/ai/support                        → Kora Support
POST   /api/ai/summary                        → chat summary
POST   /api/ai/writing                        → writing assistant
POST   /api/ai/translate                       → voice note translation
```

### Payments
```
POST   /api/payments/initialize               → start payment
POST   /api/payments/webhook                  → Paystack webhook
GET    /api/payments/status                   → check subscription status
```

### Admin (owner only)
```
POST   /api/admin/suspend                     → suspend user
POST   /api/admin/unsuspend                   → lift suspension
POST   /api/admin/premium-override            → grant permanent premium
GET    /api/admin/suspensions                  → list suspensions
POST   /api/admin/suspensions/:id/appeal      → review appeal
```

---

## 12. Migration from Base44

1. **Set up server** — provision VPS or Railway/Render instance
2. **Configure domain** — point `api.kora.com` to server IP via Cloudflare DNS
3. **Deploy backend** — push Node.js code, set up PostgreSQL + Redis
4. **Run migration script** — read from Base44 entities, insert into PostgreSQL
5. **Update app config** — change `apiBase` in `kora_config.dart` from Base44 URLs to `https://api.kora.com`
6. **Test on staging** — build APK with new config, verify all features
7. **Flip DNS** — push production update, all traffic routes to new backend

### Migration Script (one-time)
```javascript
// scripts/migrate-from-base44.js
// Reads KoraUser, Conversation, ChatMessage, VerificationCode entities
// from Base44 → inserts into PostgreSQL
// Maps Base44 field names to new schema column names
// Deduplicates by email/kora_id
```

---

## 13. Estimated Monthly Cost (at launch)

| Service | Cost |
|---------|------|
| Hetzner VPS (2 vCPU, 4GB) | $4 |
| Cloudflare (DNS + CDN + SSL) | Free |
| Cloudflare R2 (10GB storage) | Free |
| Resend (3k emails) | Free |
| OpenRouter (free-tier models) | Free |
| Domain (.com) | ~$10/year |
| **Total** | **~$5/month** |

Scales to ~10k active users before needing upgrades.

---

## 14. Environment Variables (server)

```
DATABASE_URL=postgresql://user:pass@localhost:5432/kora
REDIS_URL=redis://localhost:6379
JWT_SECRET=<random-64-chars>
JWT_REFRESH_SECRET=<random-64-chars>
OPENROUTER_API_KEY=<key>
RESEND_API_KEY=<key>
PAYSTACK_SECRET_KEY=<key>
R2_ACCESS_KEY=<key>
R2_SECRET_KEY=<key>
R2_ENDPOINT=https://<account>.r2.cloudflarestorage.com
R2_BUCKET=kora-media
CORS_ORIGIN=https://kora.com
OWNER_EMAIL=ijezie@example.com
OWNER_KORA_ID=KORA-001
```

---

_Document maintained in `docs/backend-architecture.md` — update as the architecture evolves._
