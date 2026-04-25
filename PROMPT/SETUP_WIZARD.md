# /sc:design — Setup Wizard: Multi-channel AI Agent (WhatsApp · Instagram · Next.js)

> **Plug-and-play prompt — orquestrador modular.** Suporta Waha Core/Plus, Evolution API, WhatsApp Cloud API, Instagram Direct. Sub-prompts em `prompts/**` carregados condicionalmente via §13.

## ⚡ INVOCAÇÃO

### Modo A — Bootstrap do zero (primeira vez)
Cole no Claude Code exatamente isto:
```
/sc:design @SETUP_WIZARD.md
```

### Modo B — Retomar uma sessão interrompida
```
/sc:design @SETUP_WIZARD.md --resume
```
Claude lê o arquivo de estado (`~/.claude-wizard/<project-slug>.state.json`) e retoma do último checkpoint.

### Modo C — Dry-run (ver o plano sem executar)
```
/sc:design @SETUP_WIZARD.md --dry-run
```
Executa ondas de perguntas + gera `SPEC.md` + `setup.config.json`, mas **não** scaffolda nem deploya.

---

## 📊 EXPECTATIVA (comunicar ao usuário ANTES de começar)

| Item | Valor |
|---|---|
| Tempo total estimado | 25–45 min (ativo) + 5–10 min de build Railway (+10 min se Evolution/Meta) |
| Custo mensal base | Railway Hobby $5 + Postgres free tier + LLM (pay-per-use, ~$1–10 no MVP) |
| Add-ons opcionais | Redis Railway (grátis no Hobby) · Waha Plus $19/mo · Cloud API (~$0.005–$0.09 por conversa) |
| Canais disponíveis | Waha Core · Waha Plus · Evolution API · WhatsApp Cloud API (Meta) · Instagram Direct (Meta) — **multi-select** |
| Debounce de mensagens | Redis coalesce msgs rápidas → 1 resposta (default 5s window, configurável) |
| Decisões do usuário | 4 ondas de 3–4 perguntas (AskUserQuestion) |
| Outputs | Repo git local + N serviços Railway (app + postgres + [redis, waha, evolution] conforme config) + URL pública + SPEC.md + config.json |

Claude deve imprimir esta tabela no início da sessão e pedir confirmação textual (`ok` ou `cancelar`) antes do §1.

---

O prompt conduz dois wizards: (1) o **Dev Wizard** (Claude Code faz AskUserQuestion + scaffold + deploy), e (2) o **Client Wizard** embutido no app (onboarding passo-a-passo para o usuário final configurar WhatsApp/IA/tools).

---

## 0. OBJETIVO

Construir e publicar, em uma única sessão, um app Next.js (shadcn/ui · Nova · Alien Green) que:

1. Conecta **um ou mais canais** de mensageria escolhidos pelo usuário (multi-select na Onda 1 do §2):
   - **WhatsApp · Waha Core** (GOWS free, 1 sessão `default`, texto-only) — container próprio
   - **WhatsApp · Waha Plus** ($19/mo, mídia + multi-sessão) — container próprio
   - **WhatsApp · Evolution API** (Baileys open-source, self-hosted) — container próprio
   - **WhatsApp Cloud API** (Meta oficial) — sem container, webhook externo
   - **Instagram Direct** (Meta Graph) — sem container, webhook externo
2. Recebe mensagens de todos os canais habilitados, **coalesce via debounce** (Redis sliding window por contato+canal), roteia para um **LLM configurável** (OpenAI / Anthropic / Google / Groq / OpenRouter) e devolve a resposta pelo **mesmo canal da msg original**.
3. Expõe um **painel web** para o cliente final configurar prompt do agente, tools (calendário, RAG, handoff humano, webhooks), ajustar janela de debounce, e monitorar conversas.
4. Sobe **serviços na Railway sob demanda** conforme config: sempre `app` (Next.js) + `Postgres`; condicionalmente `Redis` (se debounce ativo), `waha` (se Waha habilitado), `evolution` (se Evolution habilitado). Todos conectados via private networking IPv6.

### 🧭 Arquitetura modular de sub-prompts (novo)
Este prompt é o **orquestrador**. Especificações detalhadas de cada framework/API vivem em `prompts/**` e são **injetadas condicionalmente** no contexto de Claude baseado em `setup.config.json`. Ver §13 para algoritmo e `prompts/_INDEX.md` para matriz completa.

**Regra forte**: Claude **NÃO** carrega todos os sub-prompts — só os que a config exige. Isso mantém contexto enxuto e evita raciocínio degradado por excesso de detalhe irrelevante.

---

## 0.9 PRÉ-MITIGAÇÕES (gotchas reais de deploys passados — aplique ANTES de scaffoldar)

> 🧭 **Contexto**: cada item abaixo corresponde a um bug concreto que explodiu durante um deploy real da versão anterior deste wizard. Claude **DEVE** ler esta seção inteira antes de começar o §4 e aplicar cada contramedida proativamente.

### 0.9.1 Prisma 7 quebrou `url = env(...)` no schema — pinar em ^6.x
Prisma 7.x removeu `url` do bloco `datasource` em `schema.prisma` (migração obrigatória para `prisma.config.ts` ou driver-adapter). Isso quebra o wizard porque o `schema.prisma` aqui usa `url = env("DATABASE_URL")`.
**Contramedida**: no §4.0, instalar explicitamente `@prisma/client@^6` e `prisma@^6` (não `@latest`). Rodar `npx prisma generate` após `npm install` pra confirmar que compila.

### 0.9.2 Prisma CLI dentro do container não acha `effect` (trans-dep de @prisma/config)
Se o Dockerfile rodar `prisma migrate deploy` no CMD, o container **vai crashar**: a árvore de node_modules copiada seletivamente (`node_modules/@prisma`, `node_modules/prisma`) não inclui deps transitivas como `effect` que o `@prisma/config` exige em runtime.
**Contramedida**: **NÃO** rodar migrate dentro do container. Rodar migrate do shell do operador contra a `DATABASE_PUBLIC_URL` da Railway, antes de levantar o app:
```bash
DB_URL=$(railway variables --service Postgres-<id> --json | jq -r .DATABASE_PUBLIC_URL)
( cd apps/web && DATABASE_URL="$DB_URL" npx prisma migrate deploy )
( cd apps/web && DATABASE_URL="$DB_URL" npx tsx prisma/seed.ts )
```
Dockerfile `CMD` final: apenas `["node", "server.js"]`.

### 0.9.3 Migration SQL precisa existir antes do `migrate deploy`
Se só `000_init/migration.sql` com `CREATE EXTENSION vector;` estiver presente, o schema nunca é criado (migrate deploy roda mas não há DDL de tabelas). Gerar o SQL completo uma vez localmente:
```bash
npx prisma migrate diff --from-empty --to-schema-datamodel prisma/schema.prisma --script > prisma/migrations/000_init/migration.sql
```
E adicionar `prisma/migrations/migration_lock.toml` com `provider = "postgresql"` (sem ele, migrate deploy reclama).

### 0.9.4 Waha duplica eventos — `message.any` causa 3× webhook por msg
Subscrever em `["message", "message.any"]` faz o Waha entregar a **mesma mensagem** várias vezes (um callback por evento subscrito). Sob concorrência, `prisma.contactIdentity.upsert()` racea e 2 de 3 chamadas falham com `P2002` (unique violation) — mas pior: o `setTimeout` de debounce pode ser setado múltiplas vezes.
**Contramedida**:
1. Subscrever **apenas** `["message", "session.status"]` (nunca `message.any`).
2. Variável de ambiente `WHATSAPP_HOOK_EVENTS=message,session.status` no serviço Waha.
3. `parseWebhook` ignora eventos que não forem `message`.

### 0.9.5 Prisma `upsert` não é atômico sob concorrência
Apesar da dedup acima, ainda podem chegar dois webhooks simultâneos (retry da Waha, edge network). `upsert()` faz SELECT + INSERT em duas queries — duas conexões podem passar o SELECT juntas e colidir no INSERT.
**Contramedida obrigatória** no webhook handler:
```ts
async function ensureContactIdentity(channel, externalId, displayName) {
  try {
    return await prisma.contactIdentity.create({
      data: { channel, externalId, contact: { create: { displayName: displayName ?? externalId } } },
      include: { contact: true },
    });
  } catch (err) {
    if (err instanceof Prisma.PrismaClientKnownRequestError && err.code === "P2002") {
      return prisma.contactIdentity.findUnique({
        where: { channel_externalId: { channel, externalId } },
        include: { contact: true },
      });
    }
    throw err;
  }
}
```
E dedup por `providerMsgId` **antes** de qualquer DB write:
```ts
if (msg.providerMsgId) {
  const dup = await prisma.message.findUnique({ where: { providerMsgId: msg.providerMsgId } });
  if (dup) return Response.json({ ok: true, duplicate: true });
}
```

### 0.9.6 Waha GOWS escuta em `:8080`, não `:3000`
O log do container mostra `WhatsApp HTTP API is running on: http://[::1]:8080`. Setar `WAHA_BASE_URL=http://<waha-service>.railway.internal:8080` no serviço `app`. Usar `3000` resulta em `fetch failed`.

### 0.9.7 Next.js 16 renomeou `middleware.ts` → `proxy.ts`
- Arquivo: `src/proxy.ts`
- Export: `export function proxy(req: NextRequest)` (não `middleware`)
- `config.matcher` continua igual.
- `NextConfig` removeu as chaves `eslint` e `typescript` — remover do `next.config.ts`.
- `useSearchParams` em Client Component **precisa** `<Suspense>` no Server Component pai.

### 0.9.8 shadcn nova precisa `--font-sans` / `--font-mono` nos nomes exatos
O preset `base-nova` lê via `@theme inline { --font-sans: var(--font-sans); --font-mono: var(--font-mono); }`. Se `layout.tsx` usar `variable: "--font-geist-sans"`, a var `--font-sans` fica undefined e o browser cai em Times. Também garantir cadeia de fallback:
```tsx
const geistSans = Geist({ variable: "--font-sans", subsets: ["latin"] });
const geistMono = Geist_Mono({ variable: "--font-mono", subsets: ["latin"] });
```
E no `globals.css`:
```css
--font-sans: var(--font-sans), ui-sans-serif, system-ui, sans-serif;
--font-mono: var(--font-mono), ui-monospace, SFMono-Regular, monospace;
```

### 0.9.9 Tema dark por default (Alien Green precisa fundo escuro)
O accent lime `oklch(0.93 0.29 130)` fica ilegível sobre branco. Sempre emitir `<html className="dark ...">` no `layout.tsx` raiz, e a classe `dark` deve estar aplicada antes de qualquer componente renderizar.

### 0.9.10 Next.js standalone — `server.js` fica na raiz do bundle, não em `apps/web/`
Para projetos **single-package** (sem workspaces pnpm/npm), `apps/web/.next/standalone/` contém `server.js` no root, **não** em `apps/web/server.js`. Copiar corretamente no Dockerfile:
```dockerfile
COPY --from=builder /app/apps/web/.next/standalone ./
COPY --from=builder /app/apps/web/.next/static ./.next/static
COPY --from=builder /app/apps/web/public ./public
# CMD:
CMD ["node", "server.js"]
```
Para monorepo (workspaces), o `server.js` **fica** em `apps/web/server.js` dentro do standalone. Teste com `find .next/standalone -name server.js` para confirmar.

### 0.9.11 setup.config.json deve existir no filesystem em runtime
O `src/lib/config.ts` faz `fs.readFileSync("setup.config.json")`. O Dockerfile precisa `COPY --chown=nextjs:nodejs setup.config.json ./setup.config.json` na stage runner. Para dev local, criar symlink: `ln -sf ../../setup.config.json apps/web/setup.config.json`.

### 0.9.12 Redis client lazy — `new Redis()` no module-load quebra `next build`
`next build` faz "Collecting page data" instanciando módulos. Se `redis.ts` faz `new Redis(process.env.REDIS_URL!)` no top-level, build falha com `REDIS_URL not set`.
**Contramedida**: expor `getRedis()` que lazy-inita no primeiro uso. Nunca instanciar no top-level.

### 0.9.13 pdf-parse 2.x mudou a API (PDFParse class)
A versão 1.x era `const pdf = require("pdf-parse"); pdf(buffer) → {text}`. A 2.x exporta `{ PDFParse }`:
```ts
const { PDFParse } = await import("pdf-parse");
const parser = new PDFParse({ data: new Uint8Array(buf) });
const { text } = await parser.getText();
await parser.destroy();
```

### 0.9.14 Railway CLI `railway add` é headless-unfriendly
Flag `-d postgres` **não** é enough — a CLI entra em TUI que espera `Enter`. Em sessão headless (ex: Claude Code), a workaround funcional é:
```bash
railway add -d postgres < /dev/null 2>&1 & sleep 12 && kill %1 2>/dev/null; wait 2>/dev/null
```
Riscos: pode criar serviços **duplicados**. Validar com `railway status --json | jq '.services.edges[].node.name'` após cada add e deletar duplicatas via dashboard Railway (CLI não tem `service delete` no momento).

### 0.9.15 `railway variable set` — usar `--skip-deploys` em batches
Cada `railway variable set` dispara rollout por default. Em batches, use `--skip-deploys` e rode um `railway redeploy --service <name> -y` no final.
Referências entre serviços com hífen **funcionam**: `"DATABASE_URL=\${{Postgres-GENI.DATABASE_URL}}"` (escapar `$` na bash).

### 0.9.16 Modelos Gemini — confirmar disponibilidade antes
Ao usar Google como provider, validar modelo via:
```bash
curl -s "https://generativelanguage.googleapis.com/v1beta/models?key=$GOOGLE_API_KEY" | jq -r '.models[] | select(.supportedGenerationMethods[]=="generateContent") | .name'
```
Defaults válidos (2026-04): `gemini-3-flash-preview`, `gemini-2.5-flash`, `gemini-flash-latest`. Embeddings: `gemini-embedding-001` (stable) ou `gemini-embedding-2`.

### 0.9.17 UI: botão de reativar IA é obrigatório
Se `conversation.aiEnabled=false` e `handoffRequested=false`, o operador precisa de um botão explícito "Reativar IA". Não esconder atrás de toggle disabled. Lógica correta:
```tsx
{handoffRequested
  ? <Button onClick={() => setAi(true)}>Resolver handoff</Button>
  : aiEnabled
    ? <Button onClick={() => setAi(false)}>Pausar IA</Button>
    : <Button onClick={() => setAi(true)}>Reativar IA</Button>}
```

### 0.9.18 Flush silencioso — logs obrigatórios por fase
Sem logs, flush que falha (ex: send falhou, modelo inexistente, timeout) parece "IA ignorou msg". Obrigatório no `flush.ts`:
- `event="flush.start"` quando começa
- `event="flush.decided"` com a decision
- `event="flush.sending"` antes de `sendText`
- `event="flush.sent_ok"` após sucesso
- `event="flush.send_failed"` no catch
E no webhook: `event="webhook.waha.recv"` e `event="webhook.waha.duplicate"`.

### 0.9.19 Agent config UI (`/agents`) precisa no MVP
§6 menciona "Client Wizard" mas é fácil pular a UI de editar o `AgentSession`. Sem ela, o operador não consegue ajustar `systemPrompt`/modelo/temperature sem `railway run psql`. Incluir `/agents` (list) + `/agents/[id]` (form) **no scaffold inicial**, não depois.

### 0.9.20 Instrumentation skip durante build
`instrumentation.ts` roda no boot e chama `ensureSession()`. Durante `next build`, Next instancia o módulo brevemente (fase `phase-production-build`). Proteger:
```ts
export async function register() {
  if (process.env.NEXT_RUNTIME !== "nodejs") return;
  if (process.env.NEXT_PHASE === "phase-production-build") return;
  // ... ensureSession por canal
}
```

### 0.9.21 @lid vs @c.us como contactId
WhatsApp moderno retorna `201932484403261@lid` para contatos com privacidade ativa. O Waha aceita ambos formatos em `sendText` — **não** converter. Salvar no DB como `externalId` exatamente como veio.

### 0.9.23 Lock TTL do debounce **precisa ser maior** que o timer fire
Bug real: `redis.set(lockKey, token, "PX", windowMs)` dá ao lock o MESMO TTL que a janela do timer. `setTimeout(cb, windowMs + 50)` dispara 50ms **depois** da expiração — o `GET` do lock retorna `null` e nunca flush. Resultado: IA nunca responde.
**Contramedida**: TTL do lock = `windowMs * N` (N ≥ 3) para absorver latência de clock/GC:
```ts
const lockTtlMs = windowMs * 4;
await redis.set(lockKey, token, "PX", lockTtlMs);
```
Também diferenciar no log: `current === null` → `"debounce.lock_expired"` (o lock sumiu — bug de config), `current && current !== token` → `"debounce.superseded"` (msg nova chegou — comportamento correto).

### 0.9.24 Prisma `log: ["error"]` gera ruído sobre violações que você cata
Sob concorrência, `P2002` é **esperado** (design intencional do dedup). Mas `PrismaClient({ log: ["error"] })` imprime `prisma:error` a cada violação, poluindo os logs e assustando o operador que acha que quebrou. Em produção, configurar `log: []` ou `log: ["warn"]`. Seus `try/catch` já registram o evento semântico ("webhook.waha.duplicate") que importa.

### 0.9.25 Debounce com fallback-síncrono se Redis falhar
Se `getRedis()` lançar ou `redis.multi().exec()` rejeitar, o user fica sem resposta para sempre. Sempre encapsular em `try/catch` e, no catch, chamar `flushConversation(conversationId, [msg])` diretamente — perde o debounce mas garante resposta:
```ts
try {
  redis = getRedis();
} catch (err) {
  console.error(JSON.stringify({ event: "debounce.redis_init_failed", err: String(err) }));
  await flushConversation(conversationId, [msg]);
  return;
}
```

