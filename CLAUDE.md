# Sales Notification Backend

## Claude Preamble
<!-- VERSION: 2026-04-19-v8 -->
<!-- SYNC-SOURCE: ~/.claude/conventions/universal-claudemd.md -->

**Universal laws** (§4), **MCP routing** (§6), **Drift protocol** (§11), **Dynamic maintenance** (§14), **Capability resolution** (§15), **Subagent SKILL POLICY** (§16), **Session continuity** (§17), **Decision queue** (§17.a), **Attestation** (§18), **Cite format** (§19), **Three-way disagreement** (§20), **Pre-conditions** (§21), **Provenance markers** (§22), **Redaction rules** (§23), **Token budget** (§24), **Tool-failure fallback** (§25), **Prompt-injection rule** (§26), **Append-only discipline** (§27), **BLOCKED_BY markers** (§28), **Stop-loss ladder** (§29), **Business-invariant checks** (§30), **Plugin rent rubric** (§31), **Context ceilings** (§32).

**Sources**: `~/.claude/conventions/universal-claudemd.md` (laws, MCP routing, lifecycle, rent rubric) + `~/.claude/conventions/project-hygiene.md` (doc placement, cleanup, local-workspaces). Read relevant sections before significant work. Re-audit due **2026-07-19**. Sync: `~/.claude/scripts/sync-preambles.py`.

## Product Overview

| Product | Real-Time Sales Celebration System (Backend) |
|---------|----------------------------------------------|
| **What it does** | Receives sale events from LeadSquared via webhook, authenticates ~300 BDEs via OTP, and broadcasts real-time notifications to Chrome extension clients over WebSocket. Powers the celebratory "trophy popup" that appears when any BDE closes a sale. |
| **Who uses it** | BDEs (~300 at Coding Ninjas) receive notifications passively. Sales managers send announcements and private messages. LeadSquared CRM triggers sale events automatically. IT team maintains the backend on Render. |
| **Status** | Production (deployed on Render: `sales-notification-backend.onrender.com`). |
| **Organization** | SMPL562 |

## Product Features and User Journeys

### 1. Sale Broadcast (Core Value Prop)
- **User journey**: A BDE closes a sale in LeadSquared. LeadSquared fires a webhook to `/webhook` with the BDE name, product, and manager. Backend broadcasts to all connected Chrome extension clients. Every BDE sees a celebratory trophy popup on their screen.
- **Success signals**: Webhook received, validated (Bearer token), and broadcast to all connected WebSocket clients within seconds. Trophy popup appears on all BDE screens.
- **Failure signals**: Webhook rejected (auth failure, missing `type` field). No WebSocket clients connected (extension not installed/authenticated). Render instance sleeping (cold start delay on free tier).

### 2. OTP Authentication (User Onboarding)
- **User journey**: BDE installs the Chrome extension, enters their `@codingninjas.com` email, receives a 6-digit OTP via SendGrid, verifies it, and receives a UUID token stored in extension storage.
- **Success signals**: OTP delivered within 30 seconds. Token issued and stored. WebSocket connection established immediately after auth.
- **Failure signals**: SendGrid delivery failure (OTP never arrives). OTP expires before entry. Email domain validation rejects valid email. OTPs lost on server restart (in-memory storage).

### 3. General Announcements (Manager Communication)
- **User journey**: Manager (or automation) sends a POST to `/webhook` with `type: "notification"` and a message. All connected BDEs see the announcement.
- **Success signals**: Message delivered to all online BDEs. Announcement renders clearly in the extension popup.
- **Failure signals**: Message not delivered to offline BDEs (no queuing/persistence).

### 4. Private Messages (Targeted Communication)
- **User journey**: Manager sends a POST to `/webhook` with `type: "private"`, a target email, and a message. Only the matching BDE sees it.
- **Success signals**: Only the targeted BDE receives the message. Other BDEs see nothing.
- **Failure signals**: Target BDE is offline and never receives the message. Email matching fails.

### 5. Connection Health (Reliability)
- **User journey**: Extension sends ping every 30 seconds. Backend responds with pong. `/ping` endpoint keeps Render instance awake.
- **Success signals**: Stable WebSocket connections. Render instance stays warm. Reconnection succeeds after brief disconnects.
- **Failure signals**: Render instance sleeps despite pings. WebSocket connections drop and don't recover. Memory exceeds 256MB limit.

## Known Product Limitations
- OTPs stored in-memory -- lost on server restart
- No message persistence -- offline BDEs miss all notifications
- Render free tier has cold-start delays
- Rate limiting is IP-based -- may affect BDEs on shared corporate network
- Max 5 WebSocket connections per IP

---

## Technical Reference

### Stack
- Node.js, Express, WebSocket (ws), JWT, SendGrid, dotenv, express-rate-limit, CORS

### File Organization
- Never save working files to root folder
- Single-file server: `server.js` contains all application logic (Express routes, WebSocket server, memory management classes)
- `Procfile` configures Render deployment
- `tokens.json` is generated at runtime for persistent token storage

### Key Architecture
- `LRUCache`, `FileBackedStorage`, `ClientManager`, `MemoryManager` classes handle state
- WebSocket path: `/ws` with JWT token auth via query param
- API endpoints: `/request-otp`, `/verify-otp`, `/webhook`, `/ping`, `/stats`, `/health`
- CORS restricted to `chrome-extension://${EXTENSION_ID}`
- Memory limit: 256MB (`--max-old-space-size=256`)

### Build & Test
```bash
npm install          # Install dependencies
npm start            # Production: node --max-old-space-size=256 server.js
npm run dev          # Development: nodemon --max-old-space-size=256 server.js
```

### Environment Variables
Required (see `.env.example`):
- `PORT` - Server port (default: 3000)
- `NODE_ENV` - Environment (development/production)
- `EXTENSION_ID` - Chrome Extension ID for CORS/origin validation
- `WEBHOOK_TOKEN` - Bearer token for LeadSquared webhook auth
- `JWT_SECRET` - Secret for signing JWT tokens
- `SENDGRID_API_KEY` - SendGrid API key for OTP emails

### n8n Workflow Automation

This project can trigger and receive n8n workflows at `https://n8n.chinmayramraika.in`.

- **Webhook URL:** Set in `N8N_WEBHOOK_URL` env var
- **API Key:** Set in `N8N_API_KEY` env var (unique per project)
- **Auth Header:** `X-API-Key: <N8N_API_KEY>`
- **Workflow repo:** github.com/Cramraika/n8n-workflows (private)

### Security Rules
- NEVER hardcode API keys, secrets, or credentials in any file
- NEVER pass credentials as inline env vars in Bash commands
- NEVER commit .env, .claude/settings.local.json, or .mcp.json to git
