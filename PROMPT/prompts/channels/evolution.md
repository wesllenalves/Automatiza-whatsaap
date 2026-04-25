---
id: evolution
loadWhen: cfg.channels.evolution_api?.enabled
context7:
  - /evolutionapi/evolution-api
tokensEstimate: 2200
verifiedAt: 2026-04-21
---

# Evolution API (Baileys)

## 📋 Quando carregar
Usuário marcou `WhatsApp · Evolution API` na Onda 1.

## 🎯 O que este sub-prompt garante
Após ler isto, Claude saberá:
- Endpoints corretos (incluindo **verbs que confundem**: restart é PUT, logout é DELETE)
- Como setar webhook via endpoint dedicado (`/webhook/set/{instance}`)
- Formato do event `MESSAGES_UPSERT` para parse do webhook
- Lista completa de events Baileys p/ assinar

## 📚 Endpoints canônicos

**Auth**: header `apikey: <AUTHENTICATION_API_KEY>` (global) OU `apikey: <hash>` (per-instance — retornado no create).
**Porta default**: `8080`. **Integration string**: `"WHATSAPP-BAILEYS"`.

### Instâncias
| Ação | Método | Endpoint | Notas |
|---|---|---|---|
| Listar (filtra) | GET | `/instance/fetchInstances?instanceName={name}` | vazio = todas; filter por `instanceName`, `instanceId`, `phoneNumber` |
| Criar | POST | `/instance/create` | retorna `hash` (api key per-instance) + `qrcode.base64` se `qrcode:true` |
| Conectar (QR) | **GET** | `/instance/connect/{instanceName}` | ⚠️ GET, não POST. Retorna QR ou `{state:"open"}` se já conectado |
| Estado | GET | `/instance/connectionState/{instanceName}` | `open` / `close` / `connecting` |
| Restart | **PUT** | `/instance/restart/{instanceName}` | ⚠️ PUT, não POST — reconecta sem deletar |
| Logout | **DELETE** | `/instance/logout/{instanceName}` | ⚠️ DELETE, não POST — desloga WA, mantém instância |
| Deletar | DELETE | `/instance/delete/{instanceName}` | remove tudo |

### Webhook
| Ação | Método | Endpoint | Notas |
|---|---|---|---|
| Configurar webhook | POST | `/webhook/set/{instanceName}` | body no shape abaixo |
| Buscar webhook | GET | `/webhook/find/{instanceName}` | |

### Mensagens (envio)
| Ação | Método | Endpoint | Notas |
|---|---|---|---|
| Texto | POST | `/message/sendText/{instanceName}` | `{number, text}` |
| Mídia | POST | `/message/sendMedia/{instanceName}` | `{number, mediatype, mimetype, caption, media}` (media = URL ou base64) |
| Áudio | POST | `/message/sendWhatsAppAudio/{instanceName}` | |

### Estados → mapeamento `SessionState`
- `open` → `WORKING`
- `connecting` → `STARTING` ou `SCAN_QR` (dependendo de ter QR em cache)
- `close` → `STOPPED` ou `NOT_CREATED`

## ⚠️ Business-rule failures conhecidos

| Sintoma | Causa raíz | Prevenção |
|---|---|---|
| 403/409 em `POST /instance/create` | instância já existe com esse nome | `GET /instance/fetchInstances?instanceName=X` primeiro; se existir, pular create |
| Restart retorna 404/405 | usou POST em vez de PUT | **usar `PUT /instance/restart/{n}`** |
| Logout retorna 404/405 | usou POST em vez de DELETE | **usar `DELETE /instance/logout/{n}`** |
| Webhook não chega após criar instância | webhook não foi setado no POST /create OU envs incorretas | chamar `POST /webhook/set/{n}` explicitamente após create — mais confiável |
| QR vem vazio no `/instance/connect` | instância já está `open` | verificar `GET /instance/connectionState/{n}` primeiro |
| Event do webhook vem em snake_case inesperado | `base64: true` muda shape do payload | fixar `base64: true` ou `false` e documentar no parseWebhook |
| "Call endpoint not found" | integration não foi `WHATSAPP-BAILEYS` | sempre enviar `"integration": "WHATSAPP-BAILEYS"` no body do create |
| Msgs duplicadas | mesmo event disparado 2× (Baileys internal retry) | dedupe por `data.key.id` do payload + `Message.providerMsgId` unique |
| Instância "fantasma" após crash | DB tem registro mas processo Baileys morreu | sempre reconciliar no boot: se estado DB=open mas connect retorna QR, resync |

