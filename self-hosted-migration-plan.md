# Kora Messenger — Self-Hosted Migration & Scaling Plan

> Executable plan for migrating Kora from the temporary Base44 backend to self-hosted infrastructure on **koramessenger.com**. Companion to [backend-architecture.md](./backend-architecture.md), which defines the tech stack. This doc defines *what moves, in what order, and how we verify each step.*

---

## Guiding Principles

1. **No downtime for users.** The app is domain-swappable via `KORA_BACKEND_URL`. Migration happens in phases; the app keeps working on Base44 until each subsystem is verified on the new infra.
2. **Real-time first.** The biggest architectural win is replacing client polling with WebSockets. This is what makes 10,000+ concurrent users cheap.
3. **Media off the API.** Voice notes, photos, and videos live on object storage + CDN and never flow through API servers.
4. **Scale by config, not rewrite.** Every component must be horizontally scalable from day one.

---

## Phase 0 — Domain Purchase & Foundations (Day 1)

- [ ] Purchase `koramessenger.com`
- [ ] Move DNS to Cloudflare (free tier): CDN, SSL, DDoS protection
- [ ] Provision one VPS (Hetzner CX32 or DO $24/mo — 4 vCPU / 8 GB) — this single box runs everything for the first few hundred users
- [ ] Set up Docker + docker-compose on the VPS: Fastify API, PostgreSQL 16, Redis 7, WebSocket server
- [ ] Create subdomains:
  ```
  koramessenger.com        → marketing site (existing)
  api.koramessenger.com    → API + WebSocket server
  cdn.koramessenger.com    → media (Cloudflare R2)
  mail.koramessenger.com   → email sender identity (Resend/Brevo)
  ```
- [ ] Wire `KORA_BACKEND_URL` in app config → `https://api.koramessenger.com`
- [ ] Backups: nightly PostgreSQL dumps to object storage, 30-day retention

**Exit criteria:** a `GET /health` on api.koramessenger.com returns 200 behind SSL.

---

## Phase 1 — Auth & Accounts (Week 1)

Move the highest-stakes data first, while user count is smallest.

- [ ] Port KoraUser / VerificationCode / TrustedDevice / Passkey schemas to PostgreSQL
- [ ] One-way export from Base44 entities → Postgres (idempotent import script, re-runnable)
- [ ] Replace Gmail-API login-code sender with transactional provider (Resend) — keep Gmail as fallback
- [ ] Rate limits: 5 code requests / 15 min / account; 50 / hr / IP
- [ ] Sessions: issue JWT access tokens (15 min) + refresh tokens (30 days, rotated), stored in Redis

**Exit criteria:** a fresh install can create an account, receive a code, and log in entirely on the new backend. Existing Base44 accounts import cleanly.

---

## Phase 2 — Messaging Core (Week 2)

- [ ] Postgres schema: `conversations`, `messages`, `reactions`, `replies`, `message_status` (sent/delivered/read)
- [ ] **WebSocket gateway**:
  - One persistent connection per device
  - Events: `message.new`, `message.status`, `typing`, `presence`
  - Redis pub/sub fans out across gateway instances (ready for horizontal scale)
- [ ] Deterministic chatId (`dm__user1__user2`) becomes a DB constraint — thread forking becomes impossible at the schema level
- [ ] Presence in Redis with TTL heartbeats (no DB writes for online/offline)
- [ ] Sync protocol: on connect, client sends last-sequence-id per conversation; server ships only the delta (same model the app already uses for cloud restore)

**Exit criteria:** two devices on different networks exchange messages with sub-200 ms delivery, delivered/read receipts work, and reconnecting after airplane mode backfills missed messages.

---

## Phase 3 — Media & Voice Notes (Week 3)

- [ ] Cloudflare R2 buckets: `media` (private, signed URLs), `avatars` (public, cached)
- [ ] Upload flow: client requests presigned URL → uploads directly to R2 → sends message with media reference. Bytes never touch the API.
- [ ] Migrate existing Base44-stored media to R2 (background migration job)
- [ ] Voice note waveforms stored as message metadata (already generated client-side)
- [ ] CDN caching for avatars and view-once media (short TTL)