### 0.9.27 **NUNCA** `sendText` com `Contact.id` — sempre usar `ContactIdentity.externalId`
Bug real devastador: no `flush.ts`, `conv.contactId` é o **FK interno** (`Contact.id`, um `cuid` tipo `cmoafu5g20001o70197efac0a`). Se passar isso pra `adapter.sendText()`, o Waha tenta resolver `cmoafu5g2...@s.whatsapp.net` no WhatsApp, espera 75s, e devolve `500 failed usync query: info query timed out`. Silencioso pra IA — flush aparece como "sent" na app mas o user nunca recebe.

**Contramedida obrigatória**:
1. O `ContactIdentity.externalId` é a **única** fonte da verdade pro ID do canal (ex: `201932484403261@lid`, `55...@c.us`, `user_id` IG).
2. Na webhook inbound, `msg.contactId` já é o externalId — **mantenha-o** em todo o pipeline de debounce (Redis já serializa o `ChannelMessage` inteiro).
3. No `flushConversation`, extrair o externalId da `msg[0]` com fallback pra lookup em DB:
```ts
const externalId = msgs[0]?.contactId ?? (await (async () => {
  const ident = await prisma.contactIdentity.findFirst({
    where: { contactId: conv.contactId, channel: conv.channel },
  });
  return ident?.externalId;
})());
if (!externalId) {
  console.error(JSON.stringify({ event: "flush.no_external_id", conversationId }));
  return;
}
```
4. Em qualquer endpoint de **reply manual** (`/api/conversations/[id]/reply`), fazer o mesmo lookup — o operador só tem o `conv.id` na URL.
5. Validar em dev local: antes de mandar pra Waha/etc, `assert(contactId.includes("@"))` — IDs de WhatsApp sempre têm `@c.us` ou `@lid`; cuids nunca têm `@`. Testa rápido sem DB.

**Por que é fácil cometer esse erro**: o nome `contactId` aparece em 3 lugares diferentes com significados diferentes:
- `ChannelMessage.contactId` — externalId do canal (certo pra `sendText`)
- `Contact.id` — cuid interno (errado)
- `Conversation.contactId` — FK pra `Contact.id` (errado)
- `ContactIdentity.externalId` — externalId do canal (certo)

Renomear em tipagem seria ideal (`Contact.id` → `contactPk`, `ChannelMessage.contactId` → `externalId`), mas se não renomear, adicionar JSDoc em cada um + esta regra no PR checklist.

### 0.9.26 Railway rename de serviço preserva o private domain antigo
Se o operador renomear `Postgres-GENI` → `Postgres` pela dashboard, o hostname interno continua `postgres-geni.railway.internal`. MAS a referência `${{Postgres-GENI.DATABASE_URL}}` deixa de resolver (nome de serviço mudou). **Nunca** referenciar serviços em env vars pelo nome com sufixo randômico — renomear **antes** de setar as vars, ou usar a syntax resolvida direta.

### 0.9.22 Checklist pré-deploy (run no final do §4)
Antes de `railway up`, rodar localmente:
- [ ] `DATABASE_URL=... AUTH_SECRET=$(openssl rand -hex 32) npm run build` — zero erros TS
- [ ] `find .next/standalone -name server.js` — confirma path do CMD
- [ ] `grep -r 'message\.any' src/` — zero matches (pré-mitigação 0.9.4)
- [ ] `grep -n 'url.*env(' prisma/schema.prisma` — existe (pré-mitigação 0.9.1 com Prisma 6)
- [ ] `ls prisma/migrations/000_init/migration.sql` — SQL não-vazio
- [ ] `ls prisma/migrations/migration_lock.toml` — existe

---

## 1. PRÉ-FLIGHT — CLIs, SKILLS E CREDENCIAIS

Rodar este bloco **inteiro em paralelo**. Cada item tem 3 estados: `OK`, `AUTO-INSTALLED`, `BLOQUEIO`. Só bloquear a sessão se houver **BLOQUEIO**. Auto-install é silencioso.

### 1.1 Script de pré-flight (copiar e rodar)

Claude deve executar este script via Bash **antes de qualquer outra coisa**. Ele imprime um relatório JSON no final que Claude parseia.

```bash
#!/usr/bin/env bash
# preflight.sh — produz JSON com status de cada dep. Nunca sai com código != 0.
set +e
REPORT=$(mktemp)
echo '{' > "$REPORT"

# --- Node 20+
NODE_VER=$(node --version 2>/dev/null | sed 's/v//' | cut -d. -f1)
if [[ "$NODE_VER" -ge 20 ]] 2>/dev/null; then
  echo '  "node": {"status":"OK","version":"'"$(node --version)"'"},' >> "$REPORT"
else
  echo '  "node": {"status":"BLOCK","install":"brew install node@20 OR https://nodejs.org"},' >> "$REPORT"
fi

# --- Railway CLI (auto-install via npm se faltar; brew é fallback)
if ! command -v railway >/dev/null; then
  npm i -g @railway/cli >/dev/null 2>&1 || brew install railway >/dev/null 2>&1
fi
if command -v railway >/dev/null; then
  echo '  "railway": {"status":"OK","version":"'"$(railway --version 2>&1 | head -1)"'"},' >> "$REPORT"
else
  echo '  "railway": {"status":"BLOCK","install":"npm i -g @railway/cli OR brew install railway"},' >> "$REPORT"
fi

# --- git
command -v git >/dev/null && echo '  "git": {"status":"OK"},' >> "$REPORT" \
  || echo '  "git": {"status":"BLOCK","install":"brew install git / xcode-select --install"},' >> "$REPORT"

# --- jq (usado no §8 p/ parsear respostas Waha)
if ! command -v jq >/dev/null; then brew install jq >/dev/null 2>&1 || apt-get install -y jq >/dev/null 2>&1; fi
command -v jq >/dev/null && echo '  "jq": {"status":"OK"},' >> "$REPORT" \
  || echo '  "jq": {"status":"WARN","note":"opcional — usado apenas na verificação final"},' >> "$REPORT"

# --- openssl (gerar secrets)
command -v openssl >/dev/null && echo '  "openssl": {"status":"OK"},' >> "$REPORT" \
  || echo '  "openssl": {"status":"BLOCK","install":"brew install openssl"},' >> "$REPORT"

# --- Railway auth
if railway whoami >/dev/null 2>&1; then
  echo '  "railway_auth": {"status":"OK","user":"'"$(railway whoami 2>/dev/null | head -1)"'"},' >> "$REPORT"
elif [[ -n "$RAILWAY_TOKEN" ]]; then
  echo '  "railway_auth": {"status":"OK","method":"token"},' >> "$REPORT"
else
  echo '  "railway_auth": {"status":"NEEDS_LOGIN","action":"railway login (abre browser) OR export RAILWAY_TOKEN=..."},' >> "$REPORT"
fi

# --- TTY (necessário p/ railway login interativo)
if [[ -t 0 ]]; then
  echo '  "tty": {"status":"OK"},' >> "$REPORT"
else
  echo '  "tty": {"status":"HEADLESS","note":"railway login interativo indisponível — usar RAILWAY_TOKEN"},' >> "$REPORT"
fi

# --- Path safety (evitar Google Drive, iCloud, caminhos com espaço/acento)
CWD="$(pwd)"
if echo "$CWD" | grep -qE '(CloudStorage|Google Drive|iCloud|Dropbox| )'; then
  echo '  "path_safety": {"status":"UNSAFE","cwd":"'"$CWD"'","note":"scaffoldar fora deste path — usar ~/Projects"},' >> "$REPORT"
else
  echo '  "path_safety": {"status":"OK","cwd":"'"$CWD"'"},' >> "$REPORT"
fi

echo '  "_end": true' >> "$REPORT"
echo '}' >> "$REPORT"
cat "$REPORT"
rm "$REPORT"
```

### 1.2 Regras sobre o relatório

Ação do Claude baseada no JSON:
- Qualquer item `"status":"BLOCK"` → imprimir a linha `install` e **parar**. Não prosseguir.
- `"railway_auth":{"status":"NEEDS_LOGIN"}` + `"tty":"HEADLESS"` → pedir ao usuário `export RAILWAY_TOKEN=...` e reperguntar. Se TTY OK, rodar `railway login` como próximo passo (usuário conclui no browser).
- `"path_safety":"UNSAFE"` → **obrigatório** perguntar via AskUserQuestion onde criar o repo. Default sugerido: `~/Projects/whatsapp-ai-agent`. Nunca scaffoldar dentro de CloudStorage/iCloud/caminhos com espaço (quebra `pnpm`, symlinks, node-pty).

### 1.3 Context7 (MCP) — probe funcional
Não confiar em nome de tool; executar uma chamada real:
```
mcp__context7__resolve-library-id(libraryName="Next.js", query="test")
```
Se retornar `Available Libraries` → OK.
Se retornar erro/timeout → pedir ao usuário:
```bash
claude mcp add context7 -- npx -y @upstash/context7-mcp@latest
```
E orientar a reiniciar o Claude Code.

### 1.4 Skills (invocar conforme necessário, sem forçar todas no preflight)
- `superpowers:brainstorming` — se perguntas do §2 retornarem ambíguas
- `superpowers:writing-plans` — ao gerar §3 (SPEC.md)
- `superpowers:executing-plans` — ao executar §4–§7
- `superpowers:systematic-debugging` — se qualquer passo falhar
- `superpowers:verification-before-completion` — obrigatório antes do §9
- `frontend-design:frontend-design` — §6 (Client Wizard UI)
- `claude-api` — se provedor Anthropic escolhido no §2

### 1.5 Credenciais — quando coletar
| Credencial | Quando | Como |
|---|---|---|
| Railway auth | §1 pré-flight | `railway login` (TTY) ou `RAILWAY_TOKEN` (headless) |
| LLM API key (1+) | §2 Onda 2 (via AskUserQuestion → "Other" c/ key) | Gravada em `setup.config.json` (gitignored) |
| Google Cal OAuth | §6 Client Wizard (opcional) | OAuth flow no painel |
| Tavily key | §6 Client Wizard (opcional) | Campo no painel |

Não pedir keys fora do fluxo acima.

---

## 2. DEV WIZARD — DESCOBERTA VIA AskUserQuestion

Claude **DEVE** usar a tool `AskUserQuestion` (não texto livre) para todas as decisões abaixo. Agrupe em ondas de no máximo 4 perguntas por chamada. Cada pergunta tem 2–4 opções; "Other" é inserido pelo sistema.

### Onda 0 — Meta (nome + path do projeto)
**Não** usar AskUserQuestion aqui (entrada livre). Perguntar via texto:
```
"Qual o nome do projeto (kebab-case, ex: meu-agente-whats)?
 Qual diretório criar (default: ~/Projects/<nome>)?"
```
Validar: `^[a-z][a-z0-9-]{2,40}$`. Gravar em `project.name`, `project.slug`, `project.repoPath` do `setup.config.json`.
Se `project.repoPath` cair dentro de `CloudStorage/Google Drive/iCloud/Dropbox`, **rejeitar** e reperguntar.

### Onda 1 — Canais + uso + escopo
```yaml
questions:
  - header: "Canais de mensagem"
    question: "Quais canais o agente vai atender? (multi-select — pode combinar)"
    multiSelect: true
    options:
      - label: "WhatsApp · Waha Core (Recommended)"
        description: "Free. 1 sessão, texto-only. Engine GOWS (sem browser, ideal Railway)."
      - label: "WhatsApp · Waha Plus ($19/mo)"
        description: "Mídia (áudio/imagem/vídeo/docs), múltiplas sessões, suporte."
      - label: "WhatsApp · Evolution API"
        description: "Self-hosted open-source (Baileys). Texto + mídia + grupos. Container próprio."
      - label: "WhatsApp Cloud API (oficial Meta)"
        description: "API oficial. Exige Business verificado + Phone Number ID. Preço por conversa (~24h window)."
      - label: "Instagram Direct (Meta Graph API)"
        description: "DMs do Instagram Business. Requer Page + review de permissões (instagram_manage_messages)."

  - header: "Caso de uso"
    question: "Qual é o principal caso de uso deste agente?"
    options:
      - label: "SDR/Pré-vendas (Recommended)"
        description: "Qualifica lead, coleta dados, marca reunião"
      - label: "Atendimento/Suporte"
        description: "Responde FAQ, abre chamado, escala para humano"
      - label: "Agendamento/Reservas"
        description: "Consulta agenda, marca/cancela horários"
      - label: "Assistente pessoal"
        description: "Conversa geral + lembretes + integrações"

  - header: "Volume"
    question: "Quantas conversas simultâneas você espera no pico?"
    options:
      - label: "Até 10 (Recommended)"
        description: "MVP/validação. Infra mínima."
      - label: "10–100"
        description: "Operação pequena. Waha Core já atende; Evolution/Cloud API escalam melhor."
      - label: "100+"
        description: "Exige Waha Plus (multi-sessão) OU Cloud API OU Evolution API multi-instance."

  - header: "Idioma"
    question: "O agente responde em qual idioma principal?"
    options:
      - label: "Português (BR) (Recommended)"
        description: "System prompt e UI em pt-BR"
      - label: "English"
        description: "System prompt e UI em inglês"
      - label: "Bilíngue (PT/EN)"
        description: "Detecta idioma da mensagem e espelha"
```

**Regras de validação da Onda 1 (Canais):**
- Se usuário marcou `Waha Core` **e** `Waha Plus` simultaneamente → reperguntar (são exclusivos: Plus substitui Core).
- Se marcou `Waha Core` **e** `Volume 100+` → avisar no SPEC que Core não atende e sugerir trocar para Plus/Evolution/Cloud API.
- Se marcou **apenas** `Instagram Direct` → avisar que Cloud API da Meta recomenda contra Instagram-only sem fallback WhatsApp.
- Se marcou `WhatsApp Cloud API` → coletar no §2.5 credenciais extras: `META_APP_ID`, `META_APP_SECRET`, `META_PHONE_NUMBER_ID`, `META_WEBHOOK_VERIFY_TOKEN`.
- Se marcou `Instagram Direct` → coletar: `META_IG_BUSINESS_ACCOUNT_ID` + mesmas credenciais Meta acima.
- Se marcou `Evolution API` → container extra no §7 (imagem `atendai/evolution-api:latest`, porta 8080).

### Onda 2 — Modelo de IA
```yaml
questions:
  - header: "Provedor LLM"
    question: "Qual provedor de IA usar como PADRÃO?"
    multiSelect: true
    options:
      - label: "Anthropic Claude (Recommended)"
        description: "claude-sonnet-4-6 default. Suporta prompt caching."
      - label: "OpenAI"
        description: "gpt-4o-mini default. Ecossistema maduro."
      - label: "Google Gemini"
        description: "gemini-3-flash-preview default (chat). gemini-embedding-2 p/ RAG MULTIMODAL (texto/imagem/áudio/vídeo/PDF mesmo espaço)."
      - label: "Groq (Llama 3.x)"
        description: "Latência sub-500ms, bom p/ WhatsApp real-time."

  - header: "Fallback"
    question: "Se o provedor principal falhar, qual comportamento?"
    options:
      - label: "Fallback para outro provedor (Recommended)"
        description: "Roteia para o segundo provedor configurado"
      - label: "Mensagem de erro educada"
        description: 'Responde "estou indisponível, tente mais tarde"'
      - label: "Fila + retry"
        description: "Guarda a msg e tenta 3x com backoff"

  - header: "Memória"
    question: "Como o agente deve lembrar de conversas anteriores?"
    options:
      - label: "Últimas 20 msgs por contato (Recommended)"
        description: "Janela deslizante. Barato e suficiente p/ MVP."
      - label: "Resumo persistente + últimas 10"
        description: "LLM resume conversa ao atingir N tokens"
      - label: "Sem memória"
        description: "Cada msg é stateless"
```

### Onda 3 — Tools + Handoff + Horário
```yaml
questions:
  - header: "Tools"
    question: "Quais ferramentas o agente precisa invocar?"
    multiSelect: true
    options:
      - label: "Calendário (Google Cal/Cal.com)"
        description: "list_events, create_event, check_availability"
      - label: "Web search (Tavily)"
        description: "Busca real-time p/ responder sobre eventos atuais"
      - label: "Knowledge base / RAG"
        description: "Upload de PDFs/docs como contexto (embeddings)"
      - label: "Webhook custom"
        description: "POST p/ sistema externo (CRM, planilha, Zapier)"

  - header: "Handoff trigger"
    question: "Quando o agente deve transferir para humano?"
    options:
      - label: "Sob pedido explícito (Recommended)"
        description: 'Palavras-chave tipo "falar com atendente"'
      - label: "Baixa confiança"
        description: "Quando o modelo sinaliza incerteza (tool call dedicada)"
      - label: "Nunca (100% IA)"
        description: "Sempre responde com IA"

  - header: "Handoff destino"
    question: "Quando transferir para humano, para onde vai a notificação?"
    options:
      - label: "Mesmo canal — outro número/conta (Recommended)"
        description: "Envia via o próprio provedor (Waha/Evolution/Cloud API/IG) p/ atendente"
      - label: "E-mail"
        description: "Dispara email com transcript"
      - label: "Slack/Discord webhook"
        description: "Posta em canal de atendimento"

  - header: "Horário"
    question: "O agente respeita horário de atendimento?"
    options:
      - label: "24/7 (Recommended)"
        description: "Sempre ativo"
      - label: "Horário comercial"
        description: "Fora do horário: msg automática de ausência"
```

