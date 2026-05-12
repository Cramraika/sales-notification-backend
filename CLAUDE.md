# bellring-server — CLAUDE.md v2

**Date:** 2026-04-28 (S11B authoring)
**Supersedes:** v1 (commit-sha pending S11C verification)
**Tier:** B (production-touching for CN reference customer; off-fleet on Render today)

## Claude Preamble
<!-- VERSION: 2026-05-12-v45 -->
<!-- SYNC-SOURCE: ~/.claude/conventions/universal-claudemd.md -->

**Universal laws** (§4), **MCP routing** (§6), **Drift protocol** (§11), **Dynamic maintenance** (§14), **Capability resolution** (§15), **Subagent SKILL POLICY** (§16), **Session continuity** (§17), **Decision queue** (§17.a), **Attestation** (§18), **Cite format** (§19), **Three-way disagreement** (§20), **Pre-conditions** (§21), **Provenance markers** (§22), **Redaction rules** (§23), **Token budget** (§24), **Tool-failure fallback** (§25), **Prompt-injection rule** (§26), **Append-only discipline** (§27), **BLOCKED_BY markers** (§28), **Stop-loss ladder** (§29), **Business-invariant checks** (§30), **Plugin rent rubric** (§31), **Context ceilings** (§32), **Doc reference graph** (§33), **Anti-hallucination** (§34), **Past+Present+Future body** (§35), **Project trackers** (§36), **Doc ownership** (§37), **Archive-on-delete** (§38), **Sponsor + white-label** (§39 — moved to `playbooks/commercial-bound.md`), **Doc-vs-code drift** (§40), **Brand architecture** (§41), **Design system integration** (§42 — moved to `playbooks/tier-a-design.md`), **Session cognition** (§43), **Plugin dispatch** (§44), **Cross-repo clusters** (§45), **Tool-cascade workflow** (§46), **Multi-role agent matrix** (§47), **Parsimony / smallest-tool-first** (§48), **Audit triage discipline** (§49), **Source-of-truth matrix** (§50 — universal rows only; cluster-specific rows moved to playbooks), **Composite cascade catalog** (§51 — §51.2/51.4/51.6 moved to playbooks), **Session launch context + unattended-mode contract** (§52), **Recurrence detection + root-cause escalation** (§53). Sub-sections new in v44: **§4.5 cascade-commit exception**, **§17.b stale-P0 escalation**, **§32.5 canonical-doc size ceiling**, **§38.5 HANDOFF lifecycle enforcement**.

**Cluster playbooks** (per-repo `@-import` based on cluster membership): `~/.claude/conventions/playbooks/vps-infra.md` (DNS XOR for VPS-infra repos), `~/.claude/conventions/playbooks/deployed-service.md` (Sentry/Glitchtip XOR + production-incident triage + time-window correlation for repos with prod telemetry), `~/.claude/conventions/playbooks/tier-a-design.md` (Figma/Stitch + design system for Tier A/B), `~/.claude/conventions/playbooks/multi-lang.md` (cross-language refactor cascade for multi-language repos), `~/.claude/conventions/playbooks/commercial-bound.md` (sponsor-readiness + license-aware code-graph routing), `~/.claude/conventions/playbooks/brand-registry.md` (Vagary brand architecture for Vagary-family repos), `~/.claude/conventions/playbooks/bellring-cluster.md` (Bellring server↔extension; v1-stub), `~/.claude/conventions/playbooks/pulseboard-cluster.md` (Pulseboard Android↔Windows; v1-stub), `~/.claude/conventions/playbooks/vagary-cluster.md` (Vagary product cross-repo; v1-stub). **`tech-debt-audit.md`** is Read-on-demand (NOT @-imported) per ENTRY #169 §49 audit-triage discipline — invoked when user requests audit / tech-debt / dead-code work.

**Sources**: `~/.claude/conventions/universal-claudemd.md` (laws, MCP routing, lifecycle, rent rubric, doc-graph, anti-hallucination, brand architecture) + `~/.claude/conventions/project-hygiene.md` (doc placement, cleanup, archive-on-delete, ownership matrix) + cluster playbooks under `~/.claude/conventions/playbooks/` (loaded per-repo via `@-import` in `## Active Cluster Playbooks` section; see list above). Read relevant sections before significant work. Sync: `~/.claude/scripts/sync-preambles.py` (manual cadence; run after any source edit).