**Exit criteria:** a voice note recorded on device A plays on device B, and media uploads survive a restart of the API server (because bytes live in R2, not on the box).

---

## Phase 4 — Calls (Week 4)

- [ ] Move WebRTC signaling from DB-record polling to WebSocket events (`call.offer`, `call.answer`, `call.ice`, `call.end`)
- [ ] Media (audio/video streams) already peer-to-peer via WebRTC — add a coturn server on the VPS (UDP 3478)
- [ ] Call connection time target: < 3 s from tap to ring

**Exit criteria:** a 1-on-1 call connects entirely over the new infra with signaling latency under 300 ms.

---

## Phase 5 — Cutover & Decommission (Week 5)

- [ ] Flip app default `KORA_BACKEND_URL` to the new API; keep Base44 as emergency fallback for one release
- [ ] Final Base44 → Postgres data sync (accounts, messages, media references)
- [ ] Publish app release with new backend baked in
- [ ] Monitor 72 h (error rates, delivery latency, WebSocket reconnect storms)
- [ ] Freeze Base44 backend to read-only; archive export

**Exit criteria:** Base44 is no longer in the request path for any user.

---

## Scaling to 10,000+ Concurrent Users

The Phase 0-1 single box (4 vCPU / 8 GB) handles ~500-1,000 concurrent users. Beyond that, scale out — each step is a config change, not a rewrite:

| Component | At 10k concurrent | Monthly cost |
|---|---|---|
| API (Fastify) | 2-3 instances × 4 vCPU behind Cloudflare load balancing | $60-90 |
| WebSocket gateway | 1-2 instances (10k idle connections ≈ 1-2 GB RAM) | $30-50 |
| PostgreSQL | Managed 2 vCPU / 4 GB + read replica | $35-80 |
| Redis | 1-2 GB (presence, sessions, pub/sub) | $15-30 |
| Object storage | Cloudflare R2 — **$0 egress** — ~1 TB media | $15-25 |
| coturn (calls) | Shared with gateway box | — |
| Email codes | Resend/Brevo paid tier | $0-20 |
| **Total (Hetzner/DO)** | | **≈ $155-295/mo** |

(AWS/GCP for the same layout: $400+/mo. Not needed.)

Key design facts that keep this cheap:
- **WebSockets, not polling** — 10k connected users generate ~0 requests/sec while idle vs. tens of thousands/min with polling
- **R2 free egress** — media downloads (the heaviest traffic) cost nothing to serve
- **Presence in Redis** — online/offline updates never hit Postgres

### Launch-day protection (do these before any public push)

- [ ] Cloudflare rate limiting: 100 req/min/IP on API, 10/min on auth endpoints
- [ ] Login-code queue: codes enqueue and drain at provider speed — users wait seconds, never see errors
- [ ] WebSocket reconnect backoff (exponential + jitter) to prevent reconnect storms
- [ ] Managed Postgres connection pooling (PgBouncer) — app instances never exceed pool
- [ ] Load test: k6 script simulating 10k connections + 1k msg/min, run against staging first

---

## Data Map — What Moves Where

| Base44 entity | New home |
|---|---|
| KoraUser | Postgres `users` |
| VerificationCode | Redis (TTL) + Postgres audit log |
| TrustedDevice, Passkey | Postgres `devices`, `passkeys` |
| Conversation, ChatMessage | Postgres `conversations`, `messages` |
| CallSignal | WebSocket signaling (ephemeral, not stored) |
| UserSettings | Postgres `user_settings` (one row, JSONB — matches current settingsJson) |
| SuspensionRecord | Postgres `suspensions` |
| Media files | Cloudflare R2 |

---

## Risk Register

| Risk | Mitigation |
|---|---|
| Data loss during account migration | Idempotent, re-runnable import scripts; verify row counts + checksums per phase |
| Login spike on launch day | Rate limits + email queue + autoscaling on API tier |
| WebSocket reconnect storm after network blip | Exponential backoff with jitter |
| Single VPS failure mid-scale | Image the VPS; Hetzner snapshots daily; upgrade to multi-instance at >1k users |
| Users on old app version still hitting Base44 | Keep Base44 read-only for one release cycle before decommission |

---

*Owner: Ijezie / Nexora Technologies. Execute phase by phase — do not skip exit criteria.*
