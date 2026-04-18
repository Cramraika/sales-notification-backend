# Sales Notification Backend

## Claude Preamble
<!-- VERSION: 2026-04-19-v9 -->
<!-- SYNC-SOURCE: ~/.claude/conventions/universal-claudemd.md -->

**Universal laws** (§4), **MCP routing** (§6), **Drift protocol** (§11), **Dynamic maintenance** (§14), **Capability resolution** (§15), **Subagent SKILL POLICY** (§16), **Session continuity** (§17), **Decision queue** (§17.a), **Attestation** (§18), **Cite format** (§19), **Three-way disagreement** (§20), **Pre-conditions** (§21), **Provenance markers** (§22), **Redaction rules** (§23), **Token budget** (§24), **Tool-failure fallback** (§25), **Prompt-injection rule** (§26), **Append-only discipline** (§27), **BLOCKED_BY markers** (§28), **Stop-loss ladder** (§29), **Business-invariant checks** (§30), **Plugin rent rubric** (§31), **Context ceilings** (§32), **Doc reference graph** (§33), **Anti-hallucination** (§34), **Past+Present+Future body** (§35), **Project trackers** (§36), **Doc ownership** (§37), **Archive-on-delete** (§38), **Sponsor + white-label** (§39), **Doc-vs-code drift** (§40).

**Sources**: `~/.claude/conventions/universal-claudemd.md` (laws, MCP routing, lifecycle, rent rubric, doc-graph, anti-hallucination) + `~/.claude/conventions/project-hygiene.md` (doc placement, cleanup, archive-on-delete, ownership matrix). Read relevant sections before significant work. Re-audit due **2026-07-19**. Sync: `~/.claude/scripts/sync-preambles.py`.

---

## 1. Business Line

Real-time sale-celebration backend for the Sales Notification Chrome Extension (~300 Coding Ninjas BDEs). Receives LeadSquared webhooks, issues OTP auth via SendGrid, fans notifications to connected extension clients over WebSocket.

## 2. References

- `~/.claude/conventions/universal-claudemd.md` — universal laws, MCP routing, lifecycle
- `~/.claude/conventions/project-hygiene.md` — doc placement, cleanup, local-workspaces
- `~/.claude/specs/2026-04-19-sales-notification-whitelabel.md` — **Salvo** white-label/multi-tenant pivot spec (roadmap Phase 2+)
- Sibling repo: `/Users/chinmayramraika/Documents/Github/sales-notification-extension/` — Chrome MV3 consumer (v1.0.4)

## 3. Stack

- **Runtime**: Node.js ≥16 (CI pins Node 20). `server.js` is a single-file monolith.
- **HTTP/API**: Express 4 + `cors` + `express-rate-limit`
- **Realtime**: `ws` library, path `/ws`, JWT-token query-param auth
- **Auth**: JWT (`jsonwebtoken`) with 30-day expiry; OTP sign-in via SendGrid (`@sendgrid/mail`); `@codingninjas.com` domain allowlist hardcoded
- **Storage**: in-memory `Map` (OTPs, clients, connection state) + `tokens.json` file-backed persistence via `FileBackedStorage` (60s flush)
- **Memory**: `--max-old-space-size=256` (256MB cap); `MemoryManager` auto-cleanup at 200MB
- **Lint/tooling**: Biome 2 + Knip 6 (devDeps; no npm scripts wired yet)

## 4. Active Role-Lanes

- **Engineer** — primary (webhook route, WebSocket lifecycle, auth middleware, memory cleanup)
- **SRE** — secondary (Render cold-start mitigation, SendGrid rate limits, WebSocket reconnect storms)
- **Architect** — active for Salvo pivot (multi-tenant data model, Supabase migration, adapter surface)
- **Security** — ongoing (webhook bearer, JWT secret rotation, CORS/origin checks, rate limit tuning)
- **Manager / Writer** — dormant (no end-user docs beyond README + ENVIRONMENTS.md)
- **Designer / Analyst / Marketer** — out of scope

## 5. Build / Test / Deploy

```bash
npm install                 # install deps
npm run dev                 # nodemon + 256MB heap cap
npm start                   # prod mode: node --max-old-space-size=256 server.js
```

- **Tests**: none present. Webhook flow is validated manually per `docs/ENVIRONMENTS.md` curl snippet.
- **CI**: `.github/workflows/ci.yml` — `node --check` on all `.js` + `npm audit --audit-level=critical` + `.env.example` var listing. Status GREEN (see `~/Documents/Github/CLAUDE.md` CI table).
- **Deploy**: Render free tier — auto-deploys on push to `main` via `Procfile: web: node server.js`. Env vars set in Render dashboard.

## 6. Key Directories

