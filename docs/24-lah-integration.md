# Integrating OpenWA with LAH (separate repos)

LAH (`C:\Users\macha\Desktop\LAH`) is a Next.js app. OpenWA is this WhatsApp API gateway. Keep them in **separate repos** and connect over HTTP.

## Architecture (recommended)

```
┌─────────────────────┐         HTTPS + X-API-Key         ┌──────────────────────┐
│  LAH (Vercel)       │  ───────────────────────────────▶ │  OpenWA (VPS/Docker) │
│  Next.js + Supabase │                                   │  NestJS + WA session │
│  sendWhatsApp()     │  ◀─────────────────────────────── │  webhooks (optional) │
└─────────────────────┘         message.received, etc.    └──────────────────────┘
```

| Piece | Where it runs | Why |
|-------|---------------|-----|
| **LAH** | Vercel | Serverless Next.js fits Vercel |
| **OpenWA** | Docker on a VPS / Railway / Fly.io / Render | Long-lived process, Chromium/Baileys, session files, websockets |

### Why OpenWA cannot run on Vercel

OpenWA is a **persistent** NestJS service that:

- Keeps WhatsApp sessions alive (QR login, reconnect)
- Needs disk (or volume) for session data
- Often runs Chromium (whatsapp-web.js) or a Baileys socket
- Expects webhooks and health checks on a stable host

Vercel serverless functions are short-lived and have no persistent browser/session store. Deploy OpenWA with Docker; deploy LAH on Vercel and point env vars at OpenWA’s public URL.

## 1. Run OpenWA locally

```bash
cd "C:\Users\macha\Desktop\Open WA"
copy .env.minimal .env   # or .env.example
npm install
npm run dev
```

Or Docker:

```bash
docker compose -f docker-compose.dev.yml up -d
```

- Dashboard: http://localhost:2785 (Docker) or http://localhost:2886 (npm run dev)
- API: http://localhost:2785/api
- Swagger: http://localhost:2785/api/docs

Create a session in the dashboard, start it, scan the QR with WhatsApp. Create an **API key** and note the **session id**.

## 2. Production OpenWA (not Vercel)

Pick any host that supports Docker + a persistent volume:

```bash
# On the server
git clone https://github.com/rmyndharis/OpenWA.git
cd OpenWA
# Set DOMAIN, BASE_URL, CORS_ORIGINS, strong secrets in .env
docker compose up -d
```

Put TLS in front (Caddy / nginx / Cloudflare Tunnel). Example public URL:

`https://wa.yourdomain.com`

Set in OpenWA `.env`:

```env
BASE_URL=https://wa.yourdomain.com
CORS_ORIGINS=https://your-lah.vercel.app,http://localhost:3000
AUTO_START_SESSIONS=true
```

Optional managed hosts with Docker: Railway, Fly.io, Render, DigitalOcean App Platform, a cheap VPS (Hetzner, Contabo, etc.).

## 3. Wire LAH (other repo)

In `C:\Users\macha\Desktop\LAH\.env.local` (and Vercel project env):

```env
WHATSAPP_PROVIDER=openwa
OPENWA_BASE_URL=https://wa.yourdomain.com
OPENWA_API_KEY=owa_k1_your_key
OPENWA_SESSION_ID=your-session-uuid-or-name
WHATSAPP_VA_PHONE=2547xxxxxxxx
```

Leave Meta vars empty when using OpenWA. LAH’s `sendWhatsApp()` posts to:

`POST {OPENWA_BASE_URL}/api/sessions/{OPENWA_SESSION_ID}/messages/send-text`

with body `{ "chatId": "2547xxxxxxxx@c.us", "text": "..." }` and header `X-API-Key`.

Local cross-repo test:

1. OpenWA on `:2785`
2. LAH on `:3000` with `OPENWA_BASE_URL=http://localhost:2785`
3. Trigger a payment/fulfillment or appointment path that calls `sendWhatsApp`

## 4. Optional: inbound webhooks (OpenWA → LAH)

In OpenWA dashboard (or API), create a webhook:

```json
{
  "url": "https://your-lah.vercel.app/api/webhooks/openwa",
  "events": ["message.received", "session.status"],
  "secret": "shared-hmac-secret"
}
```

Add a matching route in LAH to verify the HMAC and handle events (e.g. log inbound client messages for the VA).

## 5. Sending checklist

1. OpenWA session status = **connected**
2. LAH env points at the public OpenWA URL (not `localhost` when LAH is on Vercel)
3. Chat IDs use WhatsApp format: `2547xxxxxxxx@c.us` (digits + `@c.us`)
4. API key has permission to send on that session
5. For local LAH → remote OpenWA, use the remote HTTPS URL; for both local, use `http://localhost:2785`

## SDK alternative

Instead of raw `fetch`, LAH can install `@rmyndharis/openwa` (see `sdk/javascript`). The env-based `fetch` client in LAH avoids an extra dependency and keeps Meta as a fallback provider.
