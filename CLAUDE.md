# Mr. Brand Platform: Project Brief for Claude Code

Read this whole file before doing anything in this project. It is the single source of truth for what we are building, how, and how I want to work. Full detail (install + accounts checklists, full security and Syria-problems tables, scale growth path, learning roadmap, risks) is in `PLAN.md` in this same folder: read it too at the start, and check it whenever a topic comes up.

## Who I am and how to work with me

- I'm Kemo (Kamel Nassif), founder of Solunera, a software and AI agency. This is a paid client project for Solunera.
- I'm a CS graduate (Java, OOP, data structures, HTML/CSS/JS, some Python). I am NEW to Flutter/Dart, NestJS, Docker and production deployment.
- **Teaching mode is ON.** My goal is to build this AND learn to build future projects alone. So:
  - Work one small step at a time. Before writing code, tell me in 2 to 4 lines what we're about to do and why.
  - After writing code, explain the important parts in plain words (what each file does, how pieces connect). Don't explain trivial lines.
  - When there's a concept I'll need again (JWT, migrations, Riverpod providers, webhooks...), give a short explanation and suggest I write a note in `docs/notes/`.
  - Never dump huge amounts of code at once. If a feature needs many files, do them in order and pause so I can follow.
  - If I ask you to "just do it", you can move faster, but still summarize what changed.
- Prefer simple, standard, well-known solutions over clever ones. Code I don't understand is code I can't fix.
- Avoid long em dashes in any text that the client or end users will see.
- Ask me before: adding a new paid service, changing the stack, changing the database schema in a breaking way, or anything that touches money logic.

## The project

**Client:** Mr. Brand (مستر براند), Aleppo, Syria. Instagram @mr.brand_aleppo1. A well-known service with a call center that recommends the best places to eat, and also does delivery.

**Signed package (Package 3, quote SOL-2026-032):** Android + iPhone apps, no public website. $5,000 build, $1,600/year subscription from year two. Payments: 30% start, 30% at demo, 40% at launch. Subscription includes up to 5 work hours/month (extra $20/h) and AI up to 5,000 conversations/month.

**Ownership:** The code stays Solunera's (we will reuse it rebranded for future clients, so keep client-specific things in config/settings, not hard-coded). Client owns data, customers, content, name, brand, domain. Apps are published under the client's own Google Play and Apple developer accounts.

**Goal:** Better than Talabat, Toters and BeeOrder for Aleppo, by winning on:
1. The Mr. Brand voice: AI recommends places the way their call center does, in Syrian dialect.
2. Human backup: any chat or order can be handed to a real employee in one tap.
3. Built for Syrian conditions: fast on slow 3G, survives network/power drops, small app, Arabic first.
4. Reliability: orders never disappear, payments never charged twice, status always true.

Out of scope for this build (future paid upgrades): voice-note ordering, group orders, scheduled orders, automatic driver dispatch, public website.

## Roles (ONE app, different screens per role)

Login routes each user to their role's screens. Roles: `customer`, `employee`, `driver`, `admin`, `owner`.

- **Customer:** WhatsApp code login, restaurant directory (search, filters, categories, mood tags), restaurant pages + menus, cart + ordering, saved addresses with map pin, points, chat with the Mr. Brand AI assistant (with "talk to a person"), live order tracking, Sham Cash payment.
- **Employee:** take phone orders, see incoming orders, assign drivers MANUALLY, take over AI chats.
- **Driver:** go on/off shift, receive jobs, map + route, share live location only while on shift, update delivery status.
- **Admin / Owner:** control everything: restaurants, menus, prices, users, staff accounts, points rules, delivery fees, AI knowledge base, reports, settings.

## Tech stack (do not change without asking me)

**App: Flutter (Dart)**, one codebase for Android + iOS
- State: Riverpod | Navigation + role routing: go_router | HTTP: dio (retries, timeouts, token refresh)
- Realtime: socket_io_client | Models: freezed + json_serializable
- Local cache: drift (SQLite) | Secure tokens: flutter_secure_storage
- Maps: flutter_map + OpenStreetMap tiles (Google Maps is an option if the client pays) | Location: geolocator + foreground service for drivers
- Push: Firebase Cloud Messaging | Images: cached_network_image
- Localization: flutter_localizations + intl ARB files. **Arabic RTL is the primary design**, English second.
- Crashes: Sentry