### Onda 4 — Deploy / Infra
```yaml
questions:
  - header: "Persistência"
    question: "Onde armazenar conversas, configs e sessão Waha?"
    options:
      - label: "Postgres Railway (Recommended)"
        description: "Plugin nativo. Prisma como ORM."
      - label: "SQLite em volume"
        description: "Mais simples. Single-instance."
      - label: "Supabase externo"
        description: "Se já tem conta. Usa Supabase JS."

  - header: "Debounce de mensagens"
    question: "Quanto tempo aguardar entre mensagens do mesmo contato antes de responder? (evita responder cada fragmento separadamente)"
    options:
      - label: "5 segundos (Recommended)"
        description: "Balanço bom: coalesce msgs em sequência, UX natural. Requer Redis."
      - label: "8 segundos"
        description: "Para usuários que digitam devagar / mandam muitas msgs curtas."
      - label: "3 segundos"
        description: "Respostas mais rápidas, risco de quebrar msgs longas."
      - label: "Desativado"
        description: "Responde cada mensagem isolada. Sem Redis. Não recomendado p/ WhatsApp."

  - header: "Auth do painel"
    question: "Como proteger o painel web de configuração?"
    options:
      - label: "Magic link por email (Recommended)"
        description: "Auth.js com provider email. Barato e seguro."
      - label: "Senha simples + env var"
        description: "Uma senha única no .env. MVP only."
      - label: "Clerk"
        description: "Se já usa em outros projetos (ai-code-lab usa)"

  - header: "Domínio"
    question: "Qual domínio usar para o painel?"
    options:
      - label: "Subdomínio Railway (Recommended)"
        description: "app.up.railway.app gerado automaticamente"
      - label: "Domínio custom"
        description: "Usuário fornece DNS p/ apontar CNAME"
```

**Regra**: Se `Debounce != Desativado` → adicionar plugin **Redis** na Railway (§7.2.1) e gravar `cache.enabled: true` em `setup.config.json`. Redis é **obrigatório** p/ debounce funcionar em ambiente multi-worker.

> **Regra:** Claude **não prossegue para o Bloco 3** sem ter as respostas das 4 ondas. Se o usuário escolher "Other" em qualquer uma, Claude deve reperguntar 1x pedindo detalhe antes de seguir.

### 2.5 Credenciais de canais Meta (condicional)

Executado **somente** se `channels.whatsapp_cloud.enabled == true` **OU** `channels.instagram_direct.enabled == true`. Claude pede via texto livre (não AskUserQuestion — são strings opacas):

**Se `whatsapp_cloud` habilitado:**
```
Cole suas credenciais do Meta WhatsApp Business (https://business.facebook.com → seu App → WhatsApp):
  META_APP_ID=
  META_APP_SECRET=
  META_PHONE_NUMBER_ID=
  META_BUSINESS_ACCOUNT_ID=
  META_SYSTEM_USER_TOKEN=   (token permanente — não é token de sessão)
```

**Se `instagram_direct` habilitado:**
```
Cole credenciais Instagram (mesmo App Meta, aba Instagram Graph):
  META_IG_BUSINESS_ACCOUNT_ID=
  META_IG_PAGE_ID=
  META_IG_PAGE_ACCESS_TOKEN=
```

Claude grava estas credenciais em `.railway/meta.env` e `.railway/meta-ig.env` (ambos gitignored, carregados pelo §7.3 no `.env.app`). **Nunca** gravar em `setup.config.json`.

Um `META_WEBHOOK_VERIFY_TOKEN` é gerado localmente via `openssl rand -hex 32` e gravado em `.railway/secrets.json` — este é o token que o usuário cola no painel Meta App Dashboard na configuração do webhook.

Se o usuário ainda não tem credenciais Meta:
1. Claude pausa e instrui a criar app em https://developers.facebook.com/apps → "Business" type
2. Adicionar produtos "WhatsApp" e/ou "Instagram Graph API"
3. Voltar ao wizard com `--resume`

---

## 3. ARTEFATOS GERADOS (machine-readable + humano)

Claude emite **dois arquivos** a partir das respostas das ondas:

### 3.1 `setup.config.json` (fonte de verdade — downstream lê daqui)
Gravar em `$REPO_PATH/setup.config.json`. Este é o contrato entre todos os blocos. Nunca hardcodar valores fora daqui.

```json
{
  "$schema": "./setup.config.schema.json",
  "version": 2,
  "project": {
    "name": "whatsapp-ai-agent",
    "slug": "whatsapp-ai-agent",
    "repoPath": "~/Projects/whatsapp-ai-agent",
    "createdAt": "2026-04-20T12:34:56Z"
  },
  "channels": {
    "enabled": ["waha_core"],
    "waha_core":        { "enabled": true,  "session": "default", "engine": "GOWS", "image": "devlikeapro/waha:latest" },
    "waha_plus":        { "enabled": false, "session": "default", "engine": "GOWS", "image": "devlikeapro/waha-plus:latest" },
    "evolution_api":    { "enabled": false, "image": "atendai/evolution-api:latest", "instanceName": "default", "port": 8080 },
    "whatsapp_cloud":   { "enabled": false, "phoneNumberId": null, "businessAccountId": null, "webhookVerifyToken": null },
    "instagram_direct": { "enabled": false, "igBusinessAccountId": null, "pageId": null }
  },
  "useCase": "sdr | support | scheduling | assistant",
  "language": "pt-BR | en | bilingual",
  "volume": "low | medium | high",
  "handoff": { "trigger": "explicit | low_confidence | never", "destination": "same_channel | email | slack" },
  "llm": {
    "providers": ["anthropic", "openai"],
    "primary": "anthropic",
    "defaultModel": "claude-sonnet-4-6",
    "fallbackPolicy": "provider | polite_error | queue_retry",
    "memory": "window_20 | summary_plus_10 | stateless"
  },
  "tools": {
    "calendar": false,
    "webSearch": false,
    "rag": false,
    "webhook": false
  },
  "schedule": { "mode": "24_7 | business_hours", "timezone": "America/Sao_Paulo" },
  "cache": {
    "enabled": true,
    "provider": "redis_railway",
    "debounce": {
      "enabled": true,
      "windowMs": 5000,
      "maxBufferedMessages": 20,
      "keyPrefix": "debounce"
    }
  },
  "infra": {
    "db": "postgres_railway | sqlite_volume | supabase",
    "auth": "magic_link | password | clerk",
    "domain": "railway_subdomain | custom",
    "customDomain": null
  },
  "checkpoint": "preflight | wizard | scaffold | railway | verify | done",
  "warnings": []
}
```

**Regras de normalização (Claude aplica após coletar respostas das ondas):**

1. **Canais mutuamente exclusivos**: se `channels.waha_core.enabled == true` **e** `channels.waha_plus.enabled == true` → manter só `waha_plus` e adicionar warning `"waha_core e waha_plus são exclusivos — usando waha_plus"`.
2. **Volume vs Core**: se `volume == "high"` e `channels.waha_core.enabled == true` e `waha_plus.enabled == false` → adicionar warning `"Waha Core limita a 1 sessão; volume high recomenda waha_plus, evolution_api, ou whatsapp_cloud"`.
3. **Mídia + Core**: se `tools.rag == true` com necessidade de enviar PDFs/áudio pelo WhatsApp e só `waha_core` habilitado → warning `"Waha Core não envia mídia — upgrade para waha_plus ou adicione evolution_api"`.
4. **Debounce sem Redis**: se `cache.debounce.enabled == true` então `cache.enabled` **deve** ser `true` e `cache.provider` **deve** ser `"redis_railway"`. Se `cache.debounce.enabled == false` → pode-se setar `cache.enabled: false` e não provisionar Redis no §7.
5. **Credenciais Meta**: se `channels.whatsapp_cloud.enabled == true` ou `channels.instagram_direct.enabled == true` → coletar credenciais Meta no §2.5 e gravar em `.railway/.env.app` (nunca no `setup.config.json`).

### 3.2 `SPEC.md` (humano — confirmação com o usuário)
Renderizar um markdown a partir do JSON acima, **com os valores resolvidos** (sem `{{placeholders}}`). Template:

```markdown
# SPEC — <project.name>

## Caso de uso
<useCase> · idioma <language> · volume <volume>

## Canais habilitados
<lista formatada a partir de channels.enabled — ex: "WhatsApp (Waha Core) + Instagram Direct">

## Stack
- Next.js 16 (App Router, standalone build)
- shadcn/ui "nova" · base zinc · radius 0.375rem · accent #adff2f (Alien Green)
- Geist + Geist Mono
- <infra.db> (Prisma)
- <cache.provider> p/ debounce + rate-limit (se cache.enabled)
- Auth: <infra.auth>
- Containers provisionados conforme canais:
  - sempre: `app` (Next.js) + Postgres + Redis (se debounce ativo)
  - `waha` (imagem <channels.waha_core.image OU channels.waha_plus.image>) — se Waha habilitado
  - `evolution` (imagem <channels.evolution_api.image>) — se Evolution habilitado
  - Cloud API / Instagram usam só webhooks externos da Meta (sem container próprio)

## LLM
- Principal: <llm.primary> (<llm.defaultModel>)
- Fallback: <llm.fallbackPolicy>
- Memória: <llm.memory>

## Debounce de mensagens
- Ativo: <cache.debounce.enabled>
- Janela: <cache.debounce.windowMs>ms
- Buffer máximo: <cache.debounce.maxBufferedMessages> msgs por contato
- Store: Redis (chave `<cache.debounce.keyPrefix>:<channel>:<contact>`)

## Tools habilitadas
<lista formatada a partir de tools>

## Handoff
Trigger: <handoff.trigger> · Destino: <handoff.destination>

## Horário
<schedule.mode> (<schedule.timezone>)

## ⚠️ Avisos
<renderizar warnings[]>

## Critérios de aceite (§8 valida)
1. Cada canal em `channels.enabled` conecta em <5 min pelo onboarding
2. Enviar 3 msgs em sequência (intervalo <5s) no mesmo contato → receber **1** resposta coalesced em ~<cache.debounce.windowMs>ms + LLM latency
3. Alternar modelo no painel → próxima msg usa o novo modelo
4. Ativar tool "calendário" → mensagem "o que tenho hoje?" retorna eventos
5. Todos os serviços (app, postgres, redis se ativo, + container por canal) healthy na Railway
6. Redis `PING` retorna `PONG` via `railway run --service app -- redis-cli ping`
```

### 3.3 Confirmação
Imprimir o `SPEC.md` renderizado. **Não** usar AskUserQuestion aqui (resposta livre). Pedir: `"Digite 'ok' para scaffoldar, 'ajustar' para rever perguntas, ou 'cancelar' para abortar."`

### 3.4 Checkpoint
Após confirmação, escrever:
```
~/.claude-wizard/whatsapp-ai-agent.state.json
```
com `{ "phase": "wizard_done", "repoPath": "...", "configPath": ".../setup.config.json" }`.
Este arquivo é lido pelo `--resume`.

---

## 4. SCAFFOLD — ESTRUTURA DO REPO

```
whatsapp-ai-agent/
├── apps/web/                     # Next.js
│   ├── Dockerfile                # multistage, standalone (copiar de ai-code-lab)
│   ├── railway.toml
│   ├── components.json           # shadcn nova + zinc + 0.375rem
│   ├── src/
│   │   ├── app/
│   │   │   ├── (public)/login/
│   │   │   ├── (app)/
│   │   │   │   ├── onboarding/   # CLIENT WIZARD — ver §6
│   │   │   │   ├── dashboard/
│   │   │   │   ├── conversations/
│   │   │   │   ├── agents/       # prompts + modelo
│   │   │   │   ├── tools/
│   │   │   │   └── settings/
│   │   │   ├── api/
│   │   │   │   ├── webhooks/                   # 1 rota por canal em channels.enabled
│   │   │   │   │   ├── waha/route.ts              # Waha Core + Plus (mesmo formato)
│   │   │   │   │   ├── evolution/route.ts         # Evolution API (Baileys event shape)
│   │   │   │   │   ├── whatsapp-cloud/route.ts    # Meta Cloud API (GET verify + POST events)
│   │   │   │   │   └── instagram/route.ts         # Meta Graph (DMs Instagram Business)
│   │   │   │   ├── sessions/
│   │   │   │   │   └── [channel]/
│   │   │   │   │       ├── ensure/route.ts     # POST → chama adapter.ensureSession() (§5.4)
│   │   │   │   │       ├── status/route.ts     # GET → adapter.getSessionStatus()
│   │   │   │   │       ├── qr/route.ts         # GET → adapter.getQrCode() (base64)
│   │   │   │   │       └── logout/route.ts     # POST → adapter.stopSession/logout
│   │   │   │   ├── chat/route.ts               # LLM orchestrator (chamado pelo debounce flush)
│   │   │   │   ├── health/route.ts             # GET /api/health -> {ok:true}
│   │   │   │   └── tools/[name]/route.ts
│   │   │   └── globals.css        # COPIAR DESIGN TOKENS de /Apps/auto-cut/src/app/globals.css
│   │   ├── instrumentation.ts     # Next.js boot hook — chama ensureSession() de cada canal habilitado
│   │   ├── components/ui/         # shadcn nova
│   │   ├── lib/
│   │   │   ├── channels/          # client por canal (normalizam p/ ChannelMessage comum)
│   │   │   │   ├── types.ts          # ChannelMessage, ChannelAdapter, SessionState, OutboundMessage
│   │   │   │   ├── registry.ts       # factory: lê channels.enabled → devolve adapters ativos
│   │   │   │   ├── waha.ts           # REST Waha + ensureSession idempotente
│   │   │   │   ├── evolution.ts      # REST Evolution API + ensureSession idempotente
│   │   │   │   ├── whatsapp-cloud.ts # Meta Cloud API + subscription idempotente
│   │   │   │   └── instagram.ts      # Meta Graph Instagram DMs + subscription idempotente
│   │   │   ├── llm/               # provedores (openai, anthropic, google, groq)
│   │   │   ├── tools/             # implementações (calendar, search, rag, webhook)
│   │   │   ├── redis.ts           # cliente ioredis (singleton) — lê REDIS_URL
│   │   │   ├── debounce.ts        # coalesce de mensagens por contato+canal (Redis)
│   │   │   ├── memory.ts
│   │   │   └── db.ts              # Prisma
│   │   └── types/
│   ├── prisma/schema.prisma
│   └── package.json
├── infra/
│   ├── waha.Dockerfile            # FROM devlikeapro/waha:latest
│   ├── waha.railway.toml
│   └── README.md
├── SPEC.md
├── README.md
└── .env.example
```

### 4.0 Prisma schema (fonte da verdade do modelo de domínio)

`apps/web/prisma/schema.prisma` — skeleton mínimo viável que cobre contato cross-channel, conversa, msg, sessão de agente, regras de IA, RAG:

```prisma
generator client { provider = "prisma-client-js" }
datasource db { provider = "postgresql"; url = env("DATABASE_URL") }

// ─── CONTATO cross-channel ──────────────────────────────────────────
model Contact {
  id            String   @id @default(cuid())
  displayName   String?
  notes         String?
  tags          String[] @default([])
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt
  identities    ContactIdentity[]
  conversations Conversation[]
  @@index([tags], type: Gin)
}
// Cada canal pode ter 1+ identidade para o mesmo humano (dedup: email comum, número comum)
model ContactIdentity {
  id         String   @id @default(cuid())
  contactId  String
  contact    Contact  @relation(fields: [contactId], references: [id], onDelete: Cascade)
  channel    String   // "waha" | "evolution" | "whatsapp_cloud" | "instagram"
  externalId String   // phone@c.us, phone, ig_user_id
  @@unique([channel, externalId])
  @@index([contactId])
}

// ─── CONVERSAS + MSGS ───────────────────────────────────────────────
model Conversation {
  id          String   @id @default(cuid())
  contactId   String
  contact     Contact  @relation(fields: [contactId], references: [id], onDelete: Cascade)
  channel     String
  agentSessionId String?
  agentSession   AgentSession? @relation(fields: [agentSessionId], references: [id])
  aiEnabled      Boolean @default(true)   // override local — manual toggle do atendente
  lastMessageAt  DateTime?
  createdAt      DateTime @default(now())
  messages       Message[]
  @@index([contactId, channel])
  @@index([lastMessageAt])
}
model Message {
  id             String   @id @default(cuid())
  conversationId String
  conversation   Conversation @relation(fields: [conversationId], references: [id], onDelete: Cascade)
  direction      String   // "inbound" | "outbound"
  role           String   // "user" | "assistant" | "system" | "tool"
  text           String
  mediaUrl       String?
  providerMsgId  String?  @unique  // idempotência (mesmo event do provedor não duplica)
  tokensIn       Int?
  tokensOut      Int?
  costUsd        Decimal? @db.Decimal(10, 6)
  modelUsed      String?
  createdAt      DateTime @default(now())
  @@index([conversationId, createdAt])
}

// ─── AGENT SESSION (persona/config reutilizável) ────────────────────
// "Sessão" aqui = configuração do agente IA, não da sessão do canal!
// Ex: "SDR B2B", "Suporte técnico", "Atendimento pós-venda" — 1 app pode ter N
model AgentSession {
  id            String   @id @default(cuid())
  name          String   @unique
  systemPrompt  String
  model         String   @default("claude-sonnet-4-6")
  provider      String   @default("anthropic")
  temperature   Float    @default(0.7)
  maxTokens     Int      @default(1024)
  memoryMode    String   @default("window_20")  // window_N | summary_plus_N | stateless
  tools         Json     @default("[]")         // array de tool ids habilitados
  isDefault     Boolean  @default(false)
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt
  conversations Conversation[]
  aiRules       AIRule[]
}

// ─── AI ACTIVATION RULES (ver §5.5) ────────────────────────────────
model AIRule {
  id              String   @id @default(cuid())
  agentSessionId  String
  agentSession    AgentSession @relation(fields: [agentSessionId], references: [id], onDelete: Cascade)
  priority        Int      @default(100)   // menor = avaliado antes
  enabled         Boolean  @default(true)
  mode            String   // "always_on" | "contact_whitelist" | "contact_blacklist"
                           // | "keyword_trigger" | "keyword_pause" | "schedule" | "manual_approval"
  params          Json     // shape depende do mode (ex: {contacts: [...], keywords: [...], schedule: {...}})
  action          String   @default("respond")  // "respond" | "drop" | "handoff" | "static_reply"
  staticReply     String?                        // usado se action = static_reply
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt
  @@index([agentSessionId, priority])
}

// ─── RAG (se channels.tools.rag == true) ────────────────────────────
model RagDoc {
  id              String   @id @default(cuid())
  agentSessionId  String
  title           String
  sourceType      String   // "pdf" | "url" | "text"
  sourceRef       String   // path/url/raw
  checksum        String   @unique
  createdAt       DateTime @default(now())
  chunks          RagChunk[]
}
model RagChunk {
  id        String                 @id @default(cuid())
  docId     String
  doc       RagDoc                 @relation(fields: [docId], references: [id], onDelete: Cascade)
  ord       Int
  text      String
  // pgvector: 768 dims (aceita 1536/3072). Modelo default: gemini-embedding-2 (multimodal).
  // Fallback: gemini-embedding-001 (text-only). Dim deve bater com outputDimensionality do SDK.
  embedding       Unsupported("vector(768)")?
  embeddingModel  String   @default("gemini-embedding-2")  // grava qual modelo gerou cada vetor — evita mistura cross-model
  modality        String   @default("text")                // "text" | "image" | "audio" | "video" | "pdf" | "multimodal"
  @@index([docId, ord])
  @@index([embeddingModel])
}

// ─── ONBOARDING/GUARDRAILS ─────────────────────────────────────────
model OnboardingState {
  id        String   @id @default("singleton")
  step      String   // "channels" | "agent" | "rag" | "test" | "done"
  data      Json     @default("{}")
  updatedAt DateTime @updatedAt
}
model UsageDaily {
  id             String   @id @default(cuid())
  date           DateTime @db.Date
  contactId      String?
  tokensIn       Int      @default(0)
  tokensOut      Int      @default(0)
  costUsd        Decimal  @default(0) @db.Decimal(10, 6)
  @@unique([date, contactId])
  @@index([date])
}
```

