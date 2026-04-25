---
id: meta-graph-subscription
loadWhen: cfg.channels.whatsapp_cloud?.enabled || cfg.channels.instagram_direct?.enabled
tokensEstimate: 800
verifiedAt: null   # 📝 TODO
---

# Meta Graph API — Subscription pattern (compartilhado Cloud API + Instagram)

## 📋 Quando carregar
Cloud API OU Instagram habilitados — ambos usam o mesmo padrão.

## 🎯 O que garante
- Handshake `hub.verify_token` (GET challenge) idêntico
- POST `/subscribed_apps` idempotente
- `X-Hub-Signature-256` HMAC em todo POST de evento
- Error code semantics Meta (`131047` = fora da 24h window)

## 🤝 Handshake (GET)
```ts
// Rota única compartilhada: /api/webhooks/<channel>
export async function GET(req: Request) {
  const url = new URL(req.url);
  const mode = url.searchParams.get("hub.mode");
  const token = url.searchParams.get("hub.verify_token");
  const challenge = url.searchParams.get("hub.challenge");

  if (mode === "subscribe" && token === process.env.META_WEBHOOK_VERIFY_TOKEN) {
    return new Response(challenge, { status: 200 });  // plain text!
  }
  return new Response("forbidden", { status: 403 });
}
```

## 🔄 Subscription idempotente
```ts
// 1. GET /{node}/subscribed_apps — descobrir se app já tá subscrito
const subs = await fetch(`https://graph.facebook.com/v21.0/${node}/subscribed_apps`, {
  headers: { Authorization: `Bearer ${token}` },
}).then(r => r.json());

const subscribed = subs?.data?.some(s => s.id === process.env.META_APP_ID);
if (!subscribed) {
  // 2. POST subscribe (idempotente mas validamos antes pra economizar call)
  await fetch(`https://graph.facebook.com/v21.0/${node}/subscribed_apps${
    channel === "instagram" ? "?subscribed_fields=messages,messaging_postbacks" : ""
  }`, { method: "POST", headers: { Authorization: `Bearer ${token}` } });
}
```

## 🚨 Error codes Meta

| Code | Significado | Ação |
|---|---|---|
| `100` | Param inválido | ver logs, corrigir request |
| `131026` | Msg undeliverable | dropar |
| `131047` | Fora da 24h window | só responder com template aprovado |
| `131051` | Msg type não suportado | dropar ou converter |
| `200` | Permissions error | revisar permissões do app Meta |

## 🚫 Armadilhas
- **NUNCA** responder challenge com JSON — plain text string
- **NUNCA** processar POST sem validar `X-Hub-Signature-256` (ver `patterns/hmac-webhook.md`)
- **NUNCA** confiar que POST `subscribed_apps` é idempotente — sempre GET antes
