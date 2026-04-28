# bellring-server — CLAUDE.md v2

**Date:** 2026-04-28 (S11B authoring)
**Supersedes:** v1 (commit-sha pending S11C verification)
**Tier:** B (production-touching for CN reference customer; off-fleet on Render today)

## Identity & Role

`bellring-server` is the **backend half of Bellring** — whitelabel SaaS for sales-team celebration notifications (sales-floor bell-ringing ritual; closing a deal triggers unmissable team-wide popups). Receives CRM webhooks (LeadSquared, HubSpot, Salesforce, generic), issues OTP auth, fans real-time celebration events to browser extensions over WebSocket. Tier model: Free / $19 Team / $79 Growth / $299+ Enterprise. Repo renamed from `sales-notification-backend` 2026-04-19; previously codenamed "Salvo". Vagary Labs brand: **Bellring** (product brand).

## Coverage Today (post-PCN-S6/S7/S11A)

Per matrix row `bellring-server` — **bare-fork stack-up cluster (Cluster 1)**:

```
Mail | DNS | RP | Orch  | Obs | Backup | Sup | Sec | Tun | Err | Wflw | Spec
 NA  | T   | NA | NA(R) | NA  | NA     | T   | U   | NA  | NA  | NA   | NA
```

- USED: Sec (Render env vars; pre-startup validation; `EXTENSION_ID` + `WEBHOOK_TOKEN` + `JWT_SECRET` + `SENDGRID_API_KEY`).
- TRIGGER-TO-WIRE: DNS (once `bellring.<tld>` registered), Sup (Cosign post-PR-#50 — T or N-A given Render runtime; if Bellring multi-tenant pivot moves to Coolify, becomes T proper).
- NA: Mail (SendGrid via API), RP, Obs, Backup, Tun, Err, Wflw, Spec.
- NA(R) = on Render off-fleet today; pending Bellring multi-tenant pivot decision (stay on Render OR move to Coolify on Vagary).

## What's Wired (Render free tier)

- **Production:** `https://sales-notification-backend.onrender.com` — Render free tier; auto-deploys on `main` push via `Procfile`. New Bellring-branded URL TBD once `.io/.app/.ai` domain chosen.
- **CI:** GitHub Actions — `node --check` + `npm audit --audit-level=critical` + .env.example var listing. **GREEN.**
- **CN reference customer:** ~300 BDEs; LeadSquared webhook integration.
- **SendGrid:** OTP email; sender `noreply@codingninjas.com` (verified).
- **Health endpoints:** `/ping` (keepalive), `/stats` (admin-bearer-gated).

## Stack

- **Runtime:** Node.js ≥16 (CI pins Node 20). Single-file `server.js` monolith. `--max-old-space-size=256` flag.
- **HTTP/API:** Express 4 + cors + express-rate-limit
- **Realtime:** `ws` library, path `/ws`, JWT-token query-param auth
- **Auth:** JWT (jsonwebtoken) 30-day expiry; OTP via SendGrid; `@codingninjas.com` domain allowlist hardcoded
- **Storage:** in-memory `Map` + `tokens.json` file-backed (60s flush)
- **Lint/tooling:** Biome 2 + Knip 6

## Roadmap (post-S11A)

### Cluster 3 — Cosign per-repo CI fanout
- T or N-A — depends on Bellring multi-tenant pivot decision. If repo stays on Render: N-A. If pivots to Coolify+Supabase: T (post host_page PR #50 merge).

### Bare-fork stack-up (Cluster 1; per dispatch §1.m)
- Pending operator decision on multi-tenant pivot venue:
  - **Path A — stay on Render** (CN-dedicated instance; this repo retires when Bellring multi-tenant lands as fresh `bellring-webhook-worker` repo).
  - **Path B — pivot in-place to Coolify+Supabase** (rewrite Express monolith as Hono on Coolify; delegate auth to Supabase; broadcast via Supabase Realtime). Triggers full bare-fork stack-up.

### Bellring multi-tenant pivot (authoritative spec: `~/.claude/specs/2026-04-19-sales-notification-whitelabel.md`)
- **Phase 0 (Week 0):** strip `@codingninjas.com` branding; remove base64 URL hack; register `bellring.<tld>` + brand site.
- **Phase 1 (Weeks 1-2):** Hono on Coolify + Supabase Auth + Postgres + Realtime.
- **Phase 2 (Week 3):** Stripe tiers via MCP; PostHog event taxonomy; Sentry browser SDK.
- **Phase 3 (Week 4):** Chrome Web Store + Firefox + Edge submissions; privacy + ToS + DPA pages; docs site `docs.bellring.<tld>`.
- **Phase 4 (Weeks 5-6):** CRM adapters (LeadSquared P0, Salesforce + HubSpot P1, Pipedrive + Close P2); Product Hunt launch.
- **Phase 5 (ongoing):** SAML/SCIM (Enterprise), analytics v2, custom animation uploader, self-host Docker image, SOC 2 Type II if Enterprise pipeline >3 deals.

## ADR Compliance

- **ADR-038 personal-scope:** ✓ — Cramraika org; SMPL562 retired 2026-04-19.
- **ADR-033 Renovate canonical:** T (pending bare-fork stack-up).
- **ADR-041 Trivy gate:** T or N-A (Render runtime; reassess at pivot).
- **SOC2 risk-register cross-ref:** off-fleet runtime (Render) = LOW SOC2 evidence; mitigation = pivot decision.

## Cross-references

- `platform-docs/05-architecture/part-B-service-appendices/products/bellring-server.md` (pending S11B authoring)
- `~/.claude/specs/2026-04-19-sales-notification-whitelabel.md` (Bellring spec)
- `~/.claude/specs/2026-04-19-brand-rename-proposal.md` (Salvo → Bellring)
- Pair repo: `bellring-extension`
- `~/.claude/conventions/universal-claudemd.md` §41 brand architecture (Bellring)
- `~/.claude/conventions/design-system.md` (Tier A; bellring-yellow primary)

## Migration from v1

**Major v1 → v2 changes:**
1. Per-project-service-matrix row added — **bare-fork stack-up cluster** (or N-A if Path A); deployment-venue note added (Render).
2. Path A vs Path B decision flagged as pending operator.
3. Cosign per-repo CI fanout T-or-N-A flag (depends on pivot decision).
4. Off-fleet status (Render today) noted explicitly.
5. Bellring brand architecture §41 cited.