**Nota pgvector**: habilitar extensão antes do `migrate deploy` — `CREATE EXTENSION IF NOT EXISTS vector;` (adicionar em `prisma/migrations/000_init/migration.sql` prepend ou rodar `railway run --service app -- psql $DATABASE_URL -c 'CREATE EXTENSION ...'` antes).

### 4.1 Design system — copiar com fallback
Os tokens de referência vivem em `$REFERENCE_AUTO_CUT` (env var, default: `~/Applications/Apps/auto-cut`). Script que **não quebra** se o path não existir:

```bash
REF_AUTO_CUT="${REFERENCE_AUTO_CUT:-$HOME/Applications/Apps/auto-cut}"
REF_AI_LAB="${REFERENCE_AI_CODE_LAB:-$HOME/Applications/Apps/ai-code-lab/web}"

if [[ -f "$REF_AUTO_CUT/src/app/globals.css" ]]; then
  cp "$REF_AUTO_CUT/src/app/globals.css" apps/web/src/app/globals.css
  cp "$REF_AUTO_CUT/components.json"      apps/web/components.json
  echo "✅ Design tokens copiados de $REF_AUTO_CUT"
else
  echo "⚠️  Referência auto-cut não encontrada em $REF_AUTO_CUT"
  echo "   Alternativas:"
  echo "   1. Defina REFERENCE_AUTO_CUT=/caminho/correto e rode de novo"
  echo "   2. Deixe Claude gerar tokens equivalentes (Alien Green + zinc + 0.375rem)"
  echo "      — valores canônicos em §11 deste prompt"
fi
```

Se não encontrar a referência, Claude deve **gerar** `globals.css` com os tokens canônicos listados no §11 (não inventar paleta nova).

### 4.2 Dockerfile (template inline, não depende de cópia)
Gravar em `apps/web/Dockerfile` — idêntico ao padrão Next.js 16 standalone, sem depender de ai-code-lab:

```dockerfile
FROM node:20-alpine AS base
FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm ci

FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ARG NEXT_PUBLIC_APP_URL
RUN npm run build

FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production
RUN addgroup --system --gid 1001 nodejs && adduser --system --uid 1001 nextjs
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
USER nextjs
EXPOSE 3000
ENV PORT=3000 HOSTNAME=0.0.0.0
CMD ["node", "server.js"]
```

`next.config.ts` deve conter `output: "standalone"`.

### 4.3 Container Waha
Na Railway **não se constrói Dockerfile para o Waha** — referenciamos a imagem oficial diretamente (`devlikeapro/waha:latest`) e passamos tudo via env vars + volume. O arquivo `infra/waha.railway.toml` fica:

```toml
# infra/waha.railway.toml
[build]
builder = "image"
image = "devlikeapro/waha:latest"

[deploy]
restartPolicyType = "on_failure"
restartPolicyMaxRetries = 3
# /app/.sessions é onde GOWS persiste dados do WhatsApp (evita re-escanear QR no restart)
volumes = [{ mount = "/app/.sessions", name = "waha-sessions" }]
```

Variáveis que o serviço Waha recebe (set via dashboard ou `railway variables --set`):
```
WHATSAPP_DEFAULT_ENGINE=GOWS
WAHA_PRINT_QR=false
WAHA_API_KEY=sha512:<HASH>               # gerar com: echo -n "secret" | shasum -a 512
WHATSAPP_API_KEY_EXCLUDE_PATH=ping,health
WAHA_DASHBOARD_ENABLED=true
WAHA_DASHBOARD_USERNAME=admin
WAHA_DASHBOARD_PASSWORD=<senha>
WHATSAPP_HOOK_URL=https://${{app.RAILWAY_PUBLIC_DOMAIN}}/api/webhooks/waha
WHATSAPP_HOOK_EVENTS=message,message.any,session.status
WHATSAPP_HOOK_HMAC_KEY=<secret compartilhado com o app>
WAHA_LOG_LEVEL=info
```

> **Limitações do Waha Core (free) — documentadas oficialmente:**
> - ✅ **1 única sessão** (a `default`). Nunca criar outra sessão.
> - ✅ Texto ilimitado. API Key auth (`X-Api-Key`) suportado.
> - ❌ **Sem mídia** (enviar image/audio/video requer Waha Plus). Se o Onda 3 pediu RAG com PDFs como resposta ao WhatsApp ou respostas em áudio, Claude deve avisar no SPEC que precisa Plus ($19/mo, imagem `devlikeapro/waha-plus`).
> - ❌ Volume "100+" no Onda 1 → idem, exige Plus (múltiplas sessões).
> - ✅ GOWS funciona em Core (sem browser, baixo consumo — ideal p/ Railway free).

### 4.4 `instrumentation.ts` — boot hook do Next.js

Next.js 16 executa `instrumentation.ts` na **inicialização do servidor** (antes de aceitar requests). Esse é o ponto certo para rodar `ensureSession()` de cada canal, garantindo que após todo deploy / redeploy / domain change o webhook URL e a config estejam sincronizados no provedor.

```ts
// apps/web/src/instrumentation.ts
export async function register() {
  // Next.js chama isto 1× por worker; proteger contra duplicação em dev
  if (process.env.NEXT_RUNTIME !== "nodejs") return;

  const { getEnabledAdapters } = await import("./lib/channels/registry");
  const adapters = getEnabledAdapters();  // lê setup.config.json + envs

  for (const adapter of adapters) {
    try {
      const state = await adapter.ensureSession({});
      console.log(JSON.stringify({
        event: "channel.ensured",
        channel: adapter.id,
        state,
        at: new Date().toISOString(),
      }));
    } catch (err) {
      // NUNCA derrubar o app por isso — só logar. Operador conserta via /onboarding.
      console.error(JSON.stringify({
        event: "channel.ensure_failed",
        channel: adapter.id,
        error: String(err),
        at: new Date().toISOString(),
      }));
    }
  }
}
```

`next.config.ts` deve ter `experimental.instrumentationHook = true` (Next ≤15) ou nada em Next 16+ (já built-in).

---

## 5. ORCHESTRATOR — contratos mínimos

### 5.1 Channel Adapter (normaliza todos os canais para 1 interface)

`apps/web/src/lib/channels/types.ts`:
```ts
export type ChannelId = "waha" | "evolution" | "whatsapp_cloud" | "instagram";

export interface ChannelMessage {
  channel: ChannelId;
  contactId: string;         // ex: "5511999999999@c.us" (Waha) | "17034321234" (Cloud API) | "ig_user_id" (Instagram)
  contactName?: string;
  text: string;              // texto normalizado (mídia pode ir para media[])
  media?: { type: "image" | "audio" | "video" | "document"; url?: string; base64?: string }[];
  receivedAt: string;        // ISO
  raw: unknown;              // payload original do provedor (p/ debug)
}

export interface ChannelAdapter {
  id: ChannelId;
  parseWebhook(req: Request): Promise<ChannelMessage | null>;  // null se evento não é msg
  verifyWebhook?(req: Request): Promise<Response>;              // só Cloud API/Instagram (GET challenge)
  sendText(contactId: string, text: string): Promise<void>;
  sendMedia?(contactId: string, media: { type: string; url: string }): Promise<void>;
}
```
Cada canal em `apps/web/src/lib/channels/<id>.ts` implementa `ChannelAdapter`. A rota `/api/webhooks/<id>/route.ts` chama `parseWebhook` → `enqueueForDebounce`.

### 5.2 Debounce com Redis (`apps/web/src/lib/debounce.ts`)

**Problema**: usuário manda "oi", "tudo bem?", "preciso de ajuda" em <5s. Sem debounce, o LLM responde 3 vezes (3× custo, UX ruim, risco de race condition). Com debounce, as 3 msgs são coalesced em um único turn do agente.

**Contrato**:
```ts
import { redis } from "@/lib/redis";

export async function enqueueForDebounce(msg: ChannelMessage): Promise<void> {
  const key  = `${process.env.DEBOUNCE_KEY_PREFIX ?? "debounce"}:${msg.channel}:${msg.contactId}`;
  const lock = `${key}:lock`;
  const windowMs = Number(process.env.DEBOUNCE_WINDOW_MS ?? 5000);
  const maxBuf   = Number(process.env.DEBOUNCE_MAX_BUFFER ?? 20);

  // 1. Append (RPUSH) + trim p/ evitar spam
  await redis.multi()
    .rpush(key, JSON.stringify(msg))
    .ltrim(key, -maxBuf, -1)
    .pexpire(key, windowMs * 4)   // TTL de segurança (msg nunca fica órfã)
    .exec();

  // 2. Sliding window: cada msg RESETA o timer (SET com PX = windowMs)
  //    Só uma msg por contato ganha o flush — a última.
  const token = crypto.randomUUID();
  await redis.set(lock, token, "PX", windowMs);

  // 3. Agendar flush — setTimeout em Node.js standalone do Next,
  //    protegido por "compare-and-delete" do lock token
  setTimeout(async () => {
    const current = await redis.get(lock);
    if (current !== token) return;        // outra msg chegou → este flush é cancelado
    await redis.del(lock);

    const buffered = await redis.lrange(key, 0, -1);
    await redis.del(key);
    if (buffered.length === 0) return;

    const messages = buffered.map((s) => JSON.parse(s) as ChannelMessage);
    await flushToLLM(messages);           // chama /api/chat internamente
  }, windowMs + 50);
}
```

**Invariantes críticas**:
- **Sliding window por `(channel, contactId)`**: cada nova msg reseta o timer. Só flusha após `windowMs` de silêncio.
- **Compare-and-delete token**: evita que dois flushes concorrentes processem o mesmo buffer (cenário: 2 workers Railway).
- **Buffer cap** (`maxBuf=20`): defesa contra spam — mais que 20 msgs em <5s vira truncagem.
- **TTL de 4× windowMs na chave de buffer**: se o processo morrer antes do flush, Redis limpa sozinho.
- **Fallback se `cache.debounce.enabled == false`**: chamar `flushToLLM([msg])` imediatamente, sem Redis. Código gera os dois caminhos a partir do config.

### 5.3 LLM Orchestrator

`apps/web/src/lib/llm/index.ts`:
```ts
export interface LLMProvider {
  name: "openai" | "anthropic" | "google" | "groq" | "openrouter";
  chat(args: {
    system: string;
    messages: ChatMessage[];
    tools?: ToolSpec[];
    model: string;
  }): Promise<{ text: string; toolCalls?: ToolCall[] }>;
}
```
- `flushToLLM(messages: ChannelMessage[])` agrupa as msgs em um único user-turn: `messages.map(m => m.text).join("\n")` (ou estratégia mais rica se houver mídia).
- Anthropic: usar prompt caching (ver skill `claude-api`).
- Fallback automático segundo escolha do Onda 2.
- Timeout 30s por provedor. Se estourar → próxima na cadeia.
- Resposta volta pelo `ChannelAdapter.sendText` do canal de origem (`messages[0].channel`).

### 5.4 Session lifecycle — **padrão idempotente obrigatório por canal**

> 🔴 **Bug conhecido**: `POST /api/sessions` no Waha retorna **422 `Session 'default' already exists. Use PUT to update it.`** se a sessão já existir. **Nunca** criar sem checar primeiro. Aplica-se a todos os canais abaixo.

Cada `ChannelAdapter` (§5.1) **deve** expor `ensureSession()` que segue este contrato:

```ts
// apps/web/src/lib/channels/types.ts (adicionar à interface)
export interface ChannelAdapter {
  id: ChannelId;
  // ... (existentes)
  ensureSession(cfg: SessionConfig): Promise<SessionState>;  // idempotente
  getSessionStatus(): Promise<SessionState>;
  startSession?(): Promise<void>;                            // só quando aplicável
  stopSession?(): Promise<void>;
  getQrCode?(): Promise<string | null>;                      // base64, null se já WORKING
}

export type SessionState =
  | "NOT_CREATED" | "STOPPED" | "STARTING" | "SCAN_QR" | "WORKING" | "FAILED";
```

**Contrato universal de `ensureSession()`** (implementado por cada canal):
1. `GET` o recurso de sessão/instância.
2. Se **404/não existe** → `POST` para criar.
3. Se **existe** + config difere → `PUT` / `PATCH` para atualizar.
4. Se **existe** + config igual → **no-op** (retornar status atual).
5. Se `state ∈ {NOT_CREATED, STOPPED, FAILED}` → chamar start.
6. Se `state == WORKING` → no-op.
7. Se `state ∈ {STARTING, SCAN_QR}` → retornar status (usuário escaneia QR).

#### 5.4.1 Waha (Core + Plus) — ciclo de vida

**Endpoints oficiais** (confirmar via Context7 `/devlikeapro/waha-docs`):
| Ação | Método | Endpoint | Notas |
|---|---|---|---|
| Listar | GET | `/api/sessions` | Retorna array |
| Buscar | GET | `/api/sessions/{name}` | 404 se não existir |
| Criar | POST | `/api/sessions` | **422 se já existir** |
| Atualizar | PUT | `/api/sessions/{name}` | Só em sessão existente |
| Iniciar | POST | `/api/sessions/{name}/start` | Move STOPPED→STARTING→SCAN_QR→WORKING |
| Parar | POST | `/api/sessions/{name}/stop` | Mantém config, libera engine |
| Logout | POST | `/api/sessions/{name}/logout` | Desloga WhatsApp mas preserva session |
| Deletar | DELETE | `/api/sessions/{name}` | Remove config e volume |
| QR | GET | `/api/{name}/auth/qr?format=image` | Só quando state=SCAN_QR |

**Estados Waha**: `STOPPED`, `STARTING`, `SCAN_QR_CODE`, `WORKING`, `FAILED`.

**Dicionário de erros 422 do Waha** (regex sobre `response.body.message`) — **toda chamada de estado DEVE classificar o 422, não propagar:**

| Regex na mensagem | Ação | Significado |
|---|---|---|
| `Session '.+' already exists\. Use PUT to update` | fazer PUT em vez de POST | sessão já existe (criação duplicada) |
| `Session '.+' is already started` | tratar como sucesso, releitura via GET | sessão rodando (start duplicado) |
| `Session '.+' is not started` | tratar como no-op p/ stop/logout; erro p/ send | sessão parada |
| `Session '.+' status is STARTING` | polling — não retry imediato | em transição |
| `Session '.+' status is SCAN_QR_CODE` | retornar QR, não erro | aguardando QR |
| `Session '.+' status is FAILED` | GET detalhe, reportar `error` do Waha | engine crashou |
| qualquer outro | propagar como erro real | bug genuíno |

**Implementação de referência (`lib/channels/waha.ts`) — com 422 dictionary + reconciliação:**
```ts
// Classifica a resposta 422 do Waha. Retorna "ignore" se for benigno, "error" se real.
async function classifyWaha422(res: Response): Promise<"ignore" | "error" | "put"> {
  if (res.status !== 422) return res.ok ? "ignore" : "error";
  const body = await res.clone().json().catch(() => ({}));
  const msg = String(body?.message ?? "");
  if (/already exists\. Use PUT to update/i.test(msg)) return "put";
  if (/is already started/i.test(msg))                 return "ignore";
  if (/is not started/i.test(msg))                     return "ignore";
  if (/status is (STARTING|SCAN_QR_CODE|WORKING)/i.test(msg)) return "ignore";
  return "error";
}

async function safePost(base: string, path: string, headers: HeadersInit, body?: unknown) {
  const r = await fetch(`${base}${path}`, {
    method: "POST", headers,
    body: body ? JSON.stringify(body) : undefined,
  });
  const verdict = await classifyWaha422(r);
  if (verdict === "error") throw new Error(`Waha ${path} failed: ${r.status} ${await r.text()}`);
  return { response: r, verdict };
}

export async function ensureWahaSession(cfg: WahaConfig): Promise<SessionState> {
  const name = cfg.session ?? "default";
  const base = process.env.WAHA_BASE_URL!;
  const headers = { "X-Api-Key": process.env.WAHA_API_KEY!, "content-type": "application/json" };

  // 1. SEMPRE GET primeiro — fonte da verdade
  const existing = await fetch(`${base}/api/sessions/${name}`, { headers });

  const desiredBody = {
    name,
    config: {
      webhooks: [{
        url: `${process.env.NEXT_PUBLIC_APP_URL}/api/webhooks/waha`,
        events: ["message", "message.any", "session.status"],
        hmac: { key: process.env.WAHA_HMAC_KEY! },
      }],
    },
  };

  if (existing.status === 404) {
    // 2a. Não existe → POST (mas ainda pode colidir em race: trata como "put")
    const { verdict } = await safePost(base, `/api/sessions`, headers, desiredBody);
    if (verdict === "put") {
      await fetch(`${base}/api/sessions/${name}`, {
        method: "PUT", headers,
        body: JSON.stringify({ config: desiredBody.config }),
      });
    }
  } else if (existing.ok) {
    // 2b. Existe → PUT idempotente (garante webhook correto após redeploy)
    await fetch(`${base}/api/sessions/${name}`, {
      method: "PUT", headers,
      body: JSON.stringify({ config: desiredBody.config }),
    });
  } else {
    throw new Error(`Waha GET failed: ${existing.status}`);
  }

  // 3. Consultar estado ANTES de chamar start
  const statusBefore = await getWahaStatus(name);

  // 4. Só chamar start se o estado REALMENTE permite
  const startable: SessionState[] = ["STOPPED", "FAILED", "NOT_CREATED"];
  if (startable.includes(statusBefore)) {
    await safePost(base, `/api/sessions/${name}/start`, headers);
    // safePost já trata 422 "is already started" como ignore
  }

  // 5. Reconciliação: sempre retornar o GET pós-operação (não confiar em POST response)
  return getWahaStatus(name);
}
```

