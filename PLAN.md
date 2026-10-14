# Mr. Brand: Full Build Plan (reference)

Companion to CLAUDE.md. CLAUDE.md has the rules and summary; this file has the full detail. Diagrams (architecture, timeline) live in the online plan doc.

## Architecture

```
[Flutter app: customer / employee / driver / admin / owner]
        |  HTTPS + WebSocket
        v
[Cloudflare (DNS, SSL, DDoS)] -> [Caddy reverse proxy]
        v
[Our server, one VPS, Docker]
   NestJS API (REST + Socket.IO)
   BullMQ workers (WhatsApp codes, push, payments, AI embeddings)
   PostgreSQL + pgvector      Redis (cache, queues, rate limits)
        |
        v  server calls them
[Outside services]
   WhatsApp Cloud API (login codes only) | Sham Cash API (payments + webhooks)
   Claude or OpenAI API (AI assistant)   | Firebase FCM (push)
   Cloudflare R2 (photos + backups)      | Map tiles + OSRM (maps, routes)
```

The app talks directly to outside services ONLY for map tiles and receiving push. Everything else (AI, payments, WhatsApp) goes through our server so keys and money logic never live on a phone.

## Install on the laptop

- [ ] Git
- [ ] VS Code + extensions: Flutter, Dart, Prisma, Oxc (oxlint), Prettier, Docker, GitLens, Error Lens
- [ ] Claude Code
- [ ] Flutter SDK (stable), run `flutter doctor` until all green
- [ ] Android Studio (Android SDK + emulator)
- [ ] Node.js LTS + pnpm
- [ ] NestJS CLI: `pnpm add -g @nestjs/cli`
- [ ] Docker Desktop (runs Postgres + Redis locally)
- [ ] DBeaver or TablePlus (database viewer)
- [ ] Bruno or Postman (API tester)
- [ ] Figma
- [ ] A real Android phone with USB debugging on
- No Mac needed: Codemagic builds iOS in the cloud and sends to TestFlight.

## Accounts

**Solunera's (Kemo's):** GitHub (private repos), Hetzner or similar VPS, Cloudflare, Sentry, Codemagic, Anthropic or OpenAI API (cost covered by the subscription), Firebase project.

**Client's (in his name, Kemo gets access):**
- Google Play Console ($25 once)
- Apple Developer Program ($99/year), organization account if he has a registered business
- Meta Business account + WhatsApp Business number for login codes
- Sham Cash merchant service (issues agent key + secret)
- Domain for the API (client owns it)

## Scale: growth path

10,000 registered users ≈ 300 to 1,000 online at the busiest dinner hour. One well-built server handles that.

| Stage | Users | Setup | Approx. hosting cost |
| --- | --- | --- | --- |
| Launch | up to 10k | 1 VPS: API + Postgres + Redis in Docker | $10 to $30/month |
| Growth | 10k to 50k | API on 2 servers + load balancer, Postgres on its own server | $60 to $120/month |
| Big | 50k+ | Managed Postgres + read replica, 3+ API servers, separate realtime server | $200+/month |

Costs are approximate; check current prices before quoting. The client expects at most ~10k users and agreed to pay more if usage exceeds that.

Before launch: k6 load test simulating 1,000 users browsing + 200 placing orders at once. Fix whatever breaks first.

## Security: full table

| Threat | What we do |
| --- | --- |
| Fake logins / code guessing | 6-digit code, hashed, 5 min expiry, max 5 tries, max 3 codes per phone per hour, per-IP limits |
| WhatsApp cost abuse (bots requesting codes) | Rate limits + device checks; alert if codes per hour spike |
| Stolen tokens | 15 min access token + rotating refresh token in secure storage, revocable per device |
| Customer opening admin screens | Server-side role guards on every endpoint |
| Seeing other people's orders | Every query scoped to logged-in user; tests that try to read others' data |
| Bad input / SQL injection | Validation on every request; Prisma parameterized queries |
| Fake payment confirmations | Paid only from verified Sham Cash webhook, re-checked against their API; idempotency keys |
| Price tampering | Server recalculates every total from the DB |
| AI misuse (prompt injection) | Bot only calls our safe functions, can't see other users' data, output + daily caps |
| Leaked secrets | Server env vars only, never in app or git |
| Server attacks | Cloudflare in front, only 80/443 open, SSH key only, automatic security updates |
| Data loss | Nightly encrypted backups off-server, monthly restore test |
| Staff abuse | Audit log of every admin/employee action; owner can disable any account instantly |
| Driver location privacy | Shared only on shift; customer sees driver only during their own delivery |

Before launch: go through OWASP Mobile Top 10 and OWASP API Security Top 10 line by line.

## Real-world problems in Syria

