---
id: idempotent-sessions
loadWhen: always  # baseline — toda integração externa herda isto
tokensEstimate: 1200
verifiedAt: 2026-04-21
---

# Idempotent Sessions — padrão cross-channel

## 📋 Quando carregar
**Sempre**. Este é o padrão universal para lidar com recursos externos (sessões, instâncias, subscriptions) de qualquer canal.

## 🎯 O que este sub-prompt garante
- Contrato universal de `ensureSession()` aplicável a Waha, Evolution, Cloud API, Instagram
- Dicionário genérico de 422/409 → classificação benigna vs erro real
- Padrão **GET → decide → action → re-GET** (reconciliação)
- Regra UI: estado vem **sempre** do GET, nunca do POST

## 🧩 O padrão em 1 diagrama

```
            ┌─────────────────┐
            │  ensureSession  │
            └────────┬────────┘
                     │
              GET /recurso
                     │
         ┌───────────┼───────────┐
       404         200 OK       5xx
         │           │            │
         POST       PUT        FAIL
       create      update       up
         │           │
         ↓           ↓
        race?   config mudou?
         │           │
         ├─────→ 422/409? → classify (dicionário)
         │                  │
         │           ┌──────┴──────┐
         │       "ok/put"        "error"
         │           │              │
         ↓           ↓              ↓
   CONDITIONAL: só se state ∈ {STOPPED, FAILED, NOT_CREATED}
                  → POST /start
                          │
                          └→ re-GET (fonte da verdade)
                                  │
                                  ↓
                              return state
```

## 🔒 Interface universal

Todo `ChannelAdapter` implementa:

```ts
export type SessionState =
  | "NOT_CREATED" | "STOPPED" | "STARTING" | "SCAN_QR" | "WORKING" | "FAILED";

export interface ChannelAdapter {
  // Idempotente: chama N vezes sem side-effect adicional
  ensureSession(cfg: unknown): Promise<SessionState>;

  // Fonte da verdade — sempre GET do provedor
  getSessionStatus(): Promise<SessionState>;

  // Opcionais — nem todo canal tem
  startSession?(): Promise<void>;
  stopSession?(): Promise<void>;
  getQrCode?(): Promise<string | null>;
}
```

## 📖 Dicionário de erros — regex por canal

Cada canal define um `classify422` próprio, mas o **formato da saída** é universal:

```ts
type ErrorVerdict =
  | "ok"       // estado desejado já atingido, no-op
  | "put"      // recurso existe, fazer PUT em vez de POST
  | "error";   // bug real, propagar

async function classifyError(res: Response, provider: "waha" | "evolution" | "meta"): Promise<ErrorVerdict>;
```

### Padrões genéricos que todo canal deve cobrir

| Padrão na response | Verdict sugerido |
|---|---|
| `/already (started\|running)/i` | `ok` — no-op |
| `/not started/i` em `stop/logout` | `ok` — idempotente |
| `/already exists\|duplicate\|conflict/i` | `put` |
| `/unauthorized\|forbidden/i` | `error` (credenciais) |
| `/rate limit\|too many requests/i` | `error` (mas retry com backoff) |

## 🏛️ Regras universais de reconciliação

### Regra 1 — GET é fonte da verdade
```ts
// ❌ ERRADO
const r = await fetch(`${base}/start`, { method: "POST" });
return r.ok ? "WORKING" : "FAILED";

// ✅ CERTO
await fetch(`${base}/start`, { method: "POST" });  // fire-and-forget classificado
return await getSessionStatus();  // re-GET decide o state
```

### Regra 2 — UI nunca infere `FAILED` de erro de POST
Um 422 benigno (ex: "already started") **não** é erro. UI só mostra `FAILED` se `getSessionStatus()` explicitamente retornar `FAILED`.

### Regra 3 — start condicional
```ts
// Nunca chamar start cego
const state = await getSessionStatus();
const startable: SessionState[] = ["STOPPED", "FAILED", "NOT_CREATED"];
if (startable.includes(state)) await startSession();
// state == "STARTING" | "SCAN_QR" | "WORKING" → no-op, pollar
```

### Regra 4 — estruturar log
Todo call de estado emite:
```json
{
  "event": "session.ensure",
  "channel": "waha",
  "sessionName": "default",
  "stateBefore": "STOPPED",
  "stateAfter": "SCAN_QR",
  "action": "created|updated|started|ignored",
  "verdict": "ok|put|error",
  "durationMs": 342
}
```

### Regra 5 — booting (instrumentation)
No `instrumentation.ts` do Next:
```ts
for (const adapter of getEnabledAdapters()) {
  try { await adapter.ensureSession({}); }
  catch (err) { logger.error({ event: "channel.ensure_failed", channel: adapter.id, err }); }
  // NUNCA throw aqui — app precisa subir mesmo com 1 canal offline
}
```

### Regra 6 — webhook handler NÃO chama ensureSession
Request path crítico. Assume `WORKING`. Se payload vem malformed → responder 200 + logar + dropar.

## ⚠️ Anti-padrões (código a rejeitar em review)

```ts
// ❌ Todos abaixo são bugs que já ocorreram em produção

// 1. POST sem GET anterior → 422 "already exists"
await fetch(`${base}/sessions`, { method: "POST", body: ... });

// 2. Start sem verificar state → 422 "already started"
await fetch(`${base}/sessions/default/start`, { method: "POST" });

// 3. UI lê state do POST response
const res = await fetch(`${base}/start`, { method: "POST" });
setState(res.ok ? "WORKING" : "FAILED");   // MENTIRA: 422 pode ser benigno

// 4. ensureSession propaga qualquer non-2xx como erro
if (!res.ok) throw new Error();   // ignora o dicionário

// 5. Não re-GET após operação
await fetch(`${base}/start`, { method: "POST" });
return "WORKING";   // suposição sem validação
```

## ✅ Checks obrigatórios antes de marcar canal como "pronto"

1. 3 chamadas sequenciais a `POST /api/sessions/{ch}/ensure` retornam 200
2. Nenhuma resposta contém `"error": "already ..."` vazado
3. Logs incluem `verdict` em toda mudança de estado
4. UI não transiciona pra `FAILED` após erros benignos (testar com Chrome DevTools: mock 422 → UI deve ignorar)

## 📖 Referências cruzadas
- `channels/waha.md` — dicionário 422 específico do Waha
- `channels/evolution.md` — tratamento de 403/409 em `/instance/create`
- `channels/whatsapp-cloud.md` — POST em `subscribed_apps` é idempotente mas validar GET
- `patterns/hmac-webhook.md` — validação obrigatória antes de processar qualquer inbound