**Princípios de reconciliação (aplicam a TODOS os canais):**
1. **GET é fonte da verdade** — decisões de estado vêm de `getSessionStatus()`, nunca da resposta de um POST.
2. **UI nunca mostra `FAILED` baseado em erro de POST** — `FAILED` só se `getSessionStatus()` retornar `FAILED` (= engine Waha crashou). Se POST falhar com 422 benigno, re-consultar GET e refletir o estado real.
3. **Logs estruturados incluem o verdict** — `{event:"session.start", verdict:"ignore", reason:"already_started"}` em vez de `error`.
4. **Retry budget ≠ erro dictionary** — o dicionário elimina tentativas desnecessárias; retry é p/ erros transitórios reais (5xx, timeout).

#### 5.4.2 Evolution API — ciclo de vida

**Endpoints** (confirmar via Context7 `/evolutionapi/evolution-api`):
| Ação | Método | Endpoint | Notas |
|---|---|---|---|
| Listar | GET | `/instance/fetchInstances` | `?instanceName=` filtra |
| Criar | POST | `/instance/create` | **403/409 se já existir** |
| Estado | GET | `/instance/connectionState/{name}` | `open` \| `close` \| `connecting` |
| Conectar | GET | `/instance/connect/{name}` | Retorna QR (base64) |
| Desconectar | DELETE | `/instance/logout/{name}` | Preserva instância |
| Deletar | DELETE | `/instance/delete/{name}` | Remove tudo |
| Restart | POST | `/instance/restart/{name}` | |

**Estados Evolution**: `open` (= WORKING), `close` (= STOPPED/NOT_CREATED), `connecting` (= STARTING/SCAN_QR).

**Pattern:**
```ts
export async function ensureEvolutionInstance(cfg: EvoConfig): Promise<SessionState> {
  const name = cfg.instanceName ?? "default";
  const base = process.env.EVOLUTION_BASE_URL!;
  const headers = { "apikey": process.env.EVOLUTION_API_KEY!, "content-type": "application/json" };

  const list = await (await fetch(`${base}/instance/fetchInstances?instanceName=${name}`, { headers })).json();
  const exists = Array.isArray(list) && list.length > 0;

  if (!exists) {
    const r = await fetch(`${base}/instance/create`, {
      method: "POST", headers,
      body: JSON.stringify({
        instanceName: name,
        qrcode: true,
        integration: "WHATSAPP-BAILEYS",
        webhook: {
          url: `${process.env.NEXT_PUBLIC_APP_URL}/api/webhooks/evolution`,
          events: ["MESSAGES_UPSERT"],
          byEvents: false,
        },
      }),
    });
    if (!r.ok && r.status !== 403 && r.status !== 409) {
      throw new Error(`Evolution create failed: ${r.status} ${await r.text()}`);
    }
  }
  // Evolution não tem PUT — para atualizar webhook, usar POST /webhook/set/{name}
  await fetch(`${base}/webhook/set/${name}`, {
    method: "POST", headers,
    body: JSON.stringify({
      url: `${process.env.NEXT_PUBLIC_APP_URL}/api/webhooks/evolution`,
      enabled: true,
      events: ["MESSAGES_UPSERT"],
    }),
  });

  const state = (await (await fetch(`${base}/instance/connectionState/${name}`, { headers })).json())?.instance?.state;
  if (state === "close") await fetch(`${base}/instance/connect/${name}`, { headers });  // dispara QR
  return mapEvoStateToSessionState(state);
}
```

#### 5.4.3 WhatsApp Cloud API (Meta) — sem "sessão", mas tem **subscription**

Cloud API não tem QR nem sessão stateful — a "sessão" é o `Phone Number ID` + a subscription do app ao webhook. Pattern idempotente:

| Ação | Método | Endpoint | Notas |
|---|---|---|---|
| Listar subs do phone | GET | `/v21.0/{phone-id}/subscribed_apps` | |
| Subscrever app | POST | `/v21.0/{phone-id}/subscribed_apps` | **200 mesmo se já subscribed** (mas confirmar via GET) |
| Unsubscribe | DELETE | `/v21.0/{phone-id}/subscribed_apps` | |
| Info do número | GET | `/v21.0/{phone-id}` | Confirma `verified_name`, `quality_rating` |

```ts
export async function ensureCloudApiSubscription(cfg: CloudCfg): Promise<SessionState> {
  const phoneId = process.env.META_PHONE_NUMBER_ID!;
  const token   = process.env.META_SYSTEM_USER_TOKEN!;
  const headers = { "Authorization": `Bearer ${token}`, "content-type": "application/json" };

  // 1. Info do número (valida credenciais)
  const info = await fetch(`https://graph.facebook.com/v21.0/${phoneId}`, { headers });
  if (!info.ok) return "FAILED";

  // 2. Lista subs
  const subs = await (await fetch(`https://graph.facebook.com/v21.0/${phoneId}/subscribed_apps`, { headers })).json();
  const alreadySubscribed = subs?.data?.some((s: any) => s?.whatsapp_business_api_data?.id === process.env.META_APP_ID);

  if (!alreadySubscribed) {
    // 3. POST é idempotente na Meta — mas verificamos antes p/ evitar side-effects em rate-limit
    const r = await fetch(`https://graph.facebook.com/v21.0/${phoneId}/subscribed_apps`, {
      method: "POST", headers,
    });
    if (!r.ok) return "FAILED";
  }
  return "WORKING";  // Cloud API não tem intermediários; ou está subscrito ou não
}
```

**Observação crítica**: o **webhook verify token** (`hub.verify_token`) é configurado no Meta App Dashboard — só o app que **responde** o challenge. Não há endpoint para registrar programaticamente.

#### 5.4.4 Instagram Direct (Meta) — subscription de Page

Análogo ao Cloud API, mas no recurso Page:

| Ação | Método | Endpoint | Notas |
|---|---|---|---|
| Listar subs | GET | `/v21.0/{page-id}/subscribed_apps` | |
| Subscrever | POST | `/v21.0/{page-id}/subscribed_apps?subscribed_fields=messages,messaging_postbacks` | |
| Unsub | DELETE | `/v21.0/{page-id}/subscribed_apps` | |

Mesmo padrão GET antes de POST. Token usado: `META_IG_PAGE_ACCESS_TOKEN`.

#### 5.4.5 Quando cada método é chamado

| Contexto | Método chamado |
|---|---|
| `railway up` do app (boot) | `ensureSession()` p/ cada canal em `channels.enabled` — no `instrumentation.ts` do Next.js |
| Client Wizard `/onboarding` passo 1 | `ensureSession()` + polling de `getSessionStatus()` até `WORKING` |
| Webhook recebendo msg | **não** chama `ensureSession` (assume WORKING) — se falhar, responde 200 e loga |
| Botão "Reconectar" no painel | `stopSession()` → `startSession()` (ou `ensureSession()` com flag force) |
| Botão "Logout" no painel | Waha: `logout` · Evolution: `logout` · Meta: DELETE subscription |

### 5.5 Agent Sessions + AI Activation Rules

> **Contexto**: a "sessão" aqui é **do agente IA** (uma persona/config reutilizável), não a sessão do canal (§5.4). Um app pode ter N `AgentSession` (ex: "SDR B2B", "Suporte técnico") e regras decidem qual aplica a cada conversa.

#### 5.5.1 AgentSession — config persistida por persona

Campos em `AgentSession` (ver schema §4.0):
- `name` — "SDR B2B", "Suporte Noturno"
- `systemPrompt` — prompt que define o comportamento
- `provider` + `model` — override por sessão (ex: SDR usa `claude-sonnet-4-6`, suporte usa `gemini-3-flash-preview`)
- `temperature`, `maxTokens` — knobs do LLM
- `memoryMode` — `window_N` / `summary_plus_N` / `stateless`
- `tools` — subset das tools habilitadas globalmente (`["calendar", "webSearch"]`)
- `isDefault` — fallback quando nenhuma regra matcha

A conversa (`Conversation`) aponta para **uma** `AgentSession` — ou via regra automática, ou via atribuição manual no painel. Trocar de sessão no meio de uma conversa é permitido e reflete na próxima msg.

#### 5.5.2 AI Activation Rules — quando o agente responde

**Problema**: responder 24/7 a tudo não é o que o cliente quer. Casos reais:
- "agente só responde fora do horário comercial"
- "agente só responde se o contato disser 'bot' ou '/ai'"
- "agente nunca responde pros 5 números do time interno"
- "agente responde só pros leads do CRM (whitelist)"
- "agente pausa se o humano mandar msg — até o humano liberar de novo"

**Modos** (campo `AIRule.mode`):

| Mode | Semântica | `params` shape | Ex. uso |
|---|---|---|---|
| `always_on` | sempre responde | `{}` | default MVP |
| `contact_whitelist` | responde só p/ contatos listados | `{contactIds: string[] \| tags: string[]}` | "só leads do CRM" |
| `contact_blacklist` | nunca responde p/ listados | `{contactIds: string[] \| tags: string[]}` | "não responder time interno" |
| `keyword_trigger` | responde só se msg contém keyword | `{keywords: string[], matchMode: "any"\|"all", caseSensitive: false}` | "só se contato disser 'bot'" |
| `keyword_pause` | **pausa** se msg contém keyword (usuário pede humano) | `{keywords: string[]}` | "'atendente', 'humano' → pausa e handoff" |
| `schedule` | ativo em janela de tempo | `{timezone, days: [1-7], start: "HH:MM", end: "HH:MM"}` | "só fora do horário comercial" |
| `manual_approval` | IA gera resposta mas atendente aprova antes de enviar | `{notifyVia: "panel" \| "slack"}` | modo supervisionado |
| `human_takeover` | se atendente mandou msg nas últimas N horas, IA cala | `{silenceHours: 24}` | auto-pausa pós-humano |

**Actions** (o que fazer quando a regra decide "não responder automaticamente"):
- `respond` — IA responde normalmente (modo default p/ rules positivas)
- `drop` — ignora silenciosamente
- `handoff` — dispara handoff (§ handoff destination do config)
- `static_reply` — envia texto fixo (`staticReply` field), ex: "Obrigado! Voltamos em horário comercial."

#### 5.5.3 Pipeline de avaliação

Implementado em `apps/web/src/lib/ai-rules/evaluate.ts`. Chamado **dentro do debounce flush**, antes de invocar o LLM:

```ts
import { prisma } from "@/lib/db";
import type { ChannelMessage } from "@/lib/channels/types";

export type Decision =
  | { action: "respond"; agentSession: AgentSession }
  | { action: "drop"; reason: string }
  | { action: "handoff"; reason: string }
  | { action: "static_reply"; text: string };

export async function decideAction(
  msgs: ChannelMessage[],
  conversation: Conversation,
): Promise<Decision> {
  // 1. Override local do atendente tem prioridade sobre tudo
  if (!conversation.aiEnabled) return { action: "drop", reason: "conversation.aiEnabled=false" };

  // 2. Determina AgentSession — da conversa, ou default
  const agentSession = conversation.agentSessionId
    ? await prisma.agentSession.findUnique({ where: { id: conversation.agentSessionId } })
    : await prisma.agentSession.findFirst({ where: { isDefault: true } });
  if (!agentSession) return { action: "drop", reason: "no_default_agent_session" };

  // 3. Rules ordenadas por priority ASC — primeira que decide "não respond" ganha
  const rules = await prisma.aIRule.findMany({
    where: { agentSessionId: agentSession.id, enabled: true },
    orderBy: { priority: "asc" },
  });

  for (const rule of rules) {
    const match = evaluateRule(rule, msgs, conversation);
    if (!match) continue;  // regra não se aplica, próxima
    // Regra casou: retorna a action dela (que pode ser "respond" ou negar)
    if (rule.action === "respond")       return { action: "respond", agentSession };
    if (rule.action === "drop")          return { action: "drop", reason: `rule:${rule.id}` };
    if (rule.action === "handoff")       return { action: "handoff", reason: `rule:${rule.id}` };
    if (rule.action === "static_reply")  return { action: "static_reply", text: rule.staticReply ?? "" };
  }

  // 4. Fallback: nenhuma regra casou → assume always_on implícito
  return { action: "respond", agentSession };
}
```

**Short-circuit importante**: regras de negação (`drop`/`handoff`/`static_reply`) só disparam **se o `evaluateRule` retornar true para aquela regra**. Ex: `contact_blacklist` com `contactIds=[X,Y]` só casa se o contato atual ∈ {X,Y}. Se não casar, próxima regra.

**`evaluateRule(rule, msgs, conversation)`** — puro, determinístico:
```ts
function evaluateRule(rule: AIRule, msgs: ChannelMessage[], conv: Conversation): boolean {
  switch (rule.mode) {
    case "always_on": return true;
    case "contact_whitelist":  return matchesContactList(rule.params, conv.contactId, "in");
    case "contact_blacklist":  return matchesContactList(rule.params, conv.contactId, "in");
    case "keyword_trigger":    return matchesKeywords(rule.params, msgs);
    case "keyword_pause":      return matchesKeywords(rule.params, msgs);
    case "schedule":           return insideSchedule(rule.params, new Date());
    case "manual_approval":    return true;  // ação muda: abre ticket no painel
    case "human_takeover":     return recentHumanMessage(conv.id, rule.params.silenceHours);
    default: return false;
  }
}
```

#### 5.5.4 Integração no pipeline de msg

Ordem de eventos (do webhook ao envio):
```
Webhook POST → HMAC verify → persist Message(inbound) → enqueueForDebounce(msg)
                                                               ↓
                                                  [windowMs de silêncio]
                                                               ↓
                                                     flushToLLM(msgs)
                                                               ↓
                                           ┌──── decideAction(msgs, conv) ────┐
                                           ↓           ↓            ↓         ↓
                                        respond      drop       handoff   static_reply
                                           ↓                       ↓         ↓
                                    LLM + tools              notify human  sendText
                                           ↓
                                  sendText via ChannelAdapter
                                           ↓
                                  persist Message(outbound) + UsageDaily