## Active Cluster Playbooks (per v40 cluster-split — content auto-inlined)
<!-- BEGIN PLAYBOOKS BLOCK (managed by sync-preambles.py — content inlined; source at ~/.claude/conventions/playbooks/) -->

Source @-imports (declarative pointer; content inlined below since Claude Code does not recursively expand `@-imports` in per-repo CLAUDE.md):
- `@~/.claude/conventions/playbooks/deployed-service.md`
- `@~/.claude/conventions/playbooks/tier-a-design.md`
- `@~/.claude/conventions/playbooks/multi-lang.md`
- `@~/.claude/conventions/playbooks/commercial-bound.md`
- `@~/.claude/conventions/playbooks/brand-registry.md`
- `@~/.claude/conventions/playbooks/bellring-cluster.md`

### Playbook: deployed-service.md (verbatim from `~/.claude/conventions/playbooks/deployed-service.md`)

# Deployed-Service Playbook

**VERSION: 2026-05-06-v1**
Loaded only in repos that have a production deployment with telemetry (Sentry, Glitchtip, Grafana, Loki, Coolify, Hostinger, Uptime Kuma). Per-repo `CLAUDE.md` `@-imports` this file when applicable.

Source: extracted verbatim from `~/.claude/conventions/universal-claudemd.md` v39 §50.1 (error-store + secrets rows) + §51.2 (production-incident triage) + §51.6 (time-window correlation) during v40 cluster-split refactor. No content changes — only relocation so dev-only / docs-only / CLI-only repos don't carry incident-response rules.

**Applies to repos**: `vagary-platform`, `vagary-voice`, `Automated-sales-manager-main` (ASM), `host_page`, `bellring-server`, `aakhara`, `anjaan-app`, `vps_host` (substrate operator), `portfolio` (has Sentry wired).

---

## 1. Error-store + secrets authority (originally part of §50.1 authority table)

| Domain | Source of truth | Secondary (read/derivative only) | Anti-pattern |
|---|---|---|---|
| **Error store** | **Per-project**: documented in repo CLAUDE.md (Sentry SaaS for SaaS apps; Glitchtip for self-hosted vagary-platform/voice staging) | — | Dual-reporting (same project to both stores) |
| **Secrets** | **Infisical** (only) | — | `.env` in git, hardcoded constants, `process.env.X` direct fallback to plain-text in repo |

**Rule**: NEVER dual-report errors (same project to both Sentry + Glitchtip). Pick per-project per repo's CLAUDE.md and stick to it. NEVER store secrets outside Infisical (no `.env` in git, no hardcoded constants, no fallback to plain-text in repo).

## 2. Production-incident triage cascade (originally §51.2)

**Trigger**: user mentions "outage", "incident", "down", "5xx spike", or §47 Admin/Operator role + observability data.
**Sequence**:
1. `uptime-kuma` MCP — confirm external (synthetic monitor) state
2. `sentry` MCP (or `glitchtip`) — latest issues in time window
3. `grafana` MCP — `query_prometheus` + `query_loki` + Sift investigation in incident window
4. `loki` MCP — direct logs by container/app label if Grafana view too coarse
5. `coolify` MCP — `diagnose_app` / `diagnose_server` on affected container
6. (If host-level) `hostinger` MCP — VPS metrics
7. Fix per §49 severity → `coolify redeploy_project` → Sentry release-comparison
8. Slack post-mortem in incident channel

