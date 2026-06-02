# VPS handoff â€” scoped WhatsApp customer quoting via `/api/v1/ask`

**Audience:** teams-bridge / Phoenix AI (VPS) owners  
**From:** ailab WhatsApp gateway (`feat/whatsapp-gateway-agent-loop`)  
**Date:** 2026-06-02  

## Where to read this (pick one)

| Copy | URL / path |
|------|------------|
| **Public gist (works today)** | https://gist.github.com/PhoenixAssitantBot/4f0c96401a6b52446cb096791ad1339f |
| **spec-os (after org merge)** | https://github.com/Phoenix-Calibration/spec-os/blob/main/docs/phoenix-whatsapp/VPS_SCOPED_CUSTOMER_QUOTE_HANDOFF.md |
| **VPS server (recommended)** | `/root/.claude-agents/teams-bridge/docs/VPS_SCOPED_CUSTOMER_QUOTE_HANDOFF.md` â€” sync via script below |
| **ailab (private)** | `apps/phoenix-bot/gateway/whatsapp-gateway/docs/VPS_SCOPED_CUSTOMER_QUOTE_HANDOFF.md` |

### Sync onto the VPS (one-time or after updates)

```bash
mkdir -p /root/.claude-agents/teams-bridge/docs

# Handoff doc (public gist â€” no GitHub org access required)
curl -fsSL "https://gist.githubusercontent.com/PhoenixAssitantBot/4f0c96401a6b52446cb096791ad1339f/raw/VPS_SCOPED_CUSTOMER_QUOTE_HANDOFF.md" \
  -o /root/.claude-agents/teams-bridge/docs/VPS_SCOPED_CUSTOMER_QUOTE_HANDOFF.md

# RBAC SQL (phoenix-quote tools for customer bridge user)
curl -fsSL "https://gist.githubusercontent.com/PhoenixAssitantBot/568c3a6b83cd29bdecfaed7486790dc9/raw/rbac-whatsapp-quote-tools.sql" \
  -o /tmp/rbac-whatsapp-quote-tools.sql
# Edit :customer_user_email in the file, then run against MCP permissions Supabase.
```

**Related (ailab private):** `VPS_BRIDGE_GATEWAY_HANDOFF.md`, `WA_GATEWAY_AGENT_PROMPT.md`, `infra/rbac-whatsapp-quote-tools.sql`

---

## Summary

WhatsApp **scoped pilot** customers (today: **CDR00131 / B.Braun**) can ask for **pricing / quotations** on WhatsApp. The gateway no longer returns a static *â€œI canâ€™t put together a price quoteâ€¦â€* message for those users.

Instead, the gateway calls **your** bridge:

`POST /api/v1/ask` with `X-Phoenix-Scope` + quote-specific **metadata**, and expects **multi-turn** quoting via **phoenix-quote MCP** under the **customer** delegation â€” same capability as Teams Phoenix AI, not gateway `/mcp/chat`.

**Production (gateway):** revision `phoenix-whatsapp-gateway-00299-sr2` (image `e77f107-20260602-010218`), 100% traffic.

---

## What the gateway does now (quote turns)

| Step | Owner | Behavior |
|------|--------|----------|
| 1 | Gateway | Turn-context rewrite (optional): short replies expanded using `recent_history` |
| 2 | Gateway | Channel LLM router â†’ `handler: "quote"` |
| 3 | Gateway | **`deliverScopedQuoteBrainTurn()`** â€” skips agent loop for quote on scoped pilot |
| 4 | Gateway | `POST /api/v1/ask` with scope JWT + metadata below |
| 5 | Gateway | Persists `bridge_session_id` from response for follow-ups |
| 6 | Gateway | QC + WhatsApp format + send |

**Explicitly NOT used for scoped quotes:**

- Gateway `quote-flow.mjs` â†’ `phoenix_start` / **`/mcp/chat`** (operator principal â€” **must stay denied** for scoped sessions).
- Static `scoped_quote_failclosed` copy (only if `WHATSAPP_SCOPED_BRAIN_QUOTE_ENABLED=false` or brain fallback off).

---

## Gateway env (production)

| Variable | Prod value | Notes |
|----------|------------|--------|
| `PHOENIX_SCOPED_BRAIN_FALLBACK_ENABLED` | `true` | Required for any scoped `/api/v1/ask` |
| `PHOENIX_SCOPE_SIGNING_SECRET` | Secret Manager `phoenix-gateway-scope-signing-key` | Must match bridge + analyst |
| `WHATSAPP_SCOPED_BRAIN_QUOTE_ENABLED` | `true` | Routes `handler=quote` to brain |
| `PHOENIX_BRIDGE_ASK_URL` | `https://phoenix-ai-bot.ngrok.app/api/v1/ask` (or prod host) | Full ask URL |
| `PHOENIX_SCOPE_TOKEN_AUDIENCES` | `iris-ai;phoenix-bridge` | Semicolon in gcloud deploy |
| `PHOENIX_SCOPE_PILOT_CODES` | `CDR00131` | Only this code is scoped today |

