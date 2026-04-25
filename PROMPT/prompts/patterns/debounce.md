---
id: debounce
loadWhen: cfg.cache?.debounce?.enabled
context7:
  - /redis/ioredis
tokensEstimate: 1100
verifiedAt: 2026-04-21
---

# Message Debounce — Redis sliding window

## 📋 Quando carregar
`cache.debounce.enabled == true` (default). Usuário escolheu coalescing de msgs.

## 🎯 O que garante
- Sliding window por `(channel, contactId)` — cada msg RESETA o timer
- Compare-and-delete token evita race entre 2 workers Railway
- Buffer cap (`maxBuf=20`) defesa contra spam
- TTL 4× `windowMs` de segurança no buffer
- Fallback imediato quando `DEBOUNCE_ENABLED=false`

## 🔒 Contrato (em `apps/web/src/lib/debounce.ts`)

```ts
import { redis } from "@/lib/redis";

export async function enqueueForDebounce(msg: ChannelMessage) {
  if (process.env.DEBOUNCE_ENABLED !== "true") {
    return flushToLLM([msg]);  // path síncrono
  }

  const key  = `${process.env.DEBOUNCE_KEY_PREFIX ?? "debounce"}:${msg.channel}:${msg.contactId}`;
  const lock = `${key}:lock`;
  const windowMs = Number(process.env.DEBOUNCE_WINDOW_MS ?? 5000);
  const maxBuf   = Number(process.env.DEBOUNCE_MAX_BUFFER ?? 20);

  await redis.multi()
    .rpush(key, JSON.stringify(msg))
    .ltrim(key, -maxBuf, -1)
    .pexpire(key, windowMs * 4)
    .exec();

  const token = crypto.randomUUID();
  await redis.set(lock, token, "PX", windowMs);

  setTimeout(async () => {
    const current = await redis.get(lock);
    if (current !== token) return;           // outra msg chegou → cancela
    await redis.del(lock);
    const buffered = await redis.lrange(key, 0, -1);
    await redis.del(key);
    if (buffered.length) await flushToLLM(buffered.map(s => JSON.parse(s)));
  }, windowMs + 50);
}
```

## ⚠️ Failures previstos

| Sintoma | Causa | Prevenção |
|---|---|---|
| Msg perdida se worker crasha antes do setTimeout disparar | timer in-memory | TTL 4× windowMs na chave — próximo boot pode varrer chaves expiradas via SCAN e re-enfileirar |
| Flush dupla em 2 workers | sem compare-and-delete token | sempre comparar token antes de `del` |
| Buffer cresce infinito | sem LTRIM | MULTI com LTRIM obrigatório |
| Resposta vai pra canal errado em grupo cross-channel | key não incluía `channel` | key = `{prefix}:{channel}:{contactId}` |

## 🚫 Armadilhas
- **NUNCA** `setTimeout` sem token (race entre workers)
- **NUNCA** esquecer `LTRIM` no MULTI (spam quebra o processo)
- **NUNCA** `windowMs > 15000` (perde janela Meta Cloud API 24h)
- **SEMPRE** TTL de segurança no buffer (4× windowMs)

## 📖 Fonte
- Ver §5.2 do main spec p/ explicação completa
- `/redis/ioredis` — validar API do MULTI
