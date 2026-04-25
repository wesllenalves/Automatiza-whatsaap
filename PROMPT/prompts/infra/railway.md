---
id: infra-railway
loadWhen: always  # baseline — todo deploy usa Railway
context7:
  - /railwayapp/cli
  - /railwayapp/docs
tokensEstimate: 1800
verifiedAt: 2026-04-21
---

# Railway — CLI, plugins, private networking

## 📋 Quando carregar
**Sempre**. Todo deploy no wizard passa por Railway.

## 🎯 O que este sub-prompt garante
- CLI correto (verbs exatos — `variable` singular, `--database` etc.)
- Plugins nativos (Postgres, Redis) e como referenciar em outros serviços
- Private networking IPv6 (pegadinha comum)
- Volumes persistentes
- Idempotência dos comandos (`add`, `service`, `link`)

## 📚 CLI verificado (Context7 `/railwayapp/cli`)

### Criar/linkar projeto
```bash
railway login                         # TTY ou export RAILWAY_TOKEN
railway init --name "my-project"      # criar
railway link                          # linkar repo a projeto existente
railway status                        # inspecionar projeto linkado
railway whoami                        # valida auth
```

### Serviços
```bash
# Criar a partir de repo ou imagem
railway add --service my-backend --repo user/repo
railway add --service web-server --image nginx:latest
railway add --service empty-svc --variables "PORT=3000"

# Selecionar serviço (persiste em .railway/config.json do repo)
railway service my-backend

# Linkar pasta local a um serviço (subpath em monorepo)
( cd apps/web && railway link --service app )
```

### Plugins (databases)
```bash
# Adicionar plugin nativo
railway add --database postgres
railway add --database redis
railway add --database postgres --database redis   # múltiplos em 1 call

# JSON output p/ parse
railway add --database postgres --json
# → {"id": "...", "name": "Postgres"}
```

### Variáveis (⚠️ singular `variable`, não `variables`)
```bash
# List
railway variable list                  # tabela
railway variable list --kv             # formato .env
railway variable list --json

# Set
railway variable set API_KEY=secret
railway variable set A=1 B=2 C=3                    # múltiplas em 1 call
railway variable set KEY=val --service backend      # cross-service
echo "secret-value" | railway variable set TOKEN --stdin   # multiline/sensível

# Skip redeploy ao setar (útil em bulk)
railway variable set K=V --skip-deploys

# Delete
railway variable delete API_KEY
```

### Referenciar plugin em outro serviço
No `.env` do serviço `app`, referenciar como variável:
```
DATABASE_URL=${{Postgres.DATABASE_URL}}
REDIS_URL=${{Redis.REDIS_URL}}
```
Railway resolve `${{<Service>.<Var>}}` **em runtime** — não precisa hardcodar nem copiar.

### Deploy
```bash
railway up                           # sobe current dir, stream logs
railway up --ci                      # CI mode: stream logs only, exit on done
railway up --detach                  # detach (equivalente antigo de `-d`)
railway up ./apps/web                # path específico
railway up --service app             # forçar serviço
railway up --message "fix: 422"      # msg do deploy
railway up --json                    # → {"deploymentId":"..."}
```

### Run (executar comando com envs injetadas)
```bash
railway run npm start                         # local, com envs do serviço linkado
railway run --service app npm start
railway run --service app npx prisma migrate deploy   # rodar migrations
railway run --environment staging npm test
```

### Domínios
```bash
railway domain                       # gera *.up.railway.app p/ serviço linkado
railway domain --service app         # cross-service
railway domain --json
```

### Volumes
```bash
railway volume add --service waha --mount-path /app/.sessions
railway volume list --service waha
```

### Logs
```bash
railway logs                         # stream do serviço linkado
railway logs --service app
railway logs --deployment <id>
```

## 🌐 Private Networking — IPv6

**Regra crítica**: private network na Railway é **IPv6**. Apps que bind só IPv4 (`127.0.0.1`) não ficam acessíveis.

Hostname interno: `<service-slug>.railway.internal` (resolve A6).