**Server: TypeScript, Node.js LTS, NestJS**
- PostgreSQL 16+ with Prisma ORM, pgvector extension for AI search
- Redis: cache, rate limits, OTP limits, Socket.IO adapter
- BullMQ (on Redis) for background jobs: WhatsApp codes, push, payment checks, AI embeddings
- Socket.IO for realtime (order status, driver location, chat)
- Validation: class-validator DTOs | API docs: Swagger | Logs: pino | Tests: Vitest + Supertest | Lint: oxlint (NestJS 12 defaults, chosen Oct 14 over Jest + ESLint)

**External services**
- WhatsApp Cloud API (Meta): login verification codes ONLY (authentication templates)
- Sham Cash API: create bill, webhook (paid/expired), refunds, transactions. Requests are encrypted with an agent key + 32-byte secret issued by Sham Cash. Reference implementation (PHP): https://packagist.org/packages/oktocode/sham-cash-sdk. We port the logic to TypeScript once we have official docs and keys.
- LLM API (Claude or OpenAI) + embeddings: called ONLY from the server, never from the app
- Firebase FCM (push), Cloudflare (DNS, SSL, DDoS, R2 storage for images + backups), Sentry

**DevOps**
- Docker + Docker Compose; one VPS (4 vCPU / 8 GB) to start; Caddy reverse proxy behind Cloudflare
- GitHub + GitHub Actions; Codemagic for iOS builds (I don't have a Mac)
- Uptime Kuma, nightly pg_dump backups to R2 (30 days), k6 load testing
- Environments: local (my laptop, Docker), staging (same VPS, separate DB), production

## Repo layout

```
mrbrand/
  CLAUDE.md     this file
  app/          Flutter app (all roles)
  server/       NestJS API + BullMQ workers
  infra/        docker-compose, Caddy config, backup scripts
  docs/         decisions, API notes, daily log (docs/LOG.md), learning notes (docs/notes/)
```

## Database (main tables)

users (phone unique, role, status) | sessions (refresh tokens per device, revocable) | otp_requests (hashed code, 5 min expiry, max 5 tries) | addresses | restaurants (soft delete) | categories, tags | menu_items (price as INTEGER) | orders | order_items (price snapshot, immutable) | order_events (every status change, who, when) | deliveries (one active driver per order) | driver_locations (latest position, short history) | payments (Sham Cash bill id, idempotency key, webhook is source of truth) | points_ledger (balance = sum of ledger, never edited directly) | ai_conversations, ai_messages | knowledge_items (Q&A + embedding vector) | notifications | audit_logs | settings (fees, points rules, AI limits, editable without app update)

**Order life cycle:** pending_payment → placed → accepted → driver_assigned → picked_up → delivered, plus cancelled and refunded branches. The server rejects any transition that skips a step.

## Non-negotiable rules

**Scale (must handle 10k+ users, ~1,000 online at peak).** The client expects at most ~10k users and agreed to pay more if usage goes beyond that, but build so we can scale past it without rewrites.
- Stateless API (no user state in server memory), so we can run more instances later
- Index every column we filter or search by; paginate every list (default 20)
- Connection pooling; cache restaurants/menus in Redis, invalidate on edit
- Heavy work goes to BullMQ, never blocks a request
- Driver GPS throttled to every 5 to 10 s or 30 m
- Images resized + compressed (WebP) on upload, served from R2/CDN
- Rate limits per user and per IP; AI caps per user per day + 5,000 conversations/month

**Security**
- OTP: 6 digits, hashed, 5 min expiry, max 5 tries, max 3 codes per phone per hour, per-IP limits
- JWT access token 15 min + rotating refresh token in secure storage, revocable per device
- Role checks with NestJS guards on EVERY endpoint. Never trust the app to hide things.
- Every query scoped to the logged-in user where relevant; tests that try to read other users' data
- Server recalculates all prices and totals; ignore prices from the app
- Orders marked paid only from a verified Sham Cash webhook, double-checked against their API; idempotency keys everywhere money moves
- AI can only call our safe server functions (search restaurants, get menu, etc.), never sees other users' data
- Secrets only in server env vars, never in the app or git. `.env` is gitignored.
- Audit log every admin/employee action
- Driver location shared only while on shift, and to a customer only during their own delivery

**Real-world conditions in Syria**
- Assume slow, unstable internet and power cuts. Retries with backoff, cached screens, small payloads.
- Orders get a client-generated unique id so resending never duplicates
- App restores exact state after being killed
- Test on a low-end Android phone and on slow network
- Every error message says what happened and what to do, in plain Syrian Arabic
- Skeleton loaders, never blank screens; dark mode from the start

## How we build each feature (vertical slices)

1. Define it in 3 to 5 lines (who, what they see, what can go wrong)
2. Screen design (Figma or a sketch)
3. Prisma schema change + migration
4. NestJS module: controller, service, DTO, guard
5. Test the endpoint (Bruno): happy path, bad input, wrong role
6. Automated tests for important rules (money, points, order status)
7. Flutter: model → repository (API calls) → Riverpod provider → widget
8. Test on a real phone, including slow network
9. Commit on a feature branch, PR, review, merge
10. Deploy to staging, check, then production

At the end of each work session, append a short entry to `docs/LOG.md`: what works, what's next, what's stuck. At the start of a session, read the last few entries.

## Timeline (assumes start Oct 19, 2026)

| Phase | Dates | Delivers |
| --- | --- | --- |
| Setup + design | Oct 19 to Oct 30 | Repos, Docker, VPS, accounts started, DB schema, Figma customer flow, API list |
| Backend core + admin | Oct 26 to Nov 20 | WhatsApp login, roles, restaurants, menus, categories, admin endpoints, first staging deploy |
| Kemo's move (light week) | Nov 1 to Nov 7 | Minimal work |
| Customer app core | Nov 9 to Nov 27 | Login, home, restaurant pages, search/filters, cart, addresses, points, AR + EN |
| AI assistant | Nov 23 to Dec 4 | Knowledge base, pgvector search, tool calls into real directory, chat, human handoff, caps |
| **Demo gate (30% paid)** | Dec 7 | Client tries it on his phone |
| Orders, staff, drivers | Dec 7 to Dec 25 | Order life cycle, employee screens, manual dispatch, driver app, live tracking |
| Sham Cash + push | Dec 21 to Jan 1 | Bills, webhooks, refunds, push for every status |
| Hardening + beta | Jan 4 to Jan 15 | Security checklist (OWASP API + Mobile Top 10), k6 load tests, beta with staff + 20 to 50 customers |
| Store review | Jan 11 to Jan 22 | Listings, privacy policy, Play + App Store submission |
| **Launch (40% paid)** | Jan 25, 2027 | Go live, two weeks of close monitoring |

## Open questions (waiting on client)

- Cash on delivery as well as Sham Cash?
- Delivery fee rules: fixed, by area, or by distance?
- Points rules: earn rate and what they redeem for?
- Do restaurants get their own login, or do employees handle restaurants by phone?
- Google Maps (paid) or OpenStreetMap (free)?
- AI knowledge source: their Q&A file + exported Instagram replies?
- Registered business name for the store listings? Apple Developer enrollment from Syria still unconfirmed.

## Current status

- Contract signed Oct 14, 2026.
- Oct 14: laptop installs all verified (see docs/LOG.md). Repo at `C:\Users\kamel\code\mrbrand`, pushed to GitHub `kemo24n/mrbrand`.
- Oct 14: NestJS 12 scaffolded in `server/` (pnpm, strict TS, Vitest, oxlint) on branch `setup/server`.
- Next step: Docker Compose (Postgres + pgvector, Redis) in `infra/`, then Flutter in `app/`.

(Update this section as we go.)
