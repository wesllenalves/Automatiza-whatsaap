---
id: llm-openai
loadWhen: cfg.llm.providers.includes("openai")
context7:
  - /openai/openai-node
tokensEstimate: 1200
verifiedAt: null
---

# OpenAI API

## 📋 Quando carregar
Usuário marcou `OpenAI` em `llm.providers`.

## 🎯 Essenciais
- SDK: `openai` (`npm i openai`)
- Auth: `Authorization: Bearer $OPENAI_API_KEY`
- Defaults recomendados: `gpt-4o-mini` (custo/perf) ou `gpt-4.1` (qualidade)
- **Responses API** (novo) > Chat Completions para tool use complexo
- Structured outputs via `response_format: { type: "json_schema", ... }`

## ⚠️ Failures previstos

| Sintoma | Causa | Prevenção |
|---|---|---|
| Tool calls retornam vazios | esqueceu de habilitar `parallel_tool_calls: false` quando tools dependem entre si | avaliar se parallel ajuda; default é true |
| Custos altos com reasoning models | `o1-mini`/`o3` cobram reasoning tokens | usar `gpt-4o-mini` p/ WhatsApp normal |
| Rate limit RPM baixo em free tier | tier baixo | plano pago; implementar retry com backoff |

## 📖 Fonte
- 📝 **TODO**: validar `/openai/openai-node` em primeira build
- Docs: https://platform.openai.com/docs