Token contract: **3-part HS256 JWT**, sign `header.payload` bytes, TTL â‰¤ 300s (see ailab `VPS_BRIDGE_GATEWAY_HANDOFF.md`).

---

## `/api/v1/ask` â€” quote request shape

### Headers

- `Authorization: Bearer <PHOENIX_GATEWAY_TOKEN>`
- `X-Phoenix-Gateway-Secret` (if configured)
- **`X-Phoenix-Scope: <JWT>`** â€” customer role, `customer_ids: ["CDR00131"]`, `identity_verified: true`
- `Idempotency-Key` â€” Twilio message sid (or QC retry suffix)

### Body (representative)

```json
{
  "channel": "whatsapp",
  "customer": {
    "canonical_name": "B.Braun D.R. Inc.",
    "customer_code": "CDR00131",
    "wa_id": "+1..."
  },
  "message": "<planner message â€” may be rewritten from short user text>",
  "locale": "en",
  "conversation": [
    { "role": "user", "content": "..." },
    { "role": "assistant", "content": "..." }
  ],
  "metadata": {
    "session_id": "<bridge session id when continuing a quote thread>",
    "intent": "customer_quote",
    "quote_mode": true,
    "channel": "whatsapp",
    "twilio_message_sid": "...",
    "handoff_guidance": "Do not deflect pricing to an account manager when phoenix-quote tools can handle this turn...",
    "quote_guidance": "<QUOTE_CUSTOMER_GUIDANCE â€” full text in gateway channel-guidance.mjs>",
    "customer_quote_guidance": "Use phoenix-quote MCP (lookup_item, create_odoo_quote_pdf, add_items_to_quote) for THIS customer only...",
    "channel_context": "...",
    "service_advisor_guidance": "...",
    "whatsapp_format_guidance": "..."
  }
}
```

**Source of truth for metadata strings (ailab):** `apps/phoenix-bot/gateway/whatsapp-gateway/src/scoped-quote-brain.mjs` â†’ `buildCustomerQuoteBrainMetadata()`.

---

## What we need from VPS (acceptance criteria)

### 1. Honor `intent: "customer_quote"`

When `metadata.intent === "customer_quote"` (and valid scope token):

- Run the **normal Phoenix AI agent** with **phoenix-quote** tools on the **customer** MCP delegation profile.
- **Do not** default to â€œcontact your account manager for a quoteâ€ when tools work.
- **Do not** invent prices â€” only amounts from quote MCP tools.

### 2. Multi-turn session continuity

- Return **`session_id`** on every quote turn.
- Gateway sends it back as `metadata.session_id` on follow-ups.

### 3. Scope enforcement on writes

- Quote tools must bind to **`customer_ids` from the JWT** (CDR00131 only).
- Cross-tenant quote creation must **403 / deny**.

### 4. RBAC â€” grant quote tools on customer bridge user

Apply MCP permissions SQL (in private ailab):

`apps/phoenix-bot/gateway/whatsapp-gateway/infra/rbac-whatsapp-quote-tools.sql`

Tools: `create_odoo_quote_pdf`, `lookup_item`, `add_items_to_quote` on `phoenix-quote`.

### 5. Conversation + pronoun resolution

| User says | You should |
|-----------|------------|
| â€œCan you provide a quote for these instruments?â€ | Resolve instruments from `conversation`, start quote discovery |
| â€œQuote a Fluke 87â€ | `lookup_item` + ask site, cal vs cal+cert, qty, turnaround |

### 6. Publish discipline

Draft in chat first; publish Odoo quote/PDF only after explicit customer confirmation.

---

## Reply checklist (paste back when done)

```
VPS scoped WhatsApp quote â€” ready for pilot:

- [ ] phoenix-quote tools granted on customer bridge MCP user (SQL applied)
- [ ] /api/v1/ask honors metadata.intent=customer_quote + X-Phoenix-Scope
- [ ] session_id returned and respected on follow-up turns
- [ ] No default "contact your account manager for a quote" when tools work
- [ ] Cross-tenant quote write denied (tested)
- [ ] Live test: CDR00131 "quote for Fluke 87" returns discovery/pricing path
- [ ] Teams/operator quoting unchanged
```
