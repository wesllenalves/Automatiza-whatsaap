---
id: hmac-webhook
loadWhen: always   # baseline — todo webhook precisa HMAC
tokensEstimate: 900
verifiedAt: 2026-04-21
---

# HMAC Webhook Verification

## 📋 Quando carregar
**Sempre**. Todo webhook handler deve validar HMAC antes de processar.

## 🎯 O que garante
- Waha: HMAC-SHA512 em header `X-Webhook-Hmac` com algoritmo `sha512:HEX`
- Meta (Cloud API + Instagram): HMAC-SHA256 em header `X-Hub-Signature-256: sha256=HEX`
- Evolution: não tem HMAC nativo — usar bearer custom em `headers.Authorization` do webhook config
- Validar em **raw body** (bytes exatos), nunca em objeto parseado

## 🔒 Implementação (genérica)

```ts
import { createHmac, timingSafeEqual } from "node:crypto";

export function verifyHmacSha256(rawBody: string, signatureHeader: string, secret: string): boolean {
  // ex: "sha256=a1b2c3..."
  const [algo, sigHex] = signatureHeader.split("=");
  if (algo !== "sha256") return false;
  const expected = createHmac("sha256", secret).update(rawBody).digest("hex");
  const a = Buffer.from(sigHex, "hex");
  const b = Buffer.from(expected, "hex");
  return a.length === b.length && timingSafeEqual(a, b);
}

export function verifyHmacSha512(rawBody: string, signatureHeader: string, secret: string): boolean {
  const expected = createHmac("sha512", secret).update(rawBody).digest("hex");
  const a = Buffer.from(signatureHeader.replace(/^sha512:/, ""), "hex");
  const b = Buffer.from(expected, "hex");
  return a.length === b.length && timingSafeEqual(a, b);
}
```

## 🧩 Uso em Next.js route handler

```ts
// apps/web/src/app/api/webhooks/waha/route.ts
export async function POST(req: Request) {
  const rawBody = await req.text();   // ⚠️ text(), NÃO json()
  const sig = req.headers.get("x-webhook-hmac") ?? "";
  if (!verifyHmacSha512(rawBody, sig, process.env.WAHA_HMAC_KEY!)) {
    return new Response("invalid signature", { status: 401 });
  }
  const payload = JSON.parse(rawBody);
  // ... processar
  return Response.json({ ok: true });
}
```

## ⚠️ Failures

| Sintoma | Causa | Prevenção |
|---|---|---|
| HMAC sempre inválido | usou `await req.json()` antes do verify | `req.text()` cru, parse depois |
| Verify OK local, fails em prod | diferença de bytes invisíveis (BOM, CRLF) | nunca transformar o body antes do verify |
| Timing attack | `===` em strings | usar `timingSafeEqual` |
| Chave compartilhada vaza em logs | log do header | nunca logar `x-webhook-hmac` |

## 🚫 Armadilhas
- **NUNCA** comparar HMAC com `===` — timing attack
- **NUNCA** usar body parseado — usar raw bytes
- **NUNCA** aceitar request sem HMAC (retornar 401)
- **SEMPRE** responder 200 rapidamente após verify — processamento pesado assíncrono (via debounce)