## 🔒 Contrato obrigatório

`apps/web/src/lib/channels/evolution.ts`:

```ts
export async function ensureEvolutionInstance(cfg: EvoConfig): Promise<SessionState> {
  const name = cfg.instanceName ?? "default";
  const base = process.env.EVOLUTION_BASE_URL!;
  const headers = { "apikey": process.env.EVOLUTION_API_KEY!, "content-type": "application/json" };

  // 1. GET sempre primeiro
  const list = await (await fetch(`${base}/instance/fetchInstances?instanceName=${name}`, { headers })).json();
  const exists = Array.isArray(list) && list.length > 0;

  if (!exists) {
    const r = await fetch(`${base}/instance/create`, {
      method: "POST", headers,
      body: JSON.stringify({
        instanceName: name,
        integration: "WHATSAPP-BAILEYS",   // OBRIGATÓRIO
        qrcode: true,
        groupsIgnore: false,
        alwaysOnline: true,
        webhook: {
          enabled: true,
          url: `${process.env.NEXT_PUBLIC_APP_URL}/api/webhooks/evolution`,
          byEvents: false,
          base64: true,
          events: ["MESSAGES_UPSERT", "CONNECTION_UPDATE"],
        },
      }),
    });
    // create pode retornar 403/409 se existir em race
    if (!r.ok && ![403, 409].includes(r.status)) {
      throw new Error(`evolution.create failed: ${r.status} ${await r.text()}`);
    }
  }

  // 2. Reconfigurar webhook SEMPRE (URL muda em redeploy)
  await fetch(`${base}/webhook/set/${name}`, {
    method: "POST", headers,
    body: JSON.stringify({
      webhook: {
        enabled: true,
        url: `${process.env.NEXT_PUBLIC_APP_URL}/api/webhooks/evolution`,
        byEvents: false,
        base64: true,
        events: ["MESSAGES_UPSERT", "CONNECTION_UPDATE"],
      },
    }),
  });

  // 3. Se close, dispara QR via /connect
  const state = (await (await fetch(`${base}/instance/connectionState/${name}`, { headers })).json())
    ?.instance?.state;
  if (state === "close") {
    await fetch(`${base}/instance/connect/${name}`, { headers });  // GET!
  }

  // 4. Re-GET como fonte de verdade
  const finalState = (await (await fetch(`${base}/instance/connectionState/${name}`, { headers })).json())
    ?.instance?.state;
  const map: Record<string, SessionState> = {
    open: "WORKING", connecting: "SCAN_QR", close: "STOPPED",
  };
  return map[finalState] ?? "FAILED";
}
```

## ✅ Checks que Claude DEVE validar

1. `curl -H "apikey: $KEY" $BASE/instance/fetchInstances?instanceName=default | jq length` retorna ≥1
2. `GET /webhook/find/{name}` retorna URL apontando pra `/api/webhooks/evolution`
3. `base64` consistente entre create e webhook/set (mesmo valor)
4. Volume `/evolution/instances` montado no Railway (persiste Baileys session)
5. Events inclui pelo menos `MESSAGES_UPSERT` (core de inbound msg)

## 🚫 Armadilhas típicas

- **NUNCA** POST em `/instance/restart` ou `/instance/logout` — é PUT e DELETE respectivamente
- **NUNCA** esquecer `"integration": "WHATSAPP-BAILEYS"` no create (default pode variar por versão)
- **NUNCA** confiar só no webhook config passado no create — sempre chamar `/webhook/set/{n}` depois
- **NUNCA** usar a global API key pra operações per-instance em deploys multi-tenant — usar o `hash` retornado no create
- **SEMPRE** dedupe msgs por `data.key.id` (evento pode repetir)
- **SEMPRE** passar `base64` com valor EXPLÍCITO (`true` recomendado p/ mídia via webhook)

## 📖 Fonte
- `/evolutionapi/evolution-api` — verificado via Context7 em 2026-04-21
- Imagem: `atendai/evolution-api:latest`
- Porta default: `8080`
- Docs: https://doc.evolution-api.com/