```
sales-notification-backend/
├── server.js           # all logic (Express, WebSocket, memory classes)
├── Procfile            # Render deploy
├── package.json        # deps + scripts
├── .env.example        # 6 vars: PORT, NODE_ENV, EXTENSION_ID, WEBHOOK_TOKEN, JWT_SECRET, SENDGRID_API_KEY + n8n
├── docs/
│   └── ENVIRONMENTS.md # deploy + troubleshooting reference
├── .github/workflows/
│   └── ci.yml          # syntax + audit + env check
└── .claude/
    └── settings.json   # permission denies + disabled-plugin allowlist
```

No `src/`, `test/`, `scripts/`, `docs/specs/`, or `docs/plans/` — the project is a single-file service. On the Salvo pivot this fans out into separate repos (`salvo-webhook-worker`, `salvo-dashboard`, `salvo-extension`, `salvo-landing`); do NOT restructure this repo in place.

## 7. Dependency Graph

**Consumed by** (downstream):
- `sales-notification-extension` (sibling repo, Chrome MV3) — connects to `/ws` with JWT, calls `/request-otp` + `/verify-otp`, receives webhook broadcasts.

**Depends on** (upstream / external):
- **SendGrid** — OTP email delivery. Sender: `noreply@codingninjas.com` (verified sender required).
- **LeadSquared** — fires webhook on sale-close (authenticates via `WEBHOOK_TOKEN` bearer).
- **Render free tier** — hosting. Single instance, ephemeral FS, 15-min idle spin-down.
- **Chrome Web Store** — extension `EXTENSION_ID` is the CORS origin allowlist.

**Upstream-of-upstream**: on Salvo pivot the graph rewires to Supabase Postgres + Realtime + Auth and Coolify/Vagaryvoice hosting (see spec §3.3-3.7).

## 8. Known Limitations

- **OTPs in-memory** — lost on Render restart; users must re-request. No DB yet.
- **No message persistence** — offline BDEs miss notifications entirely. No queue replay on reconnect.
- **Render cold start** — free tier spins down after 15 min idle; first request takes ~30s. `/ping` keepalive partially mitigates.
- **Single-instance WebSocket** — max 50 clients per process (`ClientManager.maxClients`); no horizontal scaling, sticky sessions, or pub/sub.
- **SendGrid rate limits** — free tier 100 emails/day; tight against 300-BDE onboarding waves.
- **Hardcoded `@codingninjas.com` domain** — blocks white-label reuse without code change (see Salvo spec §3.2).
- **IP-based rate limit** — corporate NAT / shared IPs can trip the 100 req/15min cap across distinct users.
- **File-backed tokens** — `tokens.json` lives on Render ephemeral FS; survives soft restarts, not instance recycles.
- **No integration tests** — CI is syntax + audit only.

## 9. Security & Secrets

- **Secrets** (required env vars, all validated at startup with `process.exit(1)` on miss): `EXTENSION_ID`, `WEBHOOK_TOKEN`, `JWT_SECRET`, `SENDGRID_API_KEY`. Optional: `N8N_WEBHOOK_URL`, `N8N_API_KEY`.
- **Storage**: Render dashboard (prod) + `.env` locally (gitignored). `.env.example` committed.
- **NEVER commit** `.env`, `.env.*`, `.claude/settings.local.json`, `.mcp.json`, `tokens.json` (all in `.gitignore`).
- **Rotation**: `JWT_SECRET` rotation invalidates every issued token (users must re-verify OTP). `WEBHOOK_TOKEN` rotation requires coordinated LeadSquared config update.
- **Auth boundaries**:
  - `/webhook` — `WEBHOOK_TOKEN` bearer only (LeadSquared trusts nothing else).
  - `/request-otp`, `/verify-otp` — open but rate-limited + domain-gated + bearer optional.
  - `/ws` — JWT in query param + origin check (`chrome-extension://${EXTENSION_ID}` only).
- **Never skip hooks (`--no-verify`)** when committing. This repo is on SMPL562 — push is blocked by invalid PAT anyway (commit locally; push is user-ops).

## 10. Deployment Environments

| Env | Host | URL | Notes |
|---|---|---|---|
| Local dev | `localhost:3000` | `http://localhost:3000` | `npm run dev` (nodemon) |
| Production | Render free tier | `https://sales-notification-backend.onrender.com` | Auto-deploys on `main` push; /ping keepalive recommended |

Full reference: `docs/ENVIRONMENTS.md`.

## 11. External Services (MCPs, integrations)

Routing hints for this repo (see universal §6 for the full map):

