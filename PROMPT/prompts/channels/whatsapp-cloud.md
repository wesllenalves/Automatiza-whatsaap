---
id: whatsapp-cloud
loadWhen: cfg.channels.whatsapp_cloud?.enabled
context7:
  - /websites/developers_facebook_com_docs_whatsapp
tokensEstimate: 2000
verifiedAt: null   # 📝 TODO: validar via Context7 antes da primeira build
---

# WhatsApp Cloud API (Meta)

## 📋 Quando carregar
Usuário marcou `WhatsApp Cloud API` na Onda 1.

## 🎯 O que garante
- Graph API v21.0+ endpoints
- Handshake `hub.verify_token` (GET) + eventos (POST)
- HMAC via `X-Hub-Signature-256` (HMAC-SHA256 com `META_APP_SECRET`)
- 24h window de conversa + fallback pra templates aprovados
- Subscription idempotente em `/subscribed_apps`

## 📚 Endpoints canônicos (a verificar Context7 no primeiro uso)

**Base**: `https://graph.facebook.com/v21.0`
**Auth**: `Authorization: Bearer $META_SYSTEM_USER_TOKEN`

| Ação | Método | Endpoint | Notas |
|---|---|---|---|
| Info do número | GET | `/{phone-number-id}` | valida credenciais |
| Listar subs do phone | GET | `/{phone-number-id}/subscribed_apps` | |
| Subscrever app | POST | `/{phone-number-id}/subscribed_apps` | idempotente |
| Unsubscribe | DELETE | `/{phone-number-id}/subscribed_apps` | |
| Enviar msg | POST | `/{phone-number-id}/messages` | body: `{messaging_product:"whatsapp", to, type, ...}` |
| Enviar template | POST | `/{phone-number-id}/messages` | fora da 24h window |
| Mark as read | POST | `/{phone-number-id}/messages` | `{status:"read", message_id}` |

### Webhook (lado do app)
- `GET /api/webhooks/whatsapp-cloud?hub.mode=subscribe&hub.verify_token=X&hub.challenge=Y` → responder `Y` (plain text) se `X == META_WEBHOOK_VERIFY_TOKEN`
- `POST /api/webhooks/whatsapp-cloud` → body é o event payload, validar `X-Hub-Signature-256`

## ⚠️ Business-rule failures previstos

| Sintoma | Causa | Prevenção |
|---|---|---|
| Webhook nunca verifica no Meta dashboard | app não responde challenge em plain text | `return new Response(challenge, { status: 200 })`, sem JSON |
| Respostas fora da 24h window rejeitadas | janela de sessão expirou | usar templates aprovados; detectar via error code `131047` |
| `X-Hub-Signature-256` inválido | HMAC key errada ou body parse modificou bytes | usar **raw body** (não body já parseado) pro HMAC |
| Rate limit 80 msgs/s | sem throttle | limitar outbound por `phone_number_id` |
| Msg duplicada | Meta reenvia se app não responde 200 | dedupe por `messages[0].id` |

## 🔒 Contrato (stub — preencher em detalhe no primeiro build)

```ts
// apps/web/src/lib/channels/whatsapp-cloud.ts
export const cloudApiAdapter: ChannelAdapter = {
  id: "whatsapp_cloud",
  async ensureSession() { /* GET subscribed_apps → se faltar, POST */ },
  async getSessionStatus() { /* GET /{phone}?fields=verified_name */ },
  async parseWebhook(req) { /* verify X-Hub-Signature-256, normalizar */ },
  async verifyWebhook(req) { /* hub.challenge handshake */ },
  async sendText(contactId, text) { /* POST /messages */ },
};
```

## ✅ Checks
1. `curl $APP/api/webhooks/whatsapp-cloud?hub.challenge=42` (com verify_token correto) → retorna `42`
2. `GET /{phone}/subscribed_apps` lista o app
3. HMAC verification roda em raw body (não em `await req.json()` já parseado)

## 🚫 Armadilhas
- **NUNCA** usar `req.body` parseado pro HMAC — usar `req.text()` cru
- **NUNCA** responder o challenge com JSON — é plain text
- **NUNCA** ignorar error code `131047` (fora da 24h) — fallback pra template
- **SEMPRE** dedupe por `messages[0].id`

## 📖 Fonte
- 📝 **TODO**: resolver `/websites/developers_facebook_com_docs_whatsapp` ou equivalente via Context7 e verificar cada endpoint na primeira build
- Docs oficiais: https://developers.facebook.com/docs/whatsapp/cloud-api