Porta interna = porta do container (ex: Next.js `3000`, Waha `3000`, Evolution `8080`). **Não** precisa expor publicamente — só o serviço `app` roda `railway domain`.

### Bind correto
```ts
// Next.js
// next.config.ts: output: "standalone"
// Dockerfile: ENV PORT=3000 HOSTNAME=0.0.0.0  (0.0.0.0 = dual stack)

// Ou explicit dual-stack:
server.listen(3000, "::");  // IPv6 any, aceita IPv4 mapped
```

## ⚠️ Business-rule failures

| Sintoma | Causa raíz | Prevenção |
|---|---|---|
| `railway variables --set K=V` comando não encontrado | plural, deprecated | usar `railway variable set K=V` (singular) |
| `connect ECONNREFUSED 127.0.0.1` em private call | app bind só IPv4 | `HOSTNAME=0.0.0.0` ou `::` |
| `DATABASE_URL is undefined` em runtime | env var referenciada antes do plugin existir | plugin Postgres deve ser criado ANTES de linkar a var no app |
| Deploy falha com "No start command" | missing `CMD` no Dockerfile ou `start` no package.json | garantir Dockerfile `CMD ["node", "server.js"]` |
| Volume perde dados após `railway down` | volume ≠ serviço (sobrevive) | ok; só `railway delete` zera |
| Plugin criado 2× | comando `add` repetido | `add` **não** é idempotente; check via `railway service Postgres` antes |
| Redeploy sobrescreve variáveis setadas manualmente | `--skip-deploys` esquecido em bulk set | usar `railway variable set ... --skip-deploys` em script, `up` explícito depois |
| `railway run` não vê env vars | serviço não linkado | `railway service <name>` primeiro |

## 🔒 Idempotência de provisioning

Sempre usar pattern "check-then-create":

```bash
# Projeto
railway status >/dev/null 2>&1 || railway init --name "$NAME"

# Plugin (Postgres)
railway variable list --service Postgres >/dev/null 2>&1 || railway add --database postgres

# Plugin (Redis) — condicional
if [[ "$DEBOUNCE_ON" == "true" ]]; then
  railway variable list --service Redis >/dev/null 2>&1 || railway add --database redis
fi

# Serviço
railway service app 2>/dev/null || railway add --service app

# Volume
railway volume list --service waha 2>/dev/null | grep -q waha-sessions \
  || railway volume add --service waha --mount-path /app/.sessions

# Domínio
railway domain --service app 2>/dev/null || true
```

## ✅ Checks antes de dizer "deploy ok"

1. `railway whoami` retorna usuário/org válido
2. `railway status` mostra os serviços esperados (app, Postgres, [Redis], [waha], [evolution])
3. `railway variable list --service app --kv | grep DATABASE_URL` mostra a referência
4. `curl https://<app>.up.railway.app/api/health` retorna `{"ok":true}`
5. Private call funciona: `railway run --service app -- curl http://waha.railway.internal:3000/api/server/version`
6. Logs recentes (`railway logs --service app`) não mostram crash loop

## 🚫 Armadilhas

- **NUNCA** `127.0.0.1` — Railway é IPv6 internamente
- **NUNCA** expor container de provider (Waha/Evolution) com `railway domain` — só `app`
- **NUNCA** passar secret via CLI arg (aparece em `ps`) — usar `--stdin`
- **NUNCA** `railway variables --set` (plural, antigo) — usar `railway variable set`
- **NUNCA** `railway up -d` assumindo que é detach — usar `--detach` explícito (shorthand pode variar)
- **SEMPRE** escrever `.env` locais em `.railway/` (gitignored) e carregar em bulk via `while read`
- **SEMPRE** usar `${{Service.VAR}}` p/ cross-service refs — nunca hardcodar connection strings

## 📖 Fonte
- `/railwayapp/cli` — verificado 2026-04-21
- `/railwayapp/docs` — guides + private networking
- CLI install: `npm i -g @railway/cli` ou `brew install railway`