```

**Invariantes**:
- `decideAction` é **pura** (exceto leitura de DB) — dá pra cachear por `(conversationId, msgs_hash)` em Redis.
- `manual_approval` **não** chama LLM na hora — abre ticket no painel e espera aprovação humana. Ao aprovar, envia com `skipRules=true`.
- `drop` **não** desliga o debounce — buffer ainda é esvaziado p/ evitar vazamento.
- Toda decisão é **logada** com `{conversationId, ruleId, mode, decision}` p/ auditoria.

#### 5.5.5 UI de configuração (ver §6.3)

Páginas novas no painel:
- `/settings/agents` — CRUD de `AgentSession` (prompt, modelo, tools, temperature)
- `/settings/agents/[id]/rules` — builder visual de `AIRule[]` com drag-to-reorder (priority) + toggle enable
- `/conversations/[id]` — toggle inline "🤖 IA ativa" (grava `conversation.aiEnabled`) + dropdown para trocar `agentSession` mid-conversation

---

## 6. CLIENT WIZARD — onboarding dentro do app

Rota `/onboarding`, passos (usar `<Tabs>` verticais + `<Progress>` do shadcn, paleta Alien Green):

1. **Conectar canais** — renderizar 1 sub-passo por item em `channels.enabled`. Toda ação de conexão no backend **DEVE** chamar `ChannelAdapter.ensureSession()` (§5.4) — **nunca** REST cru. Endpoints do app:
   - `POST /api/sessions/{channel}/ensure` → chama `ensureSession()` do adapter, retorna `{ state, qrCode? }`. **Idempotente**: pode ser chamado N vezes.
   - `GET  /api/sessions/{channel}/status` → polling a cada 2s, chama `getSessionStatus()`. UI transiciona em `STARTING → SCAN_QR → WORKING`.
   - `POST /api/sessions/{channel}/logout` (opcional) → para troca de número.

   🔴 **Regras obrigatórias do botão "Iniciar sessão" / "Conectar":**
   - Botão **DEVE** disparar `POST /api/sessions/{channel}/ensure` — **nunca** um `/start` direto. `ensure` internamente decide se precisa POST/PUT/start baseado no estado atual.
   - Badge de estado ao lado do botão **DEVE** ler do `GET /.../status` (polling) — nunca do retorno do POST `ensure`. Se `ensure` der erro mas o GET mostra `WORKING`, a UI mostra `WORKING` (erro foi benigno, classificado via 422 dictionary).
   - Quando `state=WORKING`, o botão vira "Reconectar" (e só então chama `/logout` → `/ensure`).
   - Estado `FAILED` na UI **só** aparece quando o GET explicitamente retorna `FAILED`. Erro de POST nunca pinta UI como FAILED.
   - Toast de erro só renderiza se `ensure` retornar `verdict="error"` real; 422 benignos não viram toast.

   Por canal:
   - `waha_core` / `waha_plus` → `ensureSession` faz GET → POST/PUT → start. QR renderiza quando `state=SCAN_QR`.
   - `evolution_api` → `ensureSession` faz fetchInstances → create (se faltar) → webhook/set → connect. QR via state=connecting.
   - `whatsapp_cloud` → form com `Phone Number ID`, `Business Account ID`, `System User Token`, `App ID`. `ensureSession` valida credenciais e faz GET/POST `subscribed_apps` (idempotente). Sem QR — só confirma `state=WORKING`. Mostrar ao usuário a URL de webhook + `META_WEBHOOK_VERIFY_TOKEN` a colar no Meta App Dashboard (quem responde o challenge é o próprio app).
   - `instagram_direct` → OAuth Meta → selecionar Page → escolher IG Business Account conectada. `ensureSession` subscreve a Page com `subscribed_fields=messages,messaging_postbacks`.
2. **Escolher modelo** — dropdown com provedores habilitados no deploy + campo API key por provedor (stored em DB encrypted).
3. **System prompt** — textarea com sugestão pré-preenchida baseada no caso de uso (Onda 1). Botão "Testar" abre chat sandbox.
4. **Ativar tools** — lista de tools escolhidas no Onda 3, cada uma com form próprio:
   - Calendário → OAuth Google / URL Cal.com
   - Web search → campo Tavily API key
   - RAG → upload de PDFs (armazenar em volume ou S3)
   - Webhook → URL + headers JSON
5. **Debounce (se habilitado)** — slider p/ ajustar `windowMs` (3–15s) em tempo real. Preview: "Usuário envia 3 msgs em 4s → agente responde 1× em ~Xs."
6. **Horário + handoff** — seletor de timezone, janela, destino do handoff.
7. **Pronto** — botão "Enviar msg de teste" (usa o primeiro canal de `channels.enabled` p/ mandar `Olá! Agente online ✅` ao próprio número/conta do admin).

### 6.1 Painel runtime (pós-onboarding)

Além do wizard de onboarding linear, o app expõe páginas de configuração que o cliente usa **depois** do setup — todas seguindo convenção shadcn/Tabs com paleta Alien Green:

**`/settings/agents`** — CRUD de `AgentSession` (personas do agente)
- Lista de sessões com badge `isDefault` no default
- Criar/editar: nome · prompt · provedor LLM · model ID · temperatura · memory mode · tools ativas
- Botão "Testar" abre chat sandbox sem mandar mensagem real

**`/settings/agents/[id]/rules`** — regras de ativação da IA (ver §5.5 + `prompts/patterns/ai-rules.md`)
- Lista drag-to-reorder (priority) com toggle enable
- Modal "Nova regra": seletor de `mode` (8 opções) + form dinâmico que muda shape conforme `params`
- Validação inline: `contact_whitelist` aceita csv de contatos OU seletor de tags; `keyword_trigger` aceita keywords + match mode; `schedule` aceita timezone + days + start/end
- Preview "Se a msg X chegar no contato Y, agente responderá?" com simulação das regras

**`/conversations/[id]`** — timeline da conversa + controles inline
- Toggle `🤖 IA ativa` grava `conversation.aiEnabled` (override manual sobrepondo rules)
- Dropdown "Trocar agente" muda `conversation.agentSessionId` mid-conversation
- Badge de custo (`$0.032 gastos hoje`) linkado à tabela `UsageDaily`

**`/admin/dlq`** (se erros ocorreram) — itens da Dead Letter Queue com botões Retry/Drop

**`/admin/usage`** — dashboard de `UsageDaily`: tokens, custo, top-5 contatos mais caros

Cada passo só libera o próximo quando válido. Estado persistido em `onboarding_state` no DB.

---

## 7. DEPLOY NA RAILWAY (idempotente, sem escaping bash)

> ⚠️ Private networking Railway = **IPv6**. Next.js standalone (`HOSTNAME=0.0.0.0`) e Waha GOWS já são dual-stack. Nunca usar `127.0.0.1`.

> **Estratégia**: escrever 2 arquivos `.env.app` e `.env.waha` (fora do repo, em `$REPO_PATH/.railway/`, gitignored) e usar `railway variables --set-from-file` OU `while read` para carregar em lote. Isso elimina o inferno de escaping `\${{...}}`.

### 7.1 Provisionar projeto (idempotente)
```bash
cd "$REPO_PATH"
PROJECT_NAME="$(jq -r .project.slug setup.config.json)"

# 1. Auth
if ! railway whoami >/dev/null 2>&1; then
  [[ -n "$RAILWAY_TOKEN" ]] || railway login
fi

# 2. Projeto: criar se não existir, linkar se já existe
if ! railway status >/dev/null 2>&1; then
  railway init --name "$PROJECT_NAME"
fi
railway link 2>/dev/null || true   # idempotente
```

### 7.2 Plugin Postgres (idempotente)
```bash
railway variables --service Postgres >/dev/null 2>&1 \
  || railway add -d postgres
```

### 7.2.1 Plugin Redis (condicional — só se `cache.debounce.enabled`)
```bash
DEBOUNCE_ON=$(jq -r '.cache.debounce.enabled // false' setup.config.json)
if [[ "$DEBOUNCE_ON" == "true" ]]; then
  railway variables --service Redis >/dev/null 2>&1 \
    || railway add -d redis
  echo "✅ Redis plugin provisionado — REDIS_URL injetado no serviço app"
else
  echo "ℹ️ Debounce desativado — Redis não provisionado"
fi
```
O plugin Redis da Railway exporta `REDIS_URL` (e também `REDIS_HOST`/`REDIS_PORT`/`REDIS_PASSWORD` separados). Usar sempre `REDIS_URL` no app via `${{Redis.REDIS_URL}}` em `.env.app`.

### 7.3 Gerar secrets + arquivos .env
Claude executa este script **uma vez**; ele grava `.railway/secrets.json` (local, gitignored) e os 2 `.env` files:

```bash
mkdir -p .railway
SECRETS_FILE=.railway/secrets.json

if [[ ! -f "$SECRETS_FILE" ]]; then
  PLAIN=$(openssl rand -hex 32)
  HASH="sha512:$(printf '%s' "$PLAIN" | shasum -a 512 | awk '{print $1}')"
  HMAC=$(openssl rand -hex 32)
  AUTH_SECRET=$(openssl rand -hex 32)
  DASHBOARD_PW=$(openssl rand -hex 16)
  jq -n --arg plain "$PLAIN" --arg hash "$HASH" --arg hmac "$HMAC" \
        --arg auth "$AUTH_SECRET" --arg dashpw "$DASHBOARD_PW" \
    '{waha_api_key_plain:$plain, waha_api_key_hash:$hash, waha_hmac:$hmac, auth_secret:$auth, dashboard_password:$dashpw}' \
    > "$SECRETS_FILE"
fi

# Ler de volta (p/ re-run idempotente)
PLAIN=$(jq -r .waha_api_key_plain  "$SECRETS_FILE")
HASH=$(jq -r .waha_api_key_hash   "$SECRETS_FILE")
HMAC=$(jq -r .waha_hmac           "$SECRETS_FILE")
AUTH_SECRET=$(jq -r .auth_secret  "$SECRETS_FILE")
DASHPW=$(jq -r .dashboard_password "$SECRETS_FILE")

# --- .env.app (vars do serviço Next.js) ---
# Base: db + auth + URL pública
cat > .railway/.env.app <<EOF
DATABASE_URL=\${{Postgres.DATABASE_URL}}
NEXT_PUBLIC_APP_URL=https://\${{RAILWAY_PUBLIC_DOMAIN}}
AUTH_SECRET=$AUTH_SECRET
NODE_ENV=production
EOF

# Redis + debounce (condicional)
if [[ "$DEBOUNCE_ON" == "true" ]]; then
  WINDOW=$(jq -r '.cache.debounce.windowMs // 5000' setup.config.json)
  MAX_BUF=$(jq -r '.cache.debounce.maxBufferedMessages // 20' setup.config.json)
  PREFIX=$(jq -r '.cache.debounce.keyPrefix // "debounce"' setup.config.json)
  cat >> .railway/.env.app <<EOF
REDIS_URL=\${{Redis.REDIS_URL}}
DEBOUNCE_ENABLED=true
DEBOUNCE_WINDOW_MS=$WINDOW
DEBOUNCE_MAX_BUFFER=$MAX_BUF
DEBOUNCE_KEY_PREFIX=$PREFIX
EOF
else
  echo "DEBOUNCE_ENABLED=false" >> .railway/.env.app
fi

# Canais habilitados — só injetar o que foi escolhido
WAHA_ON=$(jq -r '.channels.waha_core.enabled or .channels.waha_plus.enabled' setup.config.json)
if [[ "$WAHA_ON" == "true" ]]; then
  cat >> .railway/.env.app <<EOF
WAHA_BASE_URL=http://waha.railway.internal:3000
WAHA_API_KEY=$PLAIN
WAHA_HMAC_KEY=$HMAC
EOF
fi

EVO_ON=$(jq -r '.channels.evolution_api.enabled' setup.config.json)
if [[ "$EVO_ON" == "true" ]]; then
  EVO_INSTANCE=$(jq -r '.channels.evolution_api.instanceName' setup.config.json)
  EVO_KEY=$(openssl rand -hex 32)
  jq --arg k "$EVO_KEY" '.evolution_api_key = $k' .railway/secrets.json > .railway/secrets.json.tmp \
    && mv .railway/secrets.json.tmp .railway/secrets.json
  cat >> .railway/.env.app <<EOF
EVOLUTION_BASE_URL=http://evolution.railway.internal:8080
EVOLUTION_API_KEY=$EVO_KEY
EVOLUTION_INSTANCE=$EVO_INSTANCE
EOF
fi

CLOUD_ON=$(jq -r '.channels.whatsapp_cloud.enabled' setup.config.json)
if [[ "$CLOUD_ON" == "true" ]]; then
  # Credenciais Meta coletadas no §2.5 e gravadas em .railway/meta.env antes deste bloco
  [[ -f .railway/meta.env ]] && cat .railway/meta.env >> .railway/.env.app
fi

IG_ON=$(jq -r '.channels.instagram_direct.enabled' setup.config.json)
if [[ "$IG_ON" == "true" ]]; then
  [[ -f .railway/meta-ig.env ]] && cat .railway/meta-ig.env >> .railway/.env.app
fi

# Append LLM provider keys (lidos de setup.config.json + prompts do §2)
for KEY in ANTHROPIC_API_KEY OPENAI_API_KEY GOOGLE_API_KEY GROQ_API_KEY OPENROUTER_API_KEY; do
  VAL=$(jq -r --arg k "$KEY" '.llm.apiKeys[$k] // empty' setup.config.json)
  [[ -n "$VAL" && "$VAL" != "null" ]] && echo "$KEY=$VAL" >> .railway/.env.app
done

# --- .env.waha (vars do container Waha — só se Waha habilitado) ---
if [[ "$WAHA_ON" == "true" ]]; then
  cat > .railway/.env.waha <<EOF
WHATSAPP_DEFAULT_ENGINE=GOWS
WAHA_PRINT_QR=false
WAHA_API_KEY=$HASH
WHATSAPP_API_KEY_EXCLUDE_PATH=ping,health
WAHA_DASHBOARD_ENABLED=true
WAHA_DASHBOARD_USERNAME=admin
WAHA_DASHBOARD_PASSWORD=$DASHPW
WHATSAPP_HOOK_URL=https://\${{app.RAILWAY_PUBLIC_DOMAIN}}/api/webhooks/waha
WHATSAPP_HOOK_EVENTS=message,message.any,session.status
WHATSAPP_HOOK_HMAC_KEY=$HMAC
WAHA_LOG_LEVEL=info
EOF
fi

echo ".railway/" >> .gitignore
```

### 7.4 Serviço `app` (idempotente)
```bash
# Criar serviço só se não existir
railway service app 2>/dev/null || railway add --service app

# Linkar a pasta apps/web a este serviço
( cd apps/web && railway link --service app )