| Problem | How we handle it |
| --- | --- |
| Slow or dropping internet | Small responses, compressed images, retries with backoff, cached screens |
| Order sent but connection dies | Client-generated unique order id; resending never duplicates |
| Phone dies mid-delivery | State lives on the server; reopening restores exactly where you were |
| Driver loses signal | Show last known location with "updated 2 min ago"; queue positions, send when back |
| Old cheap Android phones | Test on a low-end phone; small app size; no heavy animations |
| WhatsApp code doesn't arrive | Resend after 60 s, clear message, employee can verify by phone as backup |
| Sham Cash payment pending/failed | Order waits in pending_payment with a timer; cash on delivery fallback if client agrees |
| Restaurant closed but listed open | Opening hours + manual "closed now" switch |
| AI gives a wrong answer | "Talk to a person" on every chat; owner corrects the knowledge base |
| Server goes down | Uptime alerts to Kemo's phone; friendly "back soon" screen |
| App update needed | Minimum-version check; force update only when old version would break |

UX principles: Arabic RTL primary, English checked on every screen; 3 taps from open to "order placed" for repeat customers; skeleton loaders; plain Syrian Arabic error messages; big touch targets for drivers; dark mode from the start.

## Learning roadmap (Kemo learns each skill in the week the project needs it)

After each skill, write a one-page note in `docs/notes/` in your own words.

| # | Skill | Must be able to do alone | When | Resource |
| --- | --- | --- | --- | --- |
| 1 | Git + GitHub | Branches, commits, PRs, undo mistakes | Week 1 | Pro Git book (free) |
| 2 | Linux + terminal | Navigate, SSH, read logs, permissions | Week 1 | Own VPS |
| 3 | TypeScript | Types, interfaces, async/await, generics basics | Weeks 1 to 2 | TypeScript Handbook |
| 4 | SQL + DB design | Tables, relations, indexes, joins | Week 1 | PostgreSQL docs, SQLBolt |
| 5 | Node.js + NestJS | Modules, controllers, services, guards, pipes | Weeks 2 to 3 | NestJS docs |
| 6 | Prisma | Schema, migrations, transactions | Week 2 | Prisma docs |
| 7 | REST API design | URLs, status codes, pagination, errors | Weeks 2 to 3 | Own Swagger output |
| 8 | Authentication | OTP, JWT access/refresh, roles | Week 2 | OWASP Authentication Cheat Sheet |
| 9 | Dart | Null safety, classes, async, streams | Week 3 | dart.dev language tour |
| 10 | Flutter | Widgets, layouts, RTL, forms, lists | Weeks 3 to 6 | flutter.dev docs + codelabs |
| 11 | Riverpod + go_router | State, role-based navigation | Weeks 3 to 4 | riverpod.dev |
| 12 | Realtime (WebSockets) | Rooms, events, reconnects | Week 8 | Socket.IO docs |
| 13 | Redis + queues | Caching, rate limits, BullMQ | Weeks 5 to 8 | Redis University, BullMQ docs |
| 14 | AI engineering | Prompts, RAG with pgvector, tool calling, cost control | Weeks 6 to 7 | Anthropic/OpenAI tool use + embeddings docs |
| 15 | Payments | Webhooks, idempotency, reconciliation | Weeks 9 to 10 | Sham Cash docs |
| 16 | Maps + location | Map display, background location, routing | Week 8 | flutter_map + geolocator docs |
| 17 | Docker | Dockerfiles, Compose, volumes, networks | Week 4 | Docker "Get started" |
| 18 | Deployment | VPS, Caddy, Cloudflare, GitHub Actions CI/CD | Weeks 4 + 11 | Do it twice: staging, then prod |
| 19 | Security | OWASP API + Mobile Top 10 | Week 11 | owasp.org |
| 20 | Testing | Unit, API, k6 load tests | Weeks 10 to 11 | Vitest, k6 docs |
| 21 | Monitoring | Sentry, logs, uptime, backups + restore | Week 11 | Own production server |
| 22 | App store publishing | Signing, listings, review rules, TestFlight | Weeks 12 to 13 | Play + App Store Review Guidelines |
| 23 | System design basics | Draw any app's architecture before coding | Ongoing | System Design Primer (GitHub) |
| 24 | Client work | Scoping, quotes, change requests, milestones | Ongoing | Every Solunera project |

Ready-to-work-alone test: after launch, start a small project (e.g. the BASE POS) from an empty folder and set up server, DB, app, deployment and backups yourself; ask for help only when stuck over an hour.

## Risks to start this week

| Risk | Why | Action |
| --- | --- | --- |
| Apple Developer account for a Syrian business | Enrollment often takes 2 to 7 weeks in 2026; acceptance from Syria unconfirmed | Client starts enrollment now; if blocked, discuss publishing under a company registered elsewhere |
| Sham Cash merchant access | API exists but keys only come from Sham Cash | Client requests merchant service + API docs; port encryption to TypeScript |
| WhatsApp Business verification | Meta must approve before auth messages send | Client opens Meta Business account now |
| Google services in Syria | Returned from Jan 2026 after sanctions lifted; confirm Play Console + FCM work | Test FCM from a Syrian SIM early |
| Kemo's move Nov 1 | Losing a week mid-build | Light week planned; laptop + VPS access ready before travel |
