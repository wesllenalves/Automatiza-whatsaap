# 📚 Sub-prompts Index — Dynamic Context Injection

> **Para Claude**: este diretório contém sub-prompts modulares. O wizard principal (`SETUP_WIZARD.md`) define QUANDO carregar cada um. **Nunca** carregue tudo — isso estoura contexto e enfraquece o raciocínio. Carregue só o que `setup.config.json` pede.

## 🧭 Matriz de injeção (dinâmica)

Após o Dev Wizard (§2 do main) gravar `setup.config.json`, Claude calcula a lista abaixo e **apenas** lê esses arquivos via `@prompts/<path>`:

| Condição em `setup.config.json` | Sub-prompt a carregar | Por quê |
|---|---|---|
| **sempre** (baseline) | `patterns/idempotent-sessions.md` | toda integração externa precisa do padrão GET→POST/PUT→re-GET |
| **sempre** (baseline) | `patterns/hmac-webhook.md` | validar webhooks de qualquer canal |
| **sempre** (baseline) | `infra/railway.md` | CLI + plugins + private networking |
| `cache.debounce.enabled == true` | `patterns/debounce.md` | + `infra/redis.md` |
| qualquer `channels.*.enabled == true` | `patterns/ai-rules.md` | lógica de quando responder |
| `channels.waha_core.enabled` OR `channels.waha_plus.enabled` | `channels/waha.md` | |
| `channels.evolution_api.enabled` | `channels/evolution.md` | |
| `channels.whatsapp_cloud.enabled` | `channels/whatsapp-cloud.md` | + `patterns/meta-graph-subscription.md` |
| `channels.instagram_direct.enabled` | `channels/instagram.md` | + `patterns/meta-graph-subscription.md` |
| `"anthropic" ∈ llm.providers` | `llm/anthropic.md` | prompt caching, thinking |
| `"openai" ∈ llm.providers` | `llm/openai.md` | responses API, structured outputs |
| `"google" ∈ llm.providers` | `llm/google.md` | `gemini-3-flash-preview` (chat) + `gemini-embedding-2` (RAG multimodal) |
| `"groq" ∈ llm.providers` | `llm/groq.md` | Llama/Kimi, latência sub-500ms |
| `tools.rag == true` | `patterns/rag-pgvector.md` | (depende também de um `llm/<provider>.md` p/ embedding) |

## 🏗️ Template obrigatório de cada sub-prompt

Todos seguem este shape (facilita parsing e evita prompt drift):

```markdown
---
id: <kebab-case>          # ex: "waha", "evolution"
loadWhen: <condição JS>   # ex: "cfg.channels.waha_core.enabled || cfg.channels.waha_plus.enabled"
context7:                 # sources de verdade usadas
  - /devlikeapro/waha-docs
tokensEstimate: ~1500     # orçamento de contexto aproximado
---

# <Framework/API Name>

## 📋 Quando carregar
<condição clara em linguagem natural>

## 🎯 O que ESTE sub-prompt garante
<bullets: "claude saberá X, Y, Z após ler isto">

## 📚 Endpoints canônicos (verificados via Context7 em <data>)
<tabela: Ação | Método | Endpoint | Notas>

## ⚠️ Business-rule failures conhecidos (preview → evita)
<tabela: Sintoma | Causa raíz | Prevenção obrigatória>

## 🔒 Contratos obrigatórios no código
<código TS mínimo>

## ✅ Checks que o app DEVE fazer
<lista acionável para Claude validar antes de declarar "pronto">

## 🚫 Armadilhas típicas
<bullets: "nunca X", "sempre Y">
```

## 📂 Árvore atual

```
prompts/
├── _INDEX.md                     # este arquivo
├── channels/
│   ├── waha.md                   ✅ verificado Context7
│   ├── evolution.md              ✅ verificado Context7
│   ├── whatsapp-cloud.md         📝 TODO (Meta Graph API)
│   └── instagram.md              📝 TODO (Meta Graph API)
├── llm/
│   ├── anthropic.md              📝 TODO
│   ├── openai.md                 📝 TODO
│   ├── google.md                 ✅ verificado (gemini-3-flash-preview + gemini-embedding-2 multimodal)
│   └── groq.md                   📝 TODO
├── infra/
│   ├── railway.md                ✅ verificado Context7
│   ├── redis.md                  📝 TODO (ioredis + MULTI + SCAN)
│   └── postgres.md               📝 TODO (Prisma + pgvector)
└── patterns/
    ├── idempotent-sessions.md    ✅ cross-cutting
    ├── debounce.md               📝 TODO (Redis sliding window)
    ├── ai-rules.md               📝 TODO (evaluate pipeline)
    ├── hmac-webhook.md           📝 TODO (HMAC SHA256 + X-Hub-Signature-256)
    ├── meta-graph-subscription.md 📝 TODO (common entre cloud+ig)
    └── rag-pgvector.md           📝 TODO
```

## 🔄 Algoritmo de injeção (o que Claude executa após §2 do main)

```ts
// Pseudo-código do loader que o main prompt deve seguir:
function determineSubprompts(cfg: SetupConfig): string[] {
  const load: string[] = [
    "patterns/idempotent-sessions.md",
    "patterns/hmac-webhook.md",
    "infra/railway.md",
    "patterns/ai-rules.md",
  ];

  if (cfg.cache?.debounce?.enabled) {
    load.push("patterns/debounce.md", "infra/redis.md");
  }

  if (cfg.channels.waha_core?.enabled || cfg.channels.waha_plus?.enabled) {
    load.push("channels/waha.md");
  }
  if (cfg.channels.evolution_api?.enabled) {
    load.push("channels/evolution.md");
  }
  if (cfg.channels.whatsapp_cloud?.enabled) {
    load.push("channels/whatsapp-cloud.md", "patterns/meta-graph-subscription.md");
  }
  if (cfg.channels.instagram_direct?.enabled) {
    load.push("channels/instagram.md", "patterns/meta-graph-subscription.md");
  }

  for (const p of cfg.llm.providers ?? []) {
    load.push(`llm/${p}.md`);
  }

  if (cfg.tools?.rag) {
    load.push("patterns/rag-pgvector.md", "infra/postgres.md");
  }

  return [...new Set(load)];  // dedupe
}
```

Claude deve imprimir **uma única linha** no início da fase de scaffold listando os sub-prompts que vai carregar (ex: `📦 Loading context: channels/waha + llms/google + patterns/debounce + ...`), para o usuário ver a decisão antes do scaffold começar.

## ♻️ Atualização do índice
Quando um sub-prompt é criado ou atualizado, atualizar aqui:
1. Marcar ✅ verificado / 📝 TODO na árvore
2. Atualizar `tokensEstimate` no frontmatter do arquivo afetado
3. Se a condição de carga mudar, atualizar a matriz acima + o `loadWhen` do frontmatter

**Regra invariante**: o `loadWhen` no frontmatter é a fonte da verdade. `_INDEX.md` deve ser derivável dele — se divergirem, o `loadWhen` vence.