# Carregar vars em lote (evita escaping)
while IFS='=' read -r K V; do
  [[ -z "$K" || "$K" =~ ^# ]] && continue
  railway variables --service app --set "$K=$V"
done < .railway/.env.app

# Deploy
( cd apps/web && railway up -d --service app )

# Domínio público (idempotente)
railway domain --service app 2>/dev/null || true
```

### 7.5 Serviço `waha` (condicional — se Waha Core ou Plus)
```bash
if [[ "$WAHA_ON" == "true" ]]; then
  # Escolher imagem: Plus tem prioridade sobre Core
  PLUS=$(jq -r '.channels.waha_plus.enabled' setup.config.json)
  if [[ "$PLUS" == "true" ]]; then
    WAHA_IMAGE=$(jq -r '.channels.waha_plus.image' setup.config.json)
  else
    WAHA_IMAGE=$(jq -r '.channels.waha_core.image' setup.config.json)
  fi

  railway service waha 2>/dev/null || railway add --image "$WAHA_IMAGE" --service waha

  # Volume p/ persistência da sessão
  railway volume list --service waha 2>/dev/null | grep -q waha-sessions \
    || railway volume add --service waha --mount-path /app/.sessions

  # Vars em lote
  while IFS='=' read -r K V; do
    [[ -z "$K" || "$K" =~ ^# ]] && continue
    railway variables --service waha --set "$K=$V"
  done < .railway/.env.waha

  # Deploy (Railway puxa imagem automaticamente)
  railway up -d --service waha

  # NÃO expor publicamente — não rodar `railway domain --service waha`
fi
```

### 7.5.1 Serviço `evolution` (condicional — se Evolution API)
```bash
if [[ "$EVO_ON" == "true" ]]; then
  EVO_IMAGE=$(jq -r '.channels.evolution_api.image' setup.config.json)
  railway service evolution 2>/dev/null || railway add --image "$EVO_IMAGE" --service evolution

  # Volume para persistência das sessões Baileys
  railway volume list --service evolution 2>/dev/null | grep -q evolution-instances \
    || railway volume add --service evolution --mount-path /evolution/instances

  # Evolution também precisa de Postgres + Redis (reusa os plugins do app)
  cat > .railway/.env.evolution <<EOF
AUTHENTICATION_API_KEY=$(jq -r .evolution_api_key .railway/secrets.json)
SERVER_URL=http://evolution.railway.internal:8080
DATABASE_ENABLED=true
DATABASE_PROVIDER=postgresql
DATABASE_CONNECTION_URI=\${{Postgres.DATABASE_URL}}
CACHE_REDIS_ENABLED=true
CACHE_REDIS_URI=\${{Redis.REDIS_URL}}
WEBHOOK_GLOBAL_URL=https://\${{app.RAILWAY_PUBLIC_DOMAIN}}/api/webhooks/evolution
WEBHOOK_GLOBAL_ENABLED=true
WEBHOOK_EVENTS_MESSAGES_UPSERT=true
CONFIG_SESSION_PHONE_CLIENT=Chrome
LOG_LEVEL=INFO
EOF

  while IFS='=' read -r K V; do
    [[ -z "$K" || "$K" =~ ^# ]] && continue
    railway variables --service evolution --set "$K=$V"
  done < .railway/.env.evolution

  railway up -d --service evolution
  # NÃO expor publicamente
fi
```

### 7.5.2 Canais Meta (Cloud API + Instagram — sem container)
Cloud API e Instagram **não exigem container próprio** — são webhooks externos da Meta. O que o Claude faz:
1. Valida que `META_*` envs estão no `.env.app` (carregados no §7.3).
2. Exibe ao usuário a URL de webhook a cadastrar no painel Meta: `https://$APP_URL/api/webhooks/whatsapp-cloud` (e `/instagram`).
3. Exibe o `META_WEBHOOK_VERIFY_TOKEN` gerado no `secrets.json` — usuário cola no Meta App Dashboard.
4. Registro efetivo do webhook na Meta é feito pelo usuário no painel ou via `/onboarding` (passo 1) no Client Wizard.

### 7.6 Migrações DB
```bash
railway run --service app -- npx prisma migrate deploy
```

### 7.7 Healthcheck
- App: `GET /api/health` → `{ ok: true }` (rota criada no §4 em `apps/web/src/app/api/health/route.ts`)
- Postgres: `railway run --service app -- npx prisma db execute --stdin <<< 'SELECT 1'`
- Redis (se `DEBOUNCE_ON`): `railway run --service app -- node -e 'require("ioredis").createClient(process.env.REDIS_URL).ping().then(r=>{console.log(r);process.exit(0)})'` → esperar `PONG`
- Waha (se habilitado): `GET /api/server/version` com header `X-Api-Key: $PLAIN`
- Evolution (se habilitado): `GET /manager/fetchInstances` com header `apikey: $EVO_KEY`
- Cloud API / Instagram: verificar webhook registrado na Meta via `GET /v21.0/<phone-number-id>/webhooks` (Cloud) ou via UI do Meta App Dashboard.

Claude deve rodar `railway status` no final e reportar:
- URL pública do `app` (ex: `https://app-production.up.railway.app`)
- Para cada container criado: hostname interno (`<service>.railway.internal`, **sem** domínio público)
- Senha do dashboard Waha: `$DASHPW` (imprimir UMA vez, usuário deve guardar) — só se Waha habilitado
- Evolution API key: `$EVO_KEY` — só se Evolution habilitado
- Meta webhook verify token — só se Cloud API ou Instagram habilitado

### 7.8 Rollback (se qualquer passo falhar)
```bash
# Derrubar apenas os serviços criados nesta execução (preserva Postgres e Redis)
railway service app        && railway down --service app        --yes
railway service waha       && railway down --service waha       --yes 2>/dev/null
railway service evolution  && railway down --service evolution  --yes 2>/dev/null
# Se quiser zerar tudo:
# railway delete --yes  # remove o projeto inteiro (CUIDADO)
```
Claude **só executa rollback após perguntar ao usuário** (AskUserQuestion: "Falhou no §X. Rollback, retry, ou pausar?").

---

## 8. VERIFICAÇÃO FINAL (antes de dizer "pronto")

Rodar em ordem; só reportar sucesso se **todos** passarem:

```bash
# 1. App respondendo publicamente
curl -sf "$APP_URL/api/health" | grep -q '"ok":true'

# 2. Redis responde PING (se debounce ativo)
if [[ "$DEBOUNCE_ON" == "true" ]]; then
  railway run --service app -- \
    node -e 'const r=require("ioredis").default ? new (require("ioredis").default)(process.env.REDIS_URL) : new (require("ioredis"))(process.env.REDIS_URL); r.ping().then(v=>{console.log(v);process.exit(v==="PONG"?0:1)})' \
    | grep -q PONG
fi

# 3. Debounce end-to-end: simular 3 msgs no mesmo contato em <windowMs e confirmar 1 flush
if [[ "$DEBOUNCE_ON" == "true" ]]; then
  railway run --service app -- \
    curl -sf -X POST "http://localhost:3000/api/test/debounce-simulate" \
    -H "content-type: application/json" \
    -d '{"contactId":"test","channel":"waha","texts":["oi","tudo bem?","preciso de ajuda"]}' \
    | jq -e '.flushCount == 1 and .messagesCoalesced == 3'
fi

# 4. Waha (se habilitado) — acessível via private networking
if [[ "$WAHA_ON" == "true" ]]; then
  railway run --service app -- \
    curl -sf -H "X-Api-Key: $WAHA_API_KEY" \
    "$WAHA_BASE_URL/api/server/version" | grep -q '"engine":"GOWS"'

  # Sessão default com webhook apontando pro app
  railway run --service app -- \
    curl -sf -H "X-Api-Key: $WAHA_API_KEY" \
    "$WAHA_BASE_URL/api/sessions" | \
    jq -e '.[0].name == "default" and (.[0].config.webhooks[0].url | test("/api/webhooks/waha"))'

  # HMAC configurado (proteção do webhook)
  railway run --service app -- \
    curl -sf -H "X-Api-Key: $WAHA_API_KEY" \
    "$WAHA_BASE_URL/api/sessions" | \
    jq -e '.[0].config.webhooks[0].hmac.key | length > 0'
fi

# 5. Evolution (se habilitado) — instância existe + webhook global ativo
if [[ "$EVO_ON" == "true" ]]; then
  railway run --service app -- \
    curl -sf -H "apikey: $EVOLUTION_API_KEY" \
    "$EVOLUTION_BASE_URL/instance/fetchInstances" | jq -e 'length >= 1'
fi

# 6. Cloud API (se habilitado) — webhook verify token responde 200 ao GET Meta
if [[ "$CLOUD_ON" == "true" ]]; then
  curl -sf "$APP_URL/api/webhooks/whatsapp-cloud?hub.mode=subscribe&hub.verify_token=$META_WEBHOOK_VERIFY_TOKEN&hub.challenge=42" \
    | grep -q '^42$'
fi

# 7. Instagram (se habilitado) — idem
if [[ "$IG_ON" == "true" ]]; then
  curl -sf "$APP_URL/api/webhooks/instagram?hub.mode=subscribe&hub.verify_token=$META_WEBHOOK_VERIFY_TOKEN&hub.challenge=42" \
    | grep -q '^42$'
fi

# 8. IDEMPOTÊNCIA de ensureSession — chamar 3× e confirmar que nunca retorna 422
# (captura regressões dos bugs:
#   - "Session 'default' already exists. Use PUT to update it."
#   - "Session 'default' is already started.")
for CHANNEL in $(jq -r '.channels.enabled[]' setup.config.json); do
  # 3 chamadas sequenciais — cobre: criar, PUT sobre existente, start sobre já startado
  for N in 1 2 3; do
    RESP=$(curl -s -X POST "$APP_URL/api/sessions/$CHANNEL/ensure" -w "\n%{http_code}")
    CODE=$(echo "$RESP" | tail -1)
    BODY=$(echo "$RESP" | sed '$d')
    if [[ "$CODE" != "200" ]]; then
      echo "❌ $CHANNEL call #$N retornou $CODE: $BODY"; exit 1
    fi
    # Body não pode conter "is already" nem "already exists" como erro propagado
    if echo "$BODY" | jq -e '.error | test("already (started|exists)")' >/dev/null 2>&1; then
      echo "❌ $CHANNEL call #$N vazou 422 benigno como erro: $BODY"; exit 1
    fi
  done
  # Depois das 3 chamadas, GET status deve ser consistente
  STATE=$(curl -sf "$APP_URL/api/sessions/$CHANNEL/status" | jq -r .state)
  if [[ "$STATE" == "FAILED" ]]; then
    echo "❌ $CHANNEL ficou FAILED após ensure idempotente (provável propagação de 422)"; exit 1
  fi
done

# 9. Logs do boot mostram channel.ensured para cada canal habilitado
railway logs --service app 2>&1 | grep -q '"event":"channel.ensured"' \
  || { echo "❌ instrumentation.ts não rodou — ensureSession não chamado no boot"; exit 1; }

# 10. Painel carrega /onboarding sem erro no console
# (Claude usa mcp__chrome-devtools__navigate_page + list_console_messages)
```

Se falhar qualquer um → aplicar skill `superpowers:systematic-debugging` e **não** marcar como completo.

---

## 8.5 VALIDAÇÃO COM USUÁRIO REAL (obrigatório antes do §9)

> 🧑‍🔬 **Por que**: os checks do §8 validam contrato técnico (health, webhook verify, ensureSession idempotente). Mas só uma mensagem real do WhatsApp expõe bugs de **fluxo vivo** — race conditions no debounce, tokens/TTL errados, modelo LLM inexistente, sendText falhando com formato `@lid`, etc. Esses bugs são silenciosos nos mocks.

### 8.5.1 Roteiro literal que Claude DEVE seguir

Quando todos os checks do §8 passarem, Claude **NÃO** vai direto ao §9. Ao invés disso, imprime este bloco e **para pra aguardar**:

```
✅ Checks automáticos passaram (§8).

🧑 Preciso de você agora — validação humana em 3 passos:

1. Abra {APP_URL}/onboarding e escaneie o QR code pelo WhatsApp
   (Aparelho conectado → Vincular aparelho)
2. Do seu celular, mande UMA mensagem qualquer (ex: "oi") para o número conectado
3. Me responda aqui: "enviei" (não precisa esperar a IA responder)

Vou monitorar os logs por ~60s e rastrear o fluxo:
   webhook.waha.recv → debounce.scheduled → debounce.timer_fired →
   flush.start → flush.decided → flush.sending → flush.sent_ok

Se qualquer fase faltar no log, eu diagnostico e corrijo antes de fechar.
```

### 8.5.2 Enquanto espera, Claude arma um Monitor

Usar a ferramenta `Monitor` pra não fazer polling manual:
```bash
Monitor(
  description: "End-to-end message flow",
  timeout_ms: 180000,
  persistent: false,
  command: """
    until railway logs --service app --lines 20 2>/dev/null | \
          grep -E 'flush.sent_ok|flush.send_failed|debounce.lock_expired|debounce.flush_error|webhook.waha.error' | \
          head -1 | grep -q .; do sleep 5; done
    railway logs --service app --lines 30 2>/dev/null | \
      grep -E 'event=\"(webhook|debounce|flush)' | tail -10
  """
)
```

### 8.5.3 Quando o usuário responder "enviei"

Claude lê os logs (`railway logs --service app --lines 40 | grep event=`) e **classifica** o estado:

| Último evento no log | Diagnóstico | Ação |
|---|---|---|
| `flush.sent_ok` | ✅ sucesso total | ir pro §9 |
| `flush.send_failed` | sendText do canal falhou | ler o erro; provável formato de contactId ou auth Waha; corrigir adapter |
| `flush.decided action=drop` | AI Rule / aiEnabled bloqueou | verificar DB: `aiEnabled`, handoff, rules — reportar ao usuário |
| `flush.decided action=handoff` | agente detectou palavra-chave | comportamento correto se usuário pediu; caso contrário afrouxar keywords |
| `llm.exhausted` | 3 tentativas Gemini falharam | validar `GOOGLE_API_KEY` + modelo existe (§0.9.16) |
| `debounce.lock_expired` | TTL do lock < timer fire | aplicar fix §0.9.23 (TTL = `windowMs * 4`) |
| `debounce.flush_error` | erro genérico no callback | logar stack completa; provável Redis disconnect |
| `debounce.redis_init_failed` | Redis inacessível | verificar `REDIS_URL` + hostname interno (§0.9.26) |
| `webhook.waha.recv` mas nada depois | debounce não foi chamado | inspecionar exceção silenciosa entre recv e enqueueForDebounce |
| Nenhum `webhook.waha.recv` | Waha não disparou webhook | confirmar `WHATSAPP_HOOK_URL` na Waha + session WORKING + sem HMAC quebrando silencioso |

### 8.5.4 Se falhar: ciclo iterativo

Claude aplica o fix, faz `railway up -d --service app -c`, espera `SUCCESS` via `Monitor`, e pede **nova mensagem** do usuário:
```
Deploy novo no ar. Mande outra mensagem (qualquer texto) pra gente confirmar.
```

Só sair desse loop quando **uma mensagem inteira** passa do recv ao `flush.sent_ok` e o usuário confirma visualmente que recebeu a resposta no WhatsApp. Essa confirmação **dupla** (log + UI do usuário) é o gate real de "pronto pra produção".

### 8.5.5 Regra forte

Claude **NÃO PODE** marcar o setup como "✅ Pronto" e ir ao §9 sem o ciclo 8.5 completo. Os checks do §8 são necessários mas **não suficientes** — o histórico deste wizard mostra que bugs só aparecem com tráfego real (debounce TTL, ContactIdentity race, Waha port errado, `@lid` format, Prisma log noise, etc — todos §0.9).

---

## 9. ENTREGÁVEIS FINAIS

Ao término, Claude imprime:

```
✅ App publicado:         https://{app}.up.railway.app
✅ Painel de onboarding:  https://{app}.up.railway.app/onboarding
✅ Canais provisionados (renderizar 1 linha por item de channels.enabled):
   • Waha     (private): http://waha.railway.internal:3000       [Core | Plus]
   • Evolution (private): http://evolution.railway.internal:8080
   • Cloud API           : webhook https://{app}.../api/webhooks/whatsapp-cloud
   • Instagram           : webhook https://{app}.../api/webhooks/instagram
✅ Redis (debounce, se ativo): private (via REDIS_URL injetado)
✅ Repo local:            ./whatsapp-ai-agent
✅ SPEC.md commitado em:  ./whatsapp-ai-agent/SPEC.md

🔐 Guarde estes secrets (impressos 1× — também em .railway/secrets.json local):
   • Dashboard Waha senha:     <DASHPW>                  (se Waha habilitado)
   • Evolution API key:        <EVO_KEY>                 (se Evolution habilitado)
   • Meta webhook verify token: <META_WEBHOOK_VERIFY_TOKEN> (se Cloud/IG habilitados)

Próximos passos pro cliente:
1. Acessar /onboarding
2. Conectar cada canal (QR para Waha/Evolution; formulário Meta p/ Cloud/IG)
3. Configurar prompt + tools + debounce window
4. Enviar msg de teste
```

---

## 10. REGRAS DE EXECUÇÃO (para Claude)

### 10.1 State machine (obrigatório)
Manter `~/.claude-wizard/<project.slug>.state.json` com:
```json
{ "phase": "preflight|wizard|scaffold|railway|verify|done", "lastError": null, "repoPath": "...", "configPath": "..." }
```
- Ao iniciar, **sempre** checar se o arquivo existe. Se existir e `--resume` foi passado → ler e pular fases concluídas.
- Após cada bloco (§1, §2, §3, §4, §7, §8), atualizar `phase`.
- Em erro: gravar `lastError` e perguntar (AskUserQuestion): `Retry | Rollback | Pausar`.

### 10.2 Fluxo (NEVER/ALWAYS)
- **NUNCA** avançar de bloco sem checkpoint atualizado.
- **NUNCA** usar texto livre para decisões do §2 — usar `AskUserQuestion`. Exceção: Onda 0 (nome/path) e confirmação final do SPEC.
- **SEMPRE** ler `setup.config.json` como fonte de verdade. Nunca hardcodar.
- **SEMPRE** invocar `mcp__context7__*` antes de escrever código de qualquer lib do §11.

### 10.3 Canais (multi-provider)
- **NUNCA** criar segunda sessão no Waha Core (só `default`). Se usuário pediu multi-sessão → mover para Plus e atualizar `channels.waha_plus.enabled: true` no config.
- **NUNCA** habilitar `waha_core` e `waha_plus` ao mesmo tempo (Plus é superset de Core — escolher um).
- **NUNCA** pedir ao Waha Core p/ enviar mídia — só texto. Se precisa áudio/imagem → Plus, Evolution, ou Cloud API.
- **NUNCA** bind em `127.0.0.1` — sempre `0.0.0.0`/`::` (Railway private net = IPv6).
- **NUNCA** expor `waha` nem `evolution` publicamente (`railway domain` só no `app`).
- **NUNCA** usar a mesma chave Meta (`META_APP_SECRET`, tokens) em múltiplos projetos — gerar app Meta dedicado por deploy.
- **SEMPRE** hash `sha512:` no Waha (`WAHA_API_KEY`) e plaintext separado p/ header `X-Api-Key` do app.
- **SEMPRE** volume em `/app/.sessions` (Waha) e `/evolution/instances` (Evolution) — sem ele, QR/instância se perde no restart.
- **SEMPRE** normalizar eventos de todos os canais para `ChannelMessage` (ver §5.1) antes de entregar ao debounce/LLM.
- **SEMPRE** roteiar a resposta de volta pelo **mesmo** canal da msg original (`messages[0].channel`), nunca hardcodar canal de saída.
- **SEMPRE** implementar o GET `hub.verify_token` para Cloud API e Instagram (Meta exige handshake de verificação).

### 10.3.1 Session lifecycle (idempotência obrigatória — ver §5.4)
- 🔴 **NUNCA** `POST /api/sessions` no Waha (ou `/instance/create` no Evolution) sem um `GET` anterior. O 422 `Session 'default' already exists. Use PUT to update it.` é resultado direto dessa violação.
- 🔴 **NUNCA** `POST /api/sessions/{n}/start` sem antes checar `getSessionStatus()`. O 422 `Session 'default' is already started.` é resultado direto dessa violação. Só chamar start se `state ∈ {STOPPED, FAILED, NOT_CREATED}`.
- **SEMPRE** usar `ChannelAdapter.ensureSession()` (§5.1) — nunca chamar o REST cru do provedor em código de produto. Toda lógica check-then-create vive no adapter, não na route handler.
- **SEMPRE** classificar 422 via dicionário de erros (§5.4.1) — distinguir **benigno** (já no estado desejado → ignore) vs **real** (bug → propagar). Propagar 422 benigno como erro é o bug que pinta a UI de `FAILED` com a sessão rodando.
- **SEMPRE** reconciliar estado via `GET` após qualquer POST de mudança — **GET é fonte da verdade**. POST response pode mentir (422 falso-positivo); GET não.
- 🔴 **NUNCA** marcar sessão como `FAILED` na UI por causa de erro de POST. UI mostra `FAILED` **somente** se `getSessionStatus()` retornar `FAILED` (engine crashou de verdade).
- **SEMPRE** seguir o fluxo: `GET` → (se 404) `POST` com fallback p/ `PUT` em race / (se existe) `PUT`/`PATCH` → `start` condicional → **re-GET** para retornar estado.
- **SEMPRE** chamar `ensureSession()` no boot do app (`instrumentation.ts` do Next.js) para garantir consistência após redeploy. Refresca webhook URL que muda a cada deploy/domain change.
- **SEMPRE** chamar `ensureSession()` também no Client Wizard `/onboarding` passo 1 — **é idempotente**, pode rodar N vezes sem quebrar.
- **NUNCA** chamar `ensureSession()` no webhook handler (request path crítico); assume `WORKING` e loga erro se não estiver.
- **NUNCA** propagar erro de `ensureSession()` sem classificar: 404/422/409 são normalmente recuperáveis; 401/403 são credenciais inválidas; 5xx é infra.
- **SEMPRE** logar o estado retornado por `ensureSession()` em structured logs com `verdict` explícito: `{channel, sessionName, state, action, verdict: "created|updated|ignored|started"}`.

### 10.4 Segurança
- **NUNCA** commitar `.env`, `.railway/`, `secrets.json`. Adicionar ao `.gitignore` no §4.
- **SEMPRE** escrever keys via `railway variables --set` — nunca no código.
- **SEMPRE** gerar `AUTH_SECRET`, senha do dashboard Waha, `EVOLUTION_API_KEY` e `META_WEBHOOK_VERIFY_TOKEN` com `openssl rand -hex`.
- **SEMPRE** validar HMAC no webhook Waha e **X-Hub-Signature-256** nos webhooks Meta (Cloud API + Instagram).

### 10.5 Qualidade
- **SEMPRE** rodar `npm run lint` + `npm run typecheck` antes de `railway up`. Se falhar, aplicar skill `superpowers:systematic-debugging`.
- **SEMPRE** invocar skill `superpowers:verification-before-completion` e rodar §8 completo antes de §9.
- **SEMPRE** usar `TodoWrite` com 1 item por bloco (§1 → §8).

### 10.6 Redis / Debounce
- **SEMPRE** provisionar Redis (§7.2.1) se `cache.debounce.enabled == true`. Nunca assumir que existe — checar via `railway add -d redis` idempotente.
- **SEMPRE** usar chave `{keyPrefix}:{channel}:{contactId}` para isolar buffers — evita colisão entre canais e entre contatos.
- **SEMPRE** combinar `RPUSH` + `LTRIM` + `PEXPIRE` em um `MULTI` (transação atômica) ao bufferar msg.
- **SEMPRE** usar compare-and-delete token em `setTimeout` flush — protege contra race condition entre 2 workers Railway.
- **NUNCA** setar `windowMs > 15000` (Meta Cloud API expira a janela de 24h com inatividade; debounce muito longo perde a janela livre).
- **NUNCA** compartilhar buffer entre `contactId`s — cada contato tem seu próprio sliding window.
- **NUNCA** usar `redis.keys("*")` em produção — usar `SCAN` se precisar iterar.
- Se `cache.debounce.enabled == false` → `flushToLLM([msg])` imediatamente, sem chamar Redis. Código gera os dois caminhos a partir do env `DEBOUNCE_ENABLED`.

---

## 11. REFERÊNCIAS OFICIAIS (verificadas via Context7 em 2026-04-20)

Para dúvidas durante a execução, consultar estes IDs Context7:

| Tópico | Library ID | Uso |
|---|---|---|
| Waha docs (v2025+) | `/devlikeapro/waha-docs` | env vars, GOWS, webhooks, sessions, Core vs Plus |
| Evolution API | `/evolutionapi/evolution-api` | config, instâncias, webhook global, env vars, Redis/Postgres integration |
| Meta WhatsApp Cloud API | `/facebook/whatsapp-business-platform` (ou `context7:whatsapp-cloud-api`) | Graph API v21.0+, phone number id, templates, 24h window, X-Hub-Signature-256 |
| Meta Instagram Graph | `/facebook/instagram-graph-api` | DMs (instagram_manage_messages), IG Business Account, webhook subscriptions |
| Railway docs | `/railwayapp/docs` | CLI `add`/`link`/`up`, private networking, volumes, plugins (postgres/redis) |
| ioredis | `/redis/ioredis` | cliente Node.js para Redis — MULTI, SCAN, pipeline |
| Redis commands | `/redis/docs` | `SET PX`, `RPUSH`/`LTRIM`, `PEXPIRE`, `LRANGE`, `SCAN` |
| Next.js 16 | `/vercel/next.js` | App Router, standalone output, Dockerfile |
| shadcn/ui | `/shadcn-ui/ui` | estilo `nova`, components.json |
| Prisma | `/prisma/docs` | schema, `migrate deploy`, Postgres driver |
| Anthropic SDK | `/anthropics/anthropic-sdk-typescript` | prompt caching, streaming |

**Fatos cravados pela doc oficial (não alucinar):**

**Waha (confirmar via Context7 antes de escrever código):**
- Imagem Core/free: `devlikeapro/waha` · Plus: `devlikeapro/waha-plus`
- Porta default: `3000` · Endpoint versão: `GET /api/server/version`
- Header auth: `X-Api-Key` (chave pode ser `sha512:HASH`)
- Session única no Core: **`default`**
- Webhook envs globais: `WHATSAPP_HOOK_URL`, `WHATSAPP_HOOK_EVENTS`, `WHATSAPP_HOOK_HMAC_KEY`
- Eventos válidos: `message`, `message.any`, `state.change`, `session.status`, `*`

**Evolution API (confirmar via Context7 — APIs mudam):**
- Imagem: `atendai/evolution-api:latest` · Porta default: `8080`
- Header auth: `apikey: <AUTHENTICATION_API_KEY>`
- Endpoint instâncias: `GET /instance/fetchInstances`
- Webhook global via envs `WEBHOOK_GLOBAL_URL` + `WEBHOOK_GLOBAL_ENABLED=true`

**Meta WhatsApp Cloud API:**
- Webhook: 2 métodos — `GET` (verificação com `hub.verify_token` + `hub.challenge`) e `POST` (eventos)
- Header de integridade: `X-Hub-Signature-256` (HMAC SHA256 com `META_APP_SECRET` como chave)
- Envio de msg: `POST https://graph.facebook.com/v21.0/{PHONE_NUMBER_ID}/messages` com bearer `META_SYSTEM_USER_TOKEN`
- Janela 24h de sessão — fora dela, só templates aprovados

**Instagram Graph API:**
- Mesmo pattern da Cloud API (GET verify + POST events + X-Hub-Signature-256)
- Exige `instagram_manage_messages` permission → review Meta obrigatório
- IG Business Account deve estar conectada a uma Facebook Page

**Redis na Railway:**
- Plugin via `railway add -d redis` → expõe `REDIS_URL` (e vars separadas)
- Usar sempre `REDIS_URL` via referência `${{Redis.REDIS_URL}}` no app
- Persistência default: RDB snapshots — p/ jobs críticos considerar AOF (config do plugin)

**Railway CLI / private networking:**
- `<service>.railway.internal` sobre IPv6 — bind `0.0.0.0` ou `::`
- CLI: `railway add` (não `service create`), `railway up -d` (não `--detach`), `railway variables --set "K=V"`, `railway volume add --mount-path`

### 11.1 Design tokens canônicos (fallback se auto-cut não existir)
Usar EXATAMENTE estes valores em `apps/web/src/app/globals.css` quando `$REFERENCE_AUTO_CUT` não for encontrado:

```
style:        nova (shadcn)
baseColor:    zinc
radius:       0.375rem
font-sans:    Geist, Inter, system-ui
font-mono:    Geist Mono, JetBrains Mono
accent:       #adff2f  (Alien Green)
accent-dim:   #8bcc26
warning:      #f59e0b
error:        #ef4444
success:      #22c55e

Light root:   bg #fafafa · fg #09090b · card #ffffff · border #e4e4e7 · primary #6abf1a
Dark .dark:   bg #09090b · fg #fafafa · card #18181b · border #27272a · primary #adff2f
Dark glow:    0 0 20px rgba(173,255,47,0.25), 0 0 60px rgba(173,255,47,0.08)
```
Next.js 16 + Tailwind v4: usar `@theme inline { ... }` + `:root {}` + `.dark {}`. Nunca gerar paleta diferente sem autorização explícita.

### 11.2 Checklist final (o que Claude imprime em §9)
Antes de dizer "pronto", Claude confirma que **todos** estes artefatos existem e são válidos:

- [ ] `$REPO_PATH/setup.config.json` (parseável com `jq`, schema version 2+)
- [ ] `$REPO_PATH/SPEC.md` (renderizado, sem `{{placeholders}}`, lista canais habilitados)
- [ ] `$REPO_PATH/.gitignore` contém `.railway/` e `.env*`
- [ ] `$REPO_PATH/.railway/secrets.json` (local, gitignored — waha/evolution/meta keys)
- [ ] `~/.claude-wizard/<slug>.state.json` com `"phase": "done"`
- [ ] `railway status` mostra serviços esperados:
  - sempre: `Postgres`, `app`
  - se `cache.debounce.enabled`: `Redis`
  - se `channels.waha_*.enabled`: `waha`
  - se `channels.evolution_api.enabled`: `evolution`
- [ ] `curl $APP_URL/api/health` retorna `{"ok":true}`
- [ ] Para cada canal em `channels.enabled`:
  - Waha: `/api/sessions` mostra `default` com webhook `/api/webhooks/waha`
  - Evolution: `/instance/fetchInstances` retorna ≥1 instância
  - Cloud API / Instagram: `GET /api/webhooks/<channel>?hub.challenge=X` retorna `X`
- [ ] Se debounce ativo: Redis responde `PONG` + teste end-to-end coalesca 3 msgs em 1 flush
- [ ] QR code renderiza em `$APP_URL/onboarding` para cada canal QR-based

---

## 12. OBSERVABILITY & GUARDRAILS

### 12.1 Structured logging (obrigatório)

Todo evento crítico emite JSON em **uma linha** via `console.log(JSON.stringify(...))`:

```ts
// apps/web/src/lib/log.ts
export function logEvent(event: string, data: Record<string, unknown>) {
  console.log(JSON.stringify({
    event,
    at: new Date().toISOString(),
    ...data,
  }));
}
```

**Eventos obrigatórios:**
- `channel.ensured` / `channel.ensure_failed` (boot)
- `webhook.received` / `webhook.hmac_invalid` / `webhook.dropped`
- `debounce.enqueued` / `debounce.flushed`
- `ai.decision` (inclui `ruleId`, `action`, `reason`)
- `llm.call` / `llm.response` (inclui `provider`, `model`, `tokensIn/Out`, `costUsd`, `latencyMs`)
- `session.state_changed` (inclui `before`, `after`, `verdict`)
- `outbound.sent` / `outbound.failed`

Railway captura stdout automaticamente — busca via `railway logs | jq 'select(.event=="...")'`.

### 12.2 Rate limiting (defesa de webhook)

Protege o app de spam/loops via Redis token bucket por contato:

```ts
// apps/web/src/lib/rate-limit.ts
export async function checkRateLimit(key: string, maxPerMinute = 30): Promise<boolean> {
  const bucket = `rl:${key}`;
  const count = await redis.incr(bucket);
  if (count === 1) await redis.expire(bucket, 60);
  return count <= maxPerMinute;
}
```

Aplicar em webhook handler **após** HMAC verify:
```ts
if (!await checkRateLimit(`${channel}:${contactId}`)) {
  logEvent("webhook.rate_limited", { channel, contactId });
  return new Response("rate limited", { status: 429 });
}
```

### 12.3 Cost guardrails

Evita LLM cost explosion — limite por contato e por dia (tabela `UsageDaily` do Prisma §4.0):

```ts
// apps/web/src/lib/guardrails.ts
const HARD_LIMIT_USD_PER_CONTACT_PER_DAY = 1.0;
const HARD_LIMIT_USD_TOTAL_PER_DAY       = 50.0;

export async function checkCostBudget(contactId: string): Promise<"ok" | "contact_over" | "total_over"> {
  const today = startOfDay(new Date());
  const contact = await prisma.usageDaily.aggregate({
    where: { date: today, contactId }, _sum: { costUsd: true },
  });
  if ((contact._sum.costUsd?.toNumber() ?? 0) >= HARD_LIMIT_USD_PER_CONTACT_PER_DAY) return "contact_over";

  const total = await prisma.usageDaily.aggregate({
    where: { date: today }, _sum: { costUsd: true },
  });
  if ((total._sum.costUsd?.toNumber() ?? 0) >= HARD_LIMIT_USD_TOTAL_PER_DAY) return "total_over";

  return "ok";
}

export async function recordUsage(contactId: string, tokensIn: number, tokensOut: number, provider: string, model: string) {
  const costUsd = computeCost(provider, model, tokensIn, tokensOut);
  const today = startOfDay(new Date());
  await prisma.usageDaily.upsert({
    where: { date_contactId: { date: today, contactId } },
    create: { date: today, contactId, tokensIn, tokensOut, costUsd },
    update: { tokensIn: { increment: tokensIn }, tokensOut: { increment: tokensOut }, costUsd: { increment: costUsd } },
  });
}
```

Chamado **antes** de cada invocação LLM; se retornar `contact_over` ou `total_over`, decisão vira `static_reply` com msg educada + alerta no painel admin.

### 12.4 Retry + Dead Letter Queue

Webhook sempre responde **200** rapidamente — processamento pesado vai pra job via Redis:

```ts
// Se flushToLLM falhar, empurra pra DLQ p/ retry manual via painel
try { await flushToLLM(msgs); }
catch (err) {
  await redis.rpush("dlq:llm_flush", JSON.stringify({ msgs, err: String(err), at: Date.now() }));
  logEvent("llm.flush_failed_dlq", { err: String(err) });
}
```

Rota admin `/admin/dlq` lista itens + botão "Retry" + "Drop".

### 12.5 Health endpoints

- `GET /api/health` — `{ok:true}` simples (liveness Railway)
- `GET /api/ready` — checa DB + Redis + cada `channel.getSessionStatus()` (readiness — inclui dependencies)
- `GET /api/metrics` — gated por auth, retorna contadores agregados de `UsageDaily` + rate limit + DLQ size

### 12.6 Regras NEVER/ALWAYS

- **NUNCA** deixar webhook retornar 5xx em caminho crítico — responde 200, empurra erro pra DLQ
- **NUNCA** processar LLM sem `checkCostBudget` passar
- **NUNCA** logar HMAC keys, tokens Meta, API keys — redact em toda camada
- **NUNCA** aceitar webhook sem rate limit
- **SEMPRE** structured logs (JSON de 1 linha), nunca `console.log("mensagem ...")` solto
- **SEMPRE** inclui `tenantId`/`conversationId` em todo log p/ correlação
- **SEMPRE** recordUsage após call LLM (mesmo em erro, p/ tracking)

---

## 13. DYNAMIC CONTEXT INJECTION (arquitetura modular)

### 13.1 Por que modular

Monolito de 2000+ linhas no prompt degrada raciocínio: Claude processa contexto irrelevante, confunde fontes, aplica regras fora do escopo. **Solução**: sub-prompts que são carregados só quando a config pede.

**Ganhos mensuráveis:**
- Contexto médio cai ~60% (de ~15k tokens p/ ~6k tokens por sessão de scaffold)
- Menos alucinação de API (cada sub-prompt cita Context7 verificado)
- Onboarding de novos frameworks sem poluir o spec principal (basta criar novo arquivo em `prompts/**`)

### 13.2 Estrutura de diretório

```
PROMPT/
├── SETUP_WIZARD.md               # este arquivo — orquestrador
└── prompts/
    ├── _INDEX.md                  # matriz de injeção + regras de load
    ├── channels/
    │   ├── waha.md                ✅ verificado Context7
    │   ├── evolution.md           ✅ verificado Context7
    │   ├── whatsapp-cloud.md      📝 stub (verificar em 1ª build)
    │   └── instagram.md           📝 stub
    ├── llm/
    │   ├── anthropic.md           📝 stub
    │   ├── openai.md              📝 stub
    │   ├── google.md              ✅ verificado (gemini-3-flash-preview, gemini-embedding-2 multimodal)
    │   └── groq.md                📝 stub
    ├── infra/
    │   ├── railway.md             ✅ verificado
    │   ├── redis.md               📝 stub
    │   └── postgres.md            📝 stub
    └── patterns/
        ├── idempotent-sessions.md ✅ baseline
        ├── debounce.md            ✅
        ├── ai-rules.md            ✅
        ├── hmac-webhook.md        ✅
        ├── meta-graph-subscription.md 📝 stub
        └── rag-pgvector.md        📝 stub
```

### 13.3 Template obrigatório de sub-prompt

Todo arquivo em `prompts/**` começa com frontmatter YAML:

```markdown
---
id: <kebab-case>
loadWhen: <expressão JS sobre cfg>   # ex: "cfg.channels.waha_core?.enabled"
context7:                             # sources validadas
  - /devlikeapro/waha-docs
tokensEstimate: ~1500
verifiedAt: 2026-04-21                # data da última validação Context7 (ou null se stub)
---
```

Depois, sempre as seções: **Quando carregar** → **O que garante** → **Endpoints canônicos** → **Business-rule failures conhecidos** (tabela) → **Contrato obrigatório** (código) → **Checks** → **Armadilhas**.

### 13.4 Algoritmo de injeção

Após §3 (SPEC confirmado) e antes de §4 (Scaffold), Claude executa:

```ts
function determineSubprompts(cfg: SetupConfig): string[] {
  const load: string[] = [
    // Baseline — sempre carregados
    "patterns/idempotent-sessions.md",
    "patterns/hmac-webhook.md",
    "patterns/ai-rules.md",
    "infra/railway.md",
  ];

  // Infra condicional
  if (cfg.cache?.debounce?.enabled) {
    load.push("patterns/debounce.md", "infra/redis.md");
  }
  if (cfg.infra.db === "postgres_railway") {
    load.push("infra/postgres.md");
  }

  // Canais
  if (cfg.channels.waha_core?.enabled || cfg.channels.waha_plus?.enabled) load.push("channels/waha.md");
  if (cfg.channels.evolution_api?.enabled) load.push("channels/evolution.md");
  if (cfg.channels.whatsapp_cloud?.enabled) load.push("channels/whatsapp-cloud.md", "patterns/meta-graph-subscription.md");
  if (cfg.channels.instagram_direct?.enabled) load.push("channels/instagram.md", "patterns/meta-graph-subscription.md");

  // LLM providers
  for (const p of cfg.llm.providers ?? []) load.push(`llm/${p}.md`);

  // RAG
  if (cfg.tools?.rag) load.push("patterns/rag-pgvector.md");

  return [...new Set(load)];
}
```

Claude então:
1. Imprime **1 linha** anunciando: `📦 Context loaded: 7 sub-prompts [channels/waha, llm/google, patterns/debounce, ...]`
2. Lê cada arquivo via Read tool (relativo ao repo do prompt)
3. Aplica as regras de cada um como se fossem extensão deste prompt

### 13.5 Regras de manutenção dos sub-prompts

- **Verificação Context7**: todo sub-prompt com `verifiedAt: null` **DEVE** ser validado via Context7 no primeiro build que o usar. Ao validar, atualizar `verifiedAt` com a data.
- **Reutilização**: se dois canais compartilham padrão (ex: Meta Cloud + Instagram compartilham subscription), extrair pra `patterns/<nome>.md` e incluir em ambos os `loadWhen`.
- **Deprecação**: quando um framework muda breaking API, não editar silenciosamente — criar `channels/<nome>-v2.md` e atualizar `loadWhen` p/ rotear por config.
- **Token budget**: se um sub-prompt ultrapassa 3k tokens, quebrar em 2 com carga condicional (ex: `channels/waha-core.md` + `channels/waha-plus.md` se a diferença crescer).

### 13.6 Fail-open, não fail-silent

Se um sub-prompt listado no `loadWhen` não existe:
- Claude **NÃO** continua silenciosamente — imprime erro claro: `❌ Sub-prompt prompts/channels/X.md não encontrado mas cfg exige. Abortando.`
- Usuário pode criar o stub com frontmatter + TODO e rodar `--resume`.

Se um sub-prompt tem `verifiedAt: null` e estamos em produção:
- Warning no início do scaffold: `⚠️ Sub-prompts não-verificados: [X, Y]. Primeira build deve validar via Context7.`

### 13.7 Anti-padrão explícito

- **NUNCA** carregar todos os sub-prompts "por segurança" — anula o ganho
- **NUNCA** duplicar conteúdo entre sub-prompts — usar `patterns/` pra cross-cutting
- **NUNCA** deixar `loadWhen` ambíguo (ex: `"cfg.channels"` genérico) — sempre condição precisa
- **NUNCA** citar um sub-prompt no main spec sem garantir que existe — usar `_INDEX.md` como fonte da verdade
