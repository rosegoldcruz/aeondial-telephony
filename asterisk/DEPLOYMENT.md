# AEON Dial – Progressive Dialer Deployment Checklist

## Prerequisites

- [ ] Asterisk 20+ or 21+ installed on the telephony server
- [ ] AEON backend running (Node 20+, Redis 7+, Supabase project active)
- [ ] Supabase migration `004_progressive_dialer.sql` applied
- [ ] `.env` file on the backend server populated (see Variables section)

---

## Environment Variables (backend `.env`)

```ini
# Supabase
SUPABASE_URL=https://<project-ref>.supabase.co
SUPABASE_SERVICE_ROLE_KEY=<service-role-key>

# Redis / BullMQ
REDIS_URL=redis://127.0.0.1:6379

# JWT
JWT_SECRET=<strong-random-secret>

# ARI – must match ari.conf [aeondial] section
ARI_URL=http://<asterisk-host>:8088/ari
ARI_USERNAME=aeondial
ARI_PASSWORD=<strong-password>
ARI_APP=aeondial
ARI_ENDPOINT_PREFIX=PJSIP

# Dialer tunables
DIALER_CPS=5                         # calls per second per worker instance
DIALER_VOICEMAIL_FILE=vm-drop-demo   # Asterisk sound file for voicemail drop (no extension)
BACKEND_INTERNAL_URL=http://localhost:4000  # used by dialer engine to set DIALER_BACKEND_URL

# CORS
CRM_ORIGIN=https://<your-crm-domain>
AI_WORKER_ORIGIN=http://localhost:8787
```

---

## Asterisk Configuration

### 1. HTTP / ARI server  (`/etc/asterisk/http.conf`)

Copy `asterisk/http.conf.template` → `/etc/asterisk/http.conf`.  
Ensure port 8088 (or 8089 for TLS) is reachable from the backend server only (firewall).

### 2. ARI credentials  (`/etc/asterisk/ari.conf`)

Copy `asterisk/ari.conf.template` → `/etc/asterisk/ari.conf`.  
Replace `${ARI_PASSWORD}` with the same value used in `.env`.  
Replace `${CRM_ORIGIN}` with your actual CRM domain.

### 3. PJSIP trunks and agents  (`/etc/asterisk/pjsip.conf`)

Copy `asterisk/pjsip.conf.template` → `/etc/asterisk/pjsip.conf`.  
Fill in:
- `${CARRIER_SIP_HOST}`, `${CARRIER_USERNAME}`, `${CARRIER_PASSWORD}` for your SIP carrier
- One `[AGENT_EXT]` block per agent softphone or WebRTC client
- For WebRTC agents add `webrtc=yes` and a WSS transport section

### 4. Dialplan  (`/etc/asterisk/extensions.conf`)

Copy `asterisk/extensions.conf.template` → `/etc/asterisk/extensions.conf`.  
Update `BACKEND_URL` in the `[globals]` section to point to your backend:

```
BACKEND_URL=http://10.0.0.5:4000
```

Tune AMD globals as needed for your carrier's audio profile:

| Variable | Default | Notes |
|---|---|---|
| `AMD_INITIAL_SILENCE` | 2500 ms | Max silence before first word |
| `AMD_GREETING` | 3000 ms | Max duration of a HUMAN greeting |
| `AMD_AFTER_GREETING_SILENCE` | 800 ms | Silence after greeting → HUMAN |
| `AMD_MAX_WORD_LENGTH` | 5000 ms | Word > this → MACHINE |
| `AMD_MAX_WORDS` | 3 | Words > this → HUMAN |

### 5. Reload Asterisk

```bash
asterisk -rx "module reload"
asterisk -rx "ari show apps"          # should show 'aeondial'
asterisk -rx "pjsip show endpoints"   # verify trunks and agents
```

---

## Database Migration

```bash
# Using Supabase CLI
supabase db push

# Or directly via psql
psql "$DATABASE_URL" -f supabase/migrations/004_progressive_dialer.sql
```

---

## Backend Start / Test

```bash
# Start backend
npm run dev

# Test ARI originate
TEST_ORG_ID=org_xxx TEST_AGENT_ID=user_xxx TEST_CAMPAIGN_ID=camp_xxx \
  TEST_CONTACT_ID=contact_xxx TEST_TO_NUMBER=+15005550006 \
  npm run test:ari
```

---

## First Dialer Run (curl examples)

### Create agent session (go ready)

```bash
curl -X POST http://localhost:4000/dialer/agents/session \
  -H "Content-Type: application/json" \
  -H "x-org-id: $ORG_ID" \
  -H "x-user-id: $AGENT_ID" \
  -H "x-role: agent" \
  -d '{"agent_id":"'"$AGENT_ID"'","campaign_id":"'"$CAMPAIGN_ID"'"}'
```

### Add leads to campaign queue

```bash
curl -X POST http://localhost:4000/dialer/campaigns/$CAMPAIGN_ID/leads \
  -H "Content-Type: application/json" \
  -H "x-org-id: $ORG_ID" \
  -H "x-user-id: $AGENT_ID" \
  -H "x-role: admin" \
  -d '{
    "leads": [
      {"lead_id":"lead_001","contact_id":"contact_001","phone":"+15005550001","priority":10},
      {"lead_id":"lead_002","contact_id":"contact_002","phone":"+15005550002","priority":5}
    ]
  }'
```

### Start campaign dialer

```bash
curl -X POST http://localhost:4000/dialer/campaigns/$CAMPAIGN_ID/start \
  -H "x-org-id: $ORG_ID" \
  -H "x-user-id: $ADMIN_ID" \
  -H "x-role: admin"
```

### Check dialer status

```bash
curl http://localhost:4000/dialer/campaigns/$CAMPAIGN_ID/status \
  -H "x-org-id: $ORG_ID" \
  -H "x-user-id: $ADMIN_ID" \
  -H "x-role: admin"
```

### Stop campaign dialer

```bash
curl -X POST http://localhost:4000/dialer/campaigns/$CAMPAIGN_ID/stop \
  -H "x-org-id: $ORG_ID" \
  -H "x-user-id: $ADMIN_ID" \
  -H "x-role: admin"
```

---

## Architecture summary

```
CRM Frontend (Next.js)
  └── /dialer page  ── WebSocket → backend /ws
                   └── REST → backend /dialer/*

AEON Backend (Fastify)
  ├── /dialer/agents/*     Agent FSM (session + state transitions)
  ├── /dialer/campaigns/*  Campaign start/stop/status/leads
  ├── /dialer/calls/*      AMD result webhook + dispositions
  ├── /telephony/calls/*   Low-level call operations (originate/bridge/end)
  └── BullMQ Workers
       └── dialer:<org>:<campaign>  Progressive dial loop

Asterisk (ARI)
  ├── ARI REST  ←→  aeondial-backend/src/core/ari.ts
  ├── AMD()          dialplan detects machine/human
  └── CURL()         posts AMD result to /dialer/calls/:id/amd_result
```

---

## Security notes

- ARI endpoint must **not** be internet-facing. Bind to internal / VPN interface.
- Use TLS (`tlsenable=yes` in `http.conf`) in production with a valid certificate.
- `ARI_PASSWORD` should be a random 32-char string; store in a secrets manager.
- All backend routes require `x-org-id` / `x-user-id` headers verified by `requireTenantContext`.
- Caller-ID is resolved per-campaign from the `phone_numbers` table (org-scoped).
- DNC (`dial_state=dnc`) leads are never re-queued by the engine.
