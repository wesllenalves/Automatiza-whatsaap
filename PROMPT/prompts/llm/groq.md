---
id: llm-groq
loadWhen: cfg.llm.providers.includes("groq")
context7:
  - /groq/groq-typescript
tokensEstimate: 1000
verifiedAt: null
---

# Groq API

## 📋 Quando carregar
Usuário marcou `Groq` em `llm.providers`.

## 🎯 Essenciais
- SDK: `groq-sdk` (OpenAI-compatible)
- Auth: `Authorization: Bearer $GROQ_API_KEY`
- Base URL: `https://api.groq.com/openai/v1`
- Models recomendados p/ WhatsApp (latência sub-500ms):
  - `llama-3.3-70b-versatile` — qualidade
  - `llama-3.1-8b-instant` — velocidade máxima
- Tool use no mesmo shape OpenAI

## ⚠️ Failures previstos
| Sintoma | Causa | Prevenção |
|---|---|---|
| 429 rapidamente | rate limit TPM (tokens/min) baixo no free | tier dev pago; usar como fallback, não primário em alto volume |
| Tool use degradado em modelos 8b | modelo pequeno | usar 70b pra tool use; 8b só chat simples |
| Contexto corta em 8k | context window menor que outros | limit dos últimos 10 turns |

## 📖 Fonte
- 📝 **TODO**: validar `/groq/groq-typescript` em primeira build
- Docs: https://console.groq.com/docs
