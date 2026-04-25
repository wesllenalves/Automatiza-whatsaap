---
id: llm-anthropic
loadWhen: cfg.llm.providers.includes("anthropic")
context7:
  - /anthropics/anthropic-sdk-typescript
tokensEstimate: 1500
verifiedAt: null   # 📝 TODO validar Context7 no primeiro build
---

# Anthropic Claude API

## 📋 Quando carregar
Usuário marcou `Anthropic Claude` em `llm.providers` na Onda 2.

## 🎯 O que garante
- Model IDs atuais: `claude-opus-4-7`, `claude-sonnet-4-6`, `claude-haiku-4-5`
- **Prompt caching** obrigatório pra system prompts do agente (>1024 tokens) — reduz custo 90%
- Streaming via SSE
- Tool use shape

## 📚 Essenciais

**SDK**: `@anthropic-ai/sdk`
**Auth**: `x-api-key: $ANTHROPIC_API_KEY`
**Base**: `https://api.anthropic.com/v1`

### Model pricing tier (escolha default)
| Uso | Model | Motivo |
|---|---|---|
| Chat SDR/suporte | `claude-sonnet-4-6` | balance qualidade/custo |
| Pesquisa profunda | `claude-opus-4-7` | só em tool calls caros |
| Resposta rápida curta | `claude-haiku-4-5` | latência sub-segundo |

### Prompt caching (1 call)
```ts
const resp = await client.messages.create({
  model: "claude-sonnet-4-6",
  system: [
    { type: "text", text: staticSystemPrompt, cache_control: { type: "ephemeral" } },
    { type: "text", text: dynamicContextMsg },
  ],
  messages,
  max_tokens: 1024,
});
```

## ⚠️ Business-rule failures previstos

| Sintoma | Causa | Prevenção |
|---|---|---|
| Custo explodindo | system prompt sem cache_control | sempre marcar system como `cache_control: ephemeral` |
| Tool calls que retornam texto cru | modelo não foi instruído a parar após tool | `stop_sequences: ["</tool_use>"]` + parse estrutural |
| 429 em burst | rate limit por RPM | backoff exponencial; tier superior para produção |

## 📖 Fonte
- 📝 **TODO**: validar via `/anthropics/anthropic-sdk-typescript` no primeiro build
- Skill SuperClaude: `claude-api` (já cobre caching + retries)
- Docs: https://docs.anthropic.com/en/api/overview