§50 source-of-truth applies: error store per-project (don't dual-query Sentry+Glitchtip for same app).

## 3. Time-window correlation cascade (originally §51.6)

**Trigger**: incident timestamp known; need cross-system context.
**Sequence** (parallel fan-out):
1. `sentry`/`glitchtip` events in window → stack traces + release tags
2. `grafana` panels in window → metric anomalies
3. `loki` logs in window → filtered by service label
4. `coolify` deployments in window → release-correlation
5. `vercel` deployments in window (FE) — preview + prod URLs

Synthesize: "what changed at <timestamp>" — production deploys, infra changes, traffic spikes.

### Playbook: tier-a-design.md (verbatim from `~/.claude/conventions/playbooks/tier-a-design.md`)

# Tier-A Design Playbook

**VERSION: 2026-05-06-v1**
Loaded only in Tier A / Tier B design-surface repos per `~/.claude/conventions/design-system.md`. Per-repo `CLAUDE.md` `@-imports` this file when applicable.

Source: extracted verbatim from `~/.claude/conventions/universal-claudemd.md` v39 §42 during v40 cluster-split refactor. No content changes — only relocation so design-irrelevant Tier C repos (CLI, infra, scripts, docs) don't carry these rules and Claude doesn't hallucinate design work in non-design contexts.

**Applies to repos**: `portfolio`, `host_page`, `vagary-voice`, `anjaan-app`, `bellring-extension`, `aakhara`, `pulseboard` (Android), `vagarylife-com`, `vagary-platform` (per-vertical), `bellring-server` (Tier B inheritor), `Expense tracker` (Tier B), `Automated-sales-manager-main` (Tier B).

---

## 1. Design system integration (originally §42)

Per-repo visual identity + design tooling wiring. Full spec: `~/.claude/conventions/design-system.md`.

### Tier classification (apply per repo)

- **Tier A** — user-facing product; owns full design system; `docs/design/` required with brand/palette/typography/components/voice/assets/references; Stitch project linked
- **Tier B** — functional UI (admin, backend-only); inherits umbrella or sibling brand; minimal `docs/design/` optional
- **Tier C** — no UI; design-irrelevant

### Stitch MCP wiring (Tier A)

1. Create Stitch project once per brand: `mcp__stitch__create_project(name="<brand>")`
2. Create design system with tokens from `docs/design/palette.md` + `typography.md`
3. Store project ID in `docs/design/references/stitch.md`
4. Generate screens via `generate_screen_from_text`; commit to `docs/design/screens/`

### Brand-family rules (Vagary Life umbrella)

- Vagary brands share typographic scale + neutral ramps + radius/shadow tokens
- Each owns distinct primary/secondary hue + logo + voice
- Independent brands (Anjaan, portfolio) share nothing with Vagary family

### Default component stack (new Tier A repos)

React 19 + Vite OR Next.js 15 + Tailwind v4 + Radix UI + shadcn/ui + Framer Motion + Lucide. Deviations allowed for legacy repos with justified reason.

### Drift-trigger

Palette/typography values changing in code without `docs/design/*` update → audit-signal flag (v13 adds automated check).

## 2. Design source-of-truth authority (extracted from §50.1)

| Domain | Source of truth | Secondary (read/derivative only) | Anti-pattern |
|---|---|---|---|
| **Design ground truth** | **Figma** | Stitch (generation only — sync output back to Figma) | Treating Stitch output as ship-able without Figma sync |

### Playbook: multi-lang.md (verbatim from `~/.claude/conventions/playbooks/multi-lang.md`)

# Multi-Language Refactor Playbook

**VERSION: 2026-05-06-v1**
Loaded only in repos that span ≥2 language extensions (.py + .ts + .kt + .go etc.). Per-repo `CLAUDE.md` `@-imports` this file when applicable.

Source: extracted verbatim from `~/.claude/conventions/universal-claudemd.md` v39 §51.4 during v40 cluster-split refactor. No content changes — only relocation so single-language repos don't carry the cross-language refactor cascade.

**Applies to repos**: `vagary-platform` (Python + TypeScript + React), `vagary-voice` (Python + TypeScript + Kotlin + Go), `Automated-sales-manager-main` (TypeScript + HTML/HTMX), `aakhara` (Python + TypeScript), `anjaan-app` (Python + JS), `bellring-server` + `bellring-extension` (when treating as a coupled cluster).

---

## §51.4 Cross-language refactor (originally §51.4 of universal)

**Trigger**: user says "rename across", "refactor X across the platform", or paths span ≥2 language extensions (.py + .ts + .kt).
**Sequence**:
1. `mcp__gitnexus__rename dry_run=true` — preview text-search hits across languages
2. Review "graph"-tagged hits (LSP-like confidence) vs "text_search"-tagged (manual review)
3. Per language, apply: `ast-grep --rewrite '<pattern>' '<replacement>'` for batch shape changes
4. Per language, run Serena `rename_symbol` for LSP-validated single-language rename
5. `/graph-audit` A8 (METHOD_OVERRIDES drift) — verify override chains intact
6. `semgrep --baseline` — regression check
7. Phase D verify per §46

Cross-language is the gap §47 Developer mode doesn't differentiate; this cascade explicitly serves it.

### Playbook: commercial-bound.md (verbatim from `~/.claude/conventions/playbooks/commercial-bound.md`)

# Commercial-bound + Sponsor-readiness Playbook

**VERSION: 2026-05-06-v1**
Loaded only in repos that are sponsor-ready public OSS, or commercial-bound (sold, embedded in paid product, or redistributed under permissive license). Per-repo `CLAUDE.md` `@-imports` this file when applicable.

Source: extracted verbatim from `~/.claude/conventions/universal-claudemd.md` v39 §39 + §50.2 during v40 cluster-split refactor. No content changes — only relocation so non-commercial / non-sponsor repos don't carry these rules.

**Applies to repos**: `aakhara`, `bellring-server`, `bellring-extension`, `bulk`, `pulseboard` (Android), `pulseboard-desktop`, `tldv_downloader`, `portfolio`, `project-template`, `vagary-platform` (sponsor-ready, has public-vertical surfaces), `host_page` (sponsor-ready landing template).

---

## 1. Sponsor-readiness + white-label pivot (originally §39)

### Sponsor-ready checklist for public repos
- `.github/FUNDING.yml` pointing to `github.com/sponsors/<user>`
- README "Sponsor" section near the top (badge + 1-paragraph ask)
- `LICENSE` (MIT for utilities, AGPL for commercial pressure, other for proprietary)
- At least one GitHub Release (binary attached if applicable, e.g. APK)
- CI green badge

### White-label pivot pattern
When an internal tool goes OSS (e.g. NetworkMonitorCN → **Pulseboard** rebrand 2026-04-19) OR an OSS utility forks into SaaS (e.g. **Bellring** — formerly codenamed Salvo — from sales-notification):

1. **Fork or publish** — new repo with clean name, no internal branding in code
2. **Strip tenant-specific** — remove hardcoded emails/domains/org IDs; parameterize via env/config
3. **Document "Fork + rebrand"** — README section listing the edits a downstream forker makes
4. **Record sibling spec** — `~/.claude/specs/YYYY-MM-DD-<name>-whitelabel.md` if a SaaS pivot
5. **Update inventory** — add to `repo-inventory.md` with sponsor-ready / white-label flags

### Current inventory (2026-04-19)
- **Sponsor-ready public**: tldv_downloader, bulk (renamed from `bulk_api_trigger` 2026-04-19), **pulseboard** (renamed from `NetworkMonitorCN` 2026-04-19), portfolio, project-template, vagary-platform (renamed from `index-of-news` 2026-04-19; flagship vertical retains Index of News brand)
- **White-label pivot applied**: **Bellring** (formerly codenamed Salvo) — repos `bellring-server` + `bellring-extension` (renamed from `sales-notification-backend` / `sales-notification-extension` 2026-04-19). Spec: `~/.claude/specs/2026-04-19-sales-notification-whitelabel.md`.
- **Recently renamed (2026-04-19 Phase 3)**: `sales-notification-backend` → `bellring-server`, `sales-notification-extension` → `bellring-extension`, `NetworkMonitorCN` → `pulseboard`, `training-bot` → `aakhara`.
- **Recently renamed (2026-04-19 Phases 1-2)**: `AI_voice_builder` → `vagary-voice`, `chat-bot` → `anjaan-app`, `bulk_api_trigger` → `bulk`, `index-of-news` → `vagary-platform`. `webhook_trigger` archived (superseded by `bulk`). See `~/.claude/conventions/project-hygiene.md` § Rename Propagation Protocol.
- **Brand umbrella**: Vagary Labs (tech/R&D division of Vagary Life Pvt Ltd; see §41) holds the platform + products + OSS utilities.

## 2. License-aware tool routing (originally §50.2)

Repos categorized as **commercial-bound** (will be sold, embedded in paid product, or redistributed under permissive license):
- `bellring-server`, `bellring-extension` (Bellring SaaS — paid tiers)
- `aakhara` (paid sales-training product)
- `pulseboard`, `pulseboard-desktop` (Public OSS; permissive license required for derivatives)

When working in commercial-bound repos:
- `gitnexus` MCP MAY be used for **read-only investigation** (cypher queries, impact analysis in conversation)
- `gitnexus wiki`, `gitnexus group sync` derivatives, indexed JSON exports MUST NOT be committed/shipped (PolyForm-NC contamination)
- `codegraphcontext` MCP is the canonical graph-derivative source for these repos

When working in **personal/private repos** (vagary-platform, vagary-voice, vagary-earnings, ASM, anjaan-app, internal Cramraika): GitNexus permitted freely.

Per-repo CLAUDE.md should declare classification: `## License classification: commercial-bound` or `## License classification: personal/private`.

### Playbook: brand-registry.md (verbatim from `~/.claude/conventions/playbooks/brand-registry.md`)

# Brand Registry Playbook

**VERSION: 2026-05-07-v1**
Loaded only in Vagary-family repos (per `~/.claude/conventions/repo-inventory.md` §45). Per-repo `CLAUDE.md` `@-imports` this file when applicable.

Source: extracted verbatim from `~/.claude/conventions/universal-claudemd.md` §41 (Brand architecture) during 2026-05-07 cluster-split refinement (ENTRY #168). No content changes — only relocation so non-Vagary repos (e.g. `metabase-cn`, `tldv_downloader`, `torn-smart-scripts`) don't load 64 lines of Vagary brand registry they have no use for.

**Applies to repos**: `vagary-platform`, `vagary-voice`, `anjaan-app`, `aakhara`, `bellring-server`, `bellring-extension`, `bulk`, `pulseboard`, `pulseboard-desktop`, `project-template`, `portfolio` (cross-link only), `host_page`, `vps_host`, `vps-ansible`, `platform-docs`, `vagary-earnings`.

---

## 41. Brand architecture (originally §41 of universal-claudemd.md)

Vagary Life Pvt Ltd is the **corporate parent**. Below it, product and tech activity is organized into named divisions. As of 2026-04-19, one division is formalized: **Vagary Labs** (tech/R&D/platform).

### Structure

```
Vagary Life Pvt Ltd (parent company; registered entity)
└── Vagary Labs (tech/R&D/platform division — vagarylabs.com [PENDING])
    ├── Platform
    │   └── vagary-platform (20-vertical substrate; repo renamed from `index-of-news` 2026-04-19)
    │       └── Index of News (flagship vertical; keeps its own news sub-brand + 6 domains)
    ├── Product brands (each lives as an independent product under its own domain)
    │   ├── Vagary Voice (vagaryvoice.cloud) — commercial voice-AI SaaS
    │   ├── Anjaan (anjaan.online) — Hinglish consumer chat
    │   ├── Bellring (.io/.app/.ai TBD) — whitelabel sale-celebration SaaS; repos `bellring-server` + `bellring-extension` (renamed from `sales-notification-*` 2026-04-19; formerly codenamed Salvo)
    │   ├── Aakhara (aakhara.com pending) — voice sales-training roleplay for BDEs (Sanskrit "आखाड़ा" = practice arena; repo renamed from `training-bot` 2026-04-19). Positioning TBD: could sit as Vagary Voice sub-product or stand alone
    │   └── Hype / Mockline / Kohort (legacy proposed names, superseded by Bellring/Aakhara above)
    └── OSS Utilities
        ├── bulk (renamed from `bulk_api_trigger` 2026-04-19)
        ├── tldv_downloader
        ├── pulseboard (renamed from `NetworkMonitorCN` 2026-04-19; Android OSS, `pulseboard.build` pending)
        └── project-template
```

Additional divisions (media, ops, consulting, etc.) may be added later. Keep Vagary Labs scoped to **tech/platform/R&D**.

### Domain strategy

- **vagarylife.com / vagarylife.in** — corporate parent marketing + investor/careers. TO BE BUILT.
- **vagarylabs.com** — tech/R&D division site. Domain **PENDING PURCHASE** (user flagged). Will host platform docs + OSS index + R&D blog once acquired.
- **Per-product domains** — each commercial product keeps its own brand domain (`vagaryvoice.cloud`, `anjaan.online`, future `hype.sh`, etc.). Product domains do NOT nest under `vagarylabs.com`.
- **chinmayramraika.in** — founder's personal hub; cross-links each Vagary Life / Vagary Labs product in a "projects" section.

### Repo-to-brand mapping (authoritative)

| Repo | Vagary Labs home | Product / sub-brand |
|---|---|---|
| `vagary-platform` | Platform | Holds all 20 verticals; flagship vertical = **Index of News** (news sub-brand, 6 domains) |
| `vagary-voice` | Product brands | **Vagary Voice** (commercial product, `vagaryvoice.cloud`) |
| `anjaan-app` | Product brands | **Anjaan** (consumer product, `anjaan.online`) |
| `aakhara` | Product brands | **Aakhara** (voice sales-training roleplay; `aakhara.com` pending). Renamed from `training-bot` 2026-04-19. Positioning TBD (standalone OR Vagary Voice sub-product) |
| `bellring-server` | Product brands | **Bellring** server (whitelabel sale-celebration SaaS backend; `.io/.app/.ai` TBD). Renamed from `sales-notification-backend` 2026-04-19 (formerly codenamed Salvo) |
| `bellring-extension` | Product brands | **Bellring** extension (Chrome MV3 + Firefox/Edge portable; pairs with `bellring-server`). Renamed from `sales-notification-extension` 2026-04-19 |
| `bulk`, `tldv_downloader`, `pulseboard`, `project-template` | OSS Utilities | Each with its own GitHub + README brand. `pulseboard` renamed from `NetworkMonitorCN` 2026-04-19 (Android OSS; `pulseboard.build` pending) |
| `portfolio` | Personal hub (OUTSIDE Vagary Labs) | `chinmayramraika.in` founder site |
| `host_page`, `platform-docs`, `vps_host`, `n8n-workflows`, `metabase-cn` | Infrastructure (internal to Vagary Labs) | No external product brand |
| `Automated-sales-manager-main` | Client work (CN-internal) | ASM — CN-branded; Cadre whitelabel TBD |
| `google-sheet-sales-manager` | Client work (CN-internal) | Sheetpilot whitelabel TBD |
| `Expense tracker` | Absorbing → Platform (`budget` vertical) | No standalone brand going forward |

### How Claude uses this

- When a repo's description says "product," check the brand table above for positioning.
- The **platform repo** (`vagary-platform`) is *not* a product. It is substrate. Individual verticals (news, budget, …) are the products that ship.
- Don't reinvent brand positioning in per-repo CLAUDE.md — reference this section and defer details to `~/.claude/specs/2026-04-19-brand-rename-proposal.md` (for rationale) + `~/.claude/conventions/repo-inventory.md` (for current state).
- For any new repo: declare its division home in its CLAUDE.md § Status / Brand section and cross-reference here.

### Caveats

- `vagarylabs.com` is **not yet purchased** (2026-04-19). Until acquired, Vagary Labs is an internal organizational concept; do not publish external references to `vagarylabs.com` until DNS is live.
- Additional divisions (media, ops, consulting) may emerge. When they do, add a sibling subtree here + bump VERSION.

### Playbook: bellring-cluster.md (verbatim from `~/.claude/conventions/playbooks/bellring-cluster.md`)

# Bellring Cluster Playbook

**VERSION: 2026-05-07-v1-stub**
Loaded only in Bellring cluster repos (per `~/.claude/conventions/repo-inventory.md` §45). Per-repo `CLAUDE.md` `@-imports` this file when applicable.

Source: NEW playbook authored 2026-05-07 (ENTRY #168) per operator decision C2. v1-STUB — rules accumulate as cross-repo concerns surface. No content forced; placeholder slots ready for first concrete rule discovery.

**Applies to repos**: `bellring-server`, `bellring-extension`.

---

## 1. Cluster identity

Bellring is a whitelabel sale-celebration SaaS:
- **`bellring-server`** — Node + Express + SendGrid + ws backend. Pairs with extension via WebSocket. Tier model: $19 / $79 / $299. Domain `.io/.app/.ai` TBD.
- **`bellring-extension`** — Chrome MV3 (vanilla JS; portable to Firefox/Edge MV3). Renders real-time celebratory popups (trophy animations, bell chime, multi-monitor placement) when a rep closes a sale.

Both repos formerly codenamed "Salvo" — renamed `sales-notification-backend` → `bellring-server` and `sales-notification-extension` → `bellring-extension` on 2026-04-19.

## 2. Cross-repo conventions (rules accumulate here)

### 2.1 Contract (server ↔ extension)

(STUB — codify the WebSocket message shape + auth-token format when first stabilized. Pact testing pattern referenced in §51.4 multi-lang playbook applies here when both halves are in scope.)

### 2.2 Auth token shape

(STUB — when token rotation cadence is decided, codify here.)

### 2.3 Sponsor / Chrome Web Store / Firefox Add-ons / Edge Add-ons cadence

Both repos are public-OSS sponsor-ready. Reference `commercial-bound.md` playbook for sponsor-readiness rules. Bellring-specific cadences (e.g., Chrome Web Store review windows, Firefox AMO submission) accumulate here when first encountered.

## 3. Notes

- v1-STUB intentionally minimal. As cross-repo rules surface in actual sessions, add them here rather than duplicating across the two per-repo `CLAUDE.md` files.
- This playbook is loaded by per-repo `CLAUDE.md` `@-import` when the repo cluster includes Bellring (per repo-inventory §45).

<!-- END PLAYBOOKS BLOCK -->

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
