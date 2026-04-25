---
id: infra-postgres
loadWhen: cfg.infra.db === "postgres_railway"
context7:
  - /prisma/docs
tokensEstimate: 900
verifiedAt: null   # 📝 TODO
---

# Postgres (Prisma + Railway plugin)

## 📋 Quando carregar
`infra.db === "postgres_railway"` (default).

## 🎯 Essenciais
- Plugin Railway expõe `DATABASE_URL`
- Prisma como ORM padrão
- pgvector via `CREATE EXTENSION` (se `tools.rag`)
- Migrations: `prisma migrate deploy` no boot/CI

## 🔧 Setup

```bash
# Deploy time (uma vez):
railway run --service app -- npx prisma generate
railway run --service app -- npx prisma migrate deploy

# Se usar pgvector:
railway run --service app -- psql $DATABASE_URL -c 'CREATE EXTENSION IF NOT EXISTS vector;'
# Rodar ANTES do migrate deploy se o schema tiver Unsupported("vector")
```

### `apps/web/src/lib/db.ts`
```ts
import { PrismaClient } from "@prisma/client";
declare global { var __prisma: PrismaClient | undefined; }
export const prisma = global.__prisma ?? new PrismaClient({
  log: process.env.NODE_ENV === "production" ? ["error"] : ["query", "error", "warn"],
});
if (process.env.NODE_ENV !== "production") global.__prisma = prisma;
```

## ⚠️ Failures

| Sintoma | Causa | Prevenção |
|---|---|---|
| Connection pool exhausted | sem singleton ou pool baixo | `globalThis.__prisma` + `connection_limit` na DATABASE_URL |
| Migration fails com "vector does not exist" | pgvector não instalado | `CREATE EXTENSION` antes do migrate |
| N+1 em queries frequentes | sem `include` | auditar com logs `query` em dev |

## 🚫 Armadilhas
- **NUNCA** criar `new PrismaClient()` por request — memory leak
- **SEMPRE** rodar migrations em `railway run`, não local contra prod
- **SEMPRE** setar `connection_limit` razoável (ex: `?connection_limit=10`) em prod
