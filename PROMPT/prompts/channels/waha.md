---
id: waha
loadWhen: cfg.channels.waha_core?.enabled || cfg.channels.waha_plus?.enabled
context7:
  - /devlikeapro/waha-docs
tokensEstimate: 2500
verifiedAt: 2026-04-21
---

# WAHA (WhatsApp HTTP API) — Core & Plus

## 📋 Quando carregar
Usuário marcou `WhatsApp · Waha Core` OU `WhatsApp · Waha Plus` na Onda 1 do wizard.

## 🎯 O que este sub-prompt garante
Após ler isto, Claude saberá:
- Endpoints e **estados válidos** de sessão Waha
- **Dicionário completo de erros 422** (vindos da prática, não só dos docs)
- Padrão `ensureSession()` idempotente com reconciliação via GET
- Diferenças Core (free, 1 sessão) vs Plus ($19/mo, multi + mídia)
- Config de webhook com HMAC + retry policy

## 📚 Endpoints canônicos

**Auth**: header `X-Api-Key: <chave>`. Chave pode ser em plaintext ou `sha512:<hash>`. Paths em `WHATSAPP_API_KEY_EXCLUDE_PATH` (`ping,health`) não precisam da chave.

### Sessões
| Ação | Método | Endpoint | Notas |
|---|---|---|---|
| Listar todas | GET | `/api/sessions` | array |
| Buscar 1 | GET | `/api/sessions/{name}` | **404 se não existe** |
| Criar | POST | `/api/sessions` | body: `{name, config}`. **422 se existir** |
| Atualizar config | PUT | `/api/sessions/{name}` | só em sessão existente |
| Iniciar | POST | `/api/sessions/{name}/start` | idempotente **nos docs**, mas versões retornam 422 se já iniciada — TRATAR |
| Parar | POST | `/api/sessions/{name}/stop` | idempotente (não desloga) |
| Logout | POST | `/api/sessions/{name}/logout` | desloga do WA, preserva config |
| Restart | POST | `/api/sessions/{name}/restart` | stop + start (ignora se STOPPED) |
| Deletar | DELETE | `/api/sessions/{name}` | remove config E volume |
| QR code | GET | `/api/{name}/auth/qr?format=image` | só se state=`SCAN_QR_CODE` |
| Server version | GET | `/api/server/version` | usar como health check — retorna `{engine: "GOWS", ...}` |

### Estados possíveis
`STOPPED` → `STARTING` → `SCAN_QR_CODE` → `WORKING` — ou → `FAILED`.

### Webhook config (inline na sessão)
```json
{
  "name": "default",
  "config": {
    "webhooks": [{
      "url": "https://<app>/api/webhooks/waha",
      "events": ["message", "message.any", "session.status"],
      "hmac": { "key": "<secret>" },
      "retries": { "policy": "linear", "delaySeconds": 2, "attempts": 4 },
      "customHeaders": [{ "name": "X-Source", "value": "<slug>" }]
    }]
  }
}
```

Alternativa via env (aplica GLOBAL a todas sessões):
```bash
WHATSAPP_HOOK_URL=https://<app>/api/webhooks/waha
WHATSAPP_HOOK_EVENTS=message,message.any,session.status
WHATSAPP_HOOK_HMAC_KEY=<secret>
```

## ⚠️ Business-rule failures conhecidos

| Sintoma (erro real observado) | Causa raíz | Prevenção obrigatória |
|---|---|---|
| `422 "Session 'default' already exists. Use PUT to update it."` | POST cego sem GET anterior | `GET /sessions/{n}` primeiro; 404 → POST; ok → PUT |
| `422 "Session 'default' is already started."` | `/start` sem checar state | consultar `getSessionStatus()`; só start se `∈ {STOPPED, FAILED, NOT_CREATED}` |
| `422 "Session 'default' is not started."` em `/stop` | stop em sessão STOPPED | 422 aqui é no-op, ignorar |
| UI mostra `FAILED` com sessão rodando | POST 422 foi propagado como erro p/ UI | UI state SEMPRE vem de `GET /sessions/{n}` (fonte da verdade), NUNCA do retorno do POST |
| QR desaparece no redeploy | `/app/.sessions` não montado como volume | Railway: `railway volume add --mount-path /app/.sessions` |
| Webhook fura HMAC após redeploy | chave do webhook diferente da env do app | PUT config da sessão com HMAC atual no boot (`instrumentation.ts`) |
| "Multiple sessions are not supported" no Core | tentou criar 2ª sessão no Core (free) | Core só aceita `default`. Multi → Plus |
| Webhook retorna 5xx, Waha dispara spam de retry | retry policy sem backoff | webhook handler deve responder 200 **mesmo em erro** (log + reenqueue) |
| Msg de mídia chega sem URL válida | Core não envia mídia | Core = texto only. Enviar mídia → upgrade Plus |
| `state=SCAN_QR_CODE` infinito | QR expirou | restart da sessão + polling QR a cada 2s |