| Service | MCP | Use for |
|---|---|---|
| Sentry (`vagary-life-pvt-ltd`) | `mcp__sentry__*` | Prod errors, WebSocket disconnect storms. Currently off by default in `.claude/settings.json` (plugin disabled); enable per-session when investigating |
| SendGrid | — | No MCP. CLI/API debugging via `curl https://api.sendgrid.com` with `SG.` bearer. Check dashboard for bounce/block state |
| Render | — | Dashboard only. No MCP. Use Render web UI for logs + env vars |
| Grafana / Loki | `mcp__grafana__*` / `mcp__loki__*` | Optional GlitchTip/Loki log queries if server is wired to Vagary observability stack |
| PostHog | `mcp__posthog__*` | Not currently instrumented; Salvo pivot adds it |
| Linear | `mcp__linear__*` | Existing "Index of News" + "ASM" projects; no Salvo project yet (spec Phase 4) |
| GitHub (SMPL562) | `$SMPL562_PAT` | PAT currently INVALID — local commits only; push is blocked. Flag at every push attempt |
| n8n | webhook | `N8N_WEBHOOK_URL=https://n8n.chinmayramraika.in` + `N8N_API_KEY` header |

**Observability**:
- **Render dashboard** — primary log source (web + cron). Reference-only, no SSH.
- **GlitchTip** (if `SENTRY_DSN` wired) — error tracker on Vagaryvoice; else Sentry org above.
- **Grafana** — alert candidates: WebSocket reconnect rate, `/webhook` 5xx, heap > 200MB (already logged by `MemoryManager`).
- **No APM / tracing** — MV3 service-worker cadence makes OpenTelemetry marginal.

## 12. Past Incidents / History (git-observable)

- **Render free-tier memory** drove the `MemoryManager` (`cleanupRoutines`, 200MB warn + `global.gc`, LRU eviction in `connectionsPerToken`). Artifact: `--max-old-space-size=256` flag in both scripts.
- **Stateless JWT migration** (`ab4497c`) replaced persistent server-side tokens to survive Render instance recycles — `tokens.json` now a soft-persist cache, not source of truth.
- **Ping rate fixes** (`8c3212d`) — extension keepalive was tripping WebSocket rate limits; relaxed `MIN_CONNECTION_INTERVAL` from 30s→5s, `MIN_PING_INTERVAL` held at 30s.
- **IP → token pivot** (`160a0ba`) — rate limits moved off IP (corporate NAT false positives) onto token identity.
- **Security patch wave** (`43eab72`) — 9 npm vulns patched in one sweep, Mar 2026.
- **CI quality upgrade** (`0739e00`) — lifted to ASM-grade syntax + audit + env validation (Mar 2026).

## 13. Roadmap

**Near-term (user-driven, probably Q2 2026):**
- **Salvo white-label pivot** — see `~/.claude/specs/2026-04-19-sales-notification-whitelabel.md`. Phase 0-4 mapped: fork to `Cramraika/salvo-webhook-worker` + adjacent repos, rewrite Express monolith as Hono on Coolify, delegate auth to Supabase, broadcast via Supabase Realtime. This repo stays as the Coding Ninjas production instance or gets retired once the extension points at `api.salvo.sh`.
- **Supabase migration path** — auth + Postgres + Realtime. Decommissions `MemoryManager`, `ClientManager`, `FileBackedStorage`, OTP/JWT handling.
- **Stripe billing** — four tiers (Free / $19 Team / $79 Growth / $299+ Enterprise). Products + prices created via `mcp__stripe__create_product`.
- **Multi-tenant data model** — `workspaces` × `memberships` × `notification_events` × `adapter_configs` + RLS.

**Later / deferred:**
- CRM adapters: LeadSquared (P0, existing prod logic), HubSpot (P1), Salesforce (P1), Pipedrive (P2), Close (P2). Generic canonical webhook + Zapier/n8n/Make recipes Day 1.
- Firefox + Edge extension builds (MV3 compatible).
- SAML/SCIM (Enterprise tier, Phase 5+).
- SOC 2 Type II (stretch; only after 3+ Enterprise deals).

**Out-of-scope:** mobile app (explicitly rejected per spec §7 Phase 5 — browser-extension-first is the wedge).

## 14. Deviations from Universal Laws

None. This repo follows universal laws as-is. Two repo-specific notes:
- **No push** from this Claude instance — SMPL562 PAT is invalid. Commit locally; user handles push.
- **README is frozen** per `project-hygiene.md` → README-curation section (`sales-notification-*` listed as "internal ops tools — frozen"). Do not edit `README.md` without explicit user request; edit `docs/ENVIRONMENTS.md` for ops changes instead.

## 15. Plugin Profile & Context

- `.claude/settings.json` lists ~180 plugins, all `false` (the "scripts"/"node-backend" profile pattern — everything off by default, enable per-session via `plugin-profiles.sh`).
- Per-project MCP disables: none currently. Figma/Stitch would be candidates to disable if pure backend work (no UI surface), but the Salvo pivot brings landing + dashboard design in-scope, so leave enabled.
- `.vscode/` is gitignored but present locally — do not commit.
