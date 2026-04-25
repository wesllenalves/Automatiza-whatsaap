---
id: infra-redis
loadWhen: cfg.cache?.debounce?.enabled
context7:
  - /redis/ioredis
tokensEstimate: 800
verifiedAt: null   # 📝 TODO
---

# Redis (via Railway plugin) — cliente ioredis

## 📋 Quando carregar
Debounce habilitado (ou qualquer outra feature que use Redis).

## 🎯 Essenciais
- Client singleton via `ioredis`
- Lê `REDIS_URL` injetado pelo plugin Railway via `${{Redis.REDIS_URL}}`
- Comandos críticos: `MULTI`, `SET PX`, `RPUSH`+`LTRIM`+`PEXPIRE`, `SCAN` (nunca `KEYS`)

## 🔒 Client singleton

```ts
// apps/web/src/lib/redis.ts
import Redis from "ioredis";

declare global { var __redis: Redis | undefined; }

export const redis = global.__redis ?? new Redis(process.env.REDIS_URL!, {
  maxRetriesPerRequest: 3,
  enableReadyCheck: true,
  reconnectOnError(err) {
    // reconnect on transient errors
    return err.message.includes("READONLY");
  },
});

if (process.env.NODE_ENV !== "production") global.__redis = redis;
```

## ⚠️ Failures

| Sintoma | Causa | Prevenção |
|---|---|---|
| `Connection is closed` no Next dev | hot-reload cria N conexões | singleton pattern via `globalThis` |
| Latência alta súbita | Redis em "saving" (RDB snapshot) | usar AOF em prod (plugin Railway — config) |
| `KEYS *` trava o servidor | full scan bloqueante | `SCAN` cursor iterativo |
| Transação parcialmente aplicada | `EXEC` sem MULTI | sempre pair `multi()` ... `exec()` |

## 🚫 Armadilhas
- **NUNCA** `redis.keys("*")` em prod — usar `scan()` com cursor
- **NUNCA** criar cliente por request — singleton via globalThis
- **SEMPRE** `MULTI` para grupos atômicos (RPUSH + LTRIM + PEXPIRE)
- **SEMPRE** timeout em comandos bloqueantes (`BLPOP` etc.)