## 🔒 Contrato obrigatório do adapter

`apps/web/src/lib/channels/waha.ts` **DEVE** exportar:

```ts
import type { ChannelAdapter, SessionState } from "./types";

// Classifica 422 do Waha — elimina falsos-positivos de erro
export async function classifyWaha422(res: Response): Promise<"ok" | "put" | "error"> {
  if (res.status !== 422) return res.ok ? "ok" : "error";
  const body = await res.clone().json().catch(() => ({}));
  const msg = String(body?.message ?? "");
  if (/already exists\. Use PUT to update/i.test(msg)) return "put";
  if (/is already (started|stopped)/i.test(msg)) return "ok";
  if (/is not started/i.test(msg)) return "ok";       // no-op em stop
  if (/status is (STARTING|SCAN_QR_CODE|WORKING)/i.test(msg)) return "ok";
  return "error";
}

export async function getWahaStatus(name: string): Promise<SessionState> {
  const r = await fetch(`${base}/api/sessions/${name}`, { headers });
  if (r.status === 404) return "NOT_CREATED";
  if (!r.ok) return "FAILED";
  const data = await r.json();
  const map: Record<string, SessionState> = {
    STOPPED: "STOPPED", STARTING: "STARTING",
    SCAN_QR_CODE: "SCAN_QR", WORKING: "WORKING", FAILED: "FAILED",
  };
  return map[data.status] ?? "FAILED";
}

export const wahaAdapter: ChannelAdapter = {
  id: "waha",
  async ensureSession(cfg) { /* GET → POST/PUT → conditional start → re-GET */ },
  async getSessionStatus() { /* delega p/ getWahaStatus(default) */ },
  async startSession() { /* POST /start com classifyWaha422 */ },
  async stopSession()  { /* POST /stop  com classifyWaha422 */ },
  async getQrCode()    { /* só se state==SCAN_QR */ },
  async parseWebhook(req) { /* HMAC verify, normaliza p/ ChannelMessage */ },
  async sendText(contactId, text) { /* POST /api/sendText */ },
};
```

## ✅ Checks que Claude DEVE validar antes de dizer "pronto"

1. `curl $APP/api/sessions/waha/ensure` chamado 3× consecutivas → todas 200, nenhuma propaga 422 benigno
2. `curl -H "X-Api-Key: $KEY" $WAHA/api/server/version` retorna `"engine":"GOWS"`
3. `/api/sessions` no Waha mostra `default` com webhook apontando pra `$APP/api/webhooks/waha`
4. `config.webhooks[0].hmac.key` não está vazio
5. Volume `/app/.sessions` montado no Railway (`railway volume list --service waha`)
6. Após qualquer POST de mudança, UI state vem do GET — nunca do POST response

## 🚫 Armadilhas típicas

- **NUNCA** `POST /api/sessions` sem GET primeiro
- **NUNCA** `POST /sessions/{n}/start` sem checar `getSessionStatus()` (mesmo que docs digam idempotente — versões reais podem divergir)
- **NUNCA** propagar 422 benigno (via dicionário) pra UI como erro
- **NUNCA** criar 2ª sessão no Core (só `default`; se precisar multi, mudar pra Plus)
- **NUNCA** expor o container Waha publicamente no Railway (`railway domain` NÃO — private network apenas)
- **NUNCA** hardcodar `127.0.0.1` — Railway private net é IPv6, usar `0.0.0.0`/`::` ou `waha.railway.internal`
- **SEMPRE** chamar `ensureSession()` no `instrumentation.ts` do Next após boot — refresca webhook URL que muda em redeploy
- **SEMPRE** guardar hash `sha512:` em `WAHA_API_KEY` do container, plaintext em `WAHA_API_KEY_PLAIN` do app (p/ header `X-Api-Key`)

## 📖 Fonte
- `/devlikeapro/waha-docs` — verificado via Context7 em 2026-04-21
- Core image: `devlikeapro/waha:latest` · Plus: `devlikeapro/waha-plus:latest`
- Porta default: `3000` · Dashboard em `/dashboard` (envs `WAHA_DASHBOARD_*`)
