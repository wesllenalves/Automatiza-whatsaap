---
id: instagram
loadWhen: cfg.channels.instagram_direct?.enabled
context7:
  - /websites/developers_facebook_com_docs_messenger-platform_instagram
tokensEstimate: 1800
verifiedAt: null   # 📝 TODO validar Context7
---

# Instagram Direct (Meta Graph API)

## 📋 Quando carregar
Usuário marcou `Instagram Direct (Meta Graph API)` na Onda 1.

## 🎯 O que garante
- Subscription de Page (não de phone number — diferente do Cloud API)
- Permissões obrigatórias: `instagram_manage_messages` (requer app review Meta)
- Webhook handshake idêntico ao Cloud API (`hub.verify_token`, `X-Hub-Signature-256`)
- IG Business Account conectada a Facebook Page (requisito Meta)

## 📚 Endpoints canônicos

**Base**: `https://graph.facebook.com/v21.0`
**Auth**: `Authorization: Bearer $META_IG_PAGE_ACCESS_TOKEN` (token da Page, não do System User)

| Ação | Método | Endpoint | Notas |
|---|---|---|---|
| Info da Page | GET | `/{page-id}` | valida token |
| IG Business conectada | GET | `/{page-id}?fields=instagram_business_account` | obtém `ig_business_account.id` |
| Listar subs da Page | GET | `/{page-id}/subscribed_apps` | |
| Subscrever | POST | `/{page-id}/subscribed_apps?subscribed_fields=messages,messaging_postbacks` | |
| Enviar DM | POST | `/{page-id}/messages` | `{recipient:{id:IG_USER_ID}, message:{text:"..."}}` |

## ⚠️ Business-rule failures

| Sintoma | Causa | Prevenção |
|---|---|---|
| Subscription retorna `(#200) Permissions error` | permissão `instagram_manage_messages` não aprovada | completar app review Meta ANTES do deploy em prod; em dev, usar Tester roles |
| Webhook não chega | Page não está conectada a IG Business Account | validar via `GET /{page}?fields=instagram_business_account` retorna objeto não-null |
| 24h window igual ao WhatsApp | sim, mesmo padrão | mesma lógica; fora da janela, só responde se user iniciou |
| Msg pra grupo/post comments não chega em DM | scope errado | `subscribed_fields=messages` só cobre DM |

## 🔒 Contrato (stub)

```ts
export const instagramAdapter: ChannelAdapter = {
  id: "instagram",
  async ensureSession() {
    // 1. GET /{page}?fields=instagram_business_account → valida conexão
    // 2. GET /{page}/subscribed_apps → se app não tá lá, POST
  },
  // resto parecido com cloud api
};
```

## 🚫 Armadilhas
- **NUNCA** confundir `page-id` com `ig_business_account.id` — subscription é na Page
- **NUNCA** assumir permissões em app não-aprovado Meta — em dev usar Tester role

## 📖 Fonte
- 📝 **TODO** validar via Context7 em primeira build
- https://developers.facebook.com/docs/messenger-platform/instagram
