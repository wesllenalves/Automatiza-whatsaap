---
id: llm-google
loadWhen: cfg.llm.providers.includes("google")
context7:
  - /websites/ai_google_dev_gemini-api
tokensEstimate: 1800
verifiedAt: 2026-04-21
---

# Google Gemini API

## 📋 Quando carregar
Usuário marcou `Google Gemini` em `llm.providers` na Onda 2.

## 🎯 O que este sub-prompt garante
- Model ID correto: **`gemini-3-flash-preview`** (chat) e **`gemini-embedding-2`** (RAG **multimodal**)
- SDK correto: **`@google/genai`** (não `@google/generative-ai` — esse é o antigo)
- Uso de `thinkingLevel` para latência sub-segundo em respostas curtas de WhatsApp
- Formato de embedding para pgvector
- **Gemini Embedding 2 = multimodal nativo**: texto, imagem, áudio, vídeo e documentos no MESMO espaço de embedding (ideal p/ WhatsApp: áudios de voz + imagens + PDFs sem OCR/transcrição prévia)

## 📚 Models canônicos (verificados 2026-04-21 — blog Google + docs AI Studio)

| Uso | Model ID | Notas |
|---|---|---|
| Chat (padrão SDR/suporte) | `gemini-3-flash-preview` | flagship speed-tier 2026; thinking opcional |
| Chat (mais caro, mais raciocínio) | `gemini-3-pro` | só p/ tarefas complexas (não precisa p/ WhatsApp MVP) |
| Embeddings (RAG, **multimodal**) | `gemini-embedding-2` | lançado 10-mar-2026 em public preview. Mapeia texto/imagem/vídeo/áudio/docs no mesmo espaço |
| Embeddings (legado text-only) | `gemini-embedding-001` | fallback; usar só se `gemini-embedding-2` indisponível na região |
| Image generation | `gemini-3-flash-image` | se precisar gerar imagem (não aplicável a agente WA) |

> ⚠️ **Gemini Embedding 2 está em preview** — verificar disponibilidade regional no primeiro call. Se 404/400 inesperado, fallback p/ `gemini-embedding-001` (apenas texto).

### 🔧 Task types de embedding (relevantes p/ RAG)
- `RETRIEVAL_DOCUMENT` — p/ indexar PDFs/docs
- `RETRIEVAL_QUERY` — p/ query do usuário na hora de buscar (usar MESMO model mas diferente taskType)
- `SEMANTIC_SIMILARITY` — p/ comparar duas strings
- `CLASSIFICATION` / `CLUSTERING` — outros usos

**Regra**: indexar com `RETRIEVAL_DOCUMENT`, buscar com `RETRIEVAL_QUERY`. Misturar as duas degrada qualidade.

## 🔒 Contrato mínimo

`apps/web/src/lib/llm/google.ts`:

```ts
import { GoogleGenAI } from "@google/genai";

const ai = new GoogleGenAI({ apiKey: process.env.GOOGLE_API_KEY });

export async function chatGemini(args: {
  system: string;
  messages: { role: "user" | "assistant"; content: string }[];
  model?: string;
  thinkingLevel?: "low" | "medium" | "high";
}) {
  const res = await ai.models.generateContent({
    model: args.model ?? "gemini-3-flash-preview",
    contents: [
      { role: "user", parts: [{ text: args.system }] },
      ...args.messages.map(m => ({
        role: m.role === "assistant" ? "model" : "user",
        parts: [{ text: m.content }],
      })),
    ],
    config: {
      thinkingConfig: { thinkingLevel: args.thinkingLevel ?? "low" },  // low p/ WhatsApp
    },
  });
  return { text: res.text ?? "" };
}

// Text-only embedding
export async function embedBatch(texts: string[], taskType: "RETRIEVAL_DOCUMENT" | "RETRIEVAL_QUERY") {
  const res = await ai.models.embedContent({
    model: "gemini-embedding-2",
    contents: texts,
    config: {
      outputDimensionality: 768,
      taskType,
    },
  });
  return res.embeddings.map(e => e.values);   // number[][]
}

// Multimodal embedding — texto + imagem/áudio/vídeo no MESMO vetor
export async function embedMultimodal(
  items: Array<{ text?: string; imageBase64?: string; audioBase64?: string; pdfBase64?: string }>,
  taskType: "RETRIEVAL_DOCUMENT" | "RETRIEVAL_QUERY",
) {
  const res = await ai.models.embedContent({
    model: "gemini-embedding-2",
    contents: items.map(i => ({
      parts: [
        ...(i.text ? [{ text: i.text }] : []),
        ...(i.imageBase64 ? [{ inlineData: { mimeType: "image/jpeg", data: i.imageBase64 } }] : []),
        ...(i.audioBase64 ? [{ inlineData: { mimeType: "audio/ogg", data: i.audioBase64 } }] : []),
        ...(i.pdfBase64   ? [{ inlineData: { mimeType: "application/pdf", data: i.pdfBase64   } }] : []),
      ],
    })),
    config: { taskType, outputDimensionality: 768 },
  });
  return res.embeddings.map(e => e.values);
}
```

### Uso no RAG (pgvector) — multimodal nativo
```ts
// Indexar: texto + imagem + áudio no MESMO espaço vetorial
const docs = [
  { text: "Política de garantia..." },               // texto puro
  { imageBase64: screenshotB64 },                    // só imagem
  { text: "Áudio do cliente:", audioBase64: voiceB64 }, // texto + áudio
];
const vecs = await embedMultimodal(docs, "RETRIEVAL_DOCUMENT");

// Query na hora da msg (texto OU áudio do WA):
const [queryVec] = await embedMultimodal(
  [{ text: userText, audioBase64: voiceNote }],
  "RETRIEVAL_QUERY",
);
const topK = await prisma.$queryRaw`
  SELECT id, text FROM "RagChunk"
  ORDER BY embedding <=> ${queryVec}::vector LIMIT 5
`;
```
> **Ganho concreto p/ WhatsApp**: áudio do usuário não precisa mais passar por Whisper/transcrição antes de buscar — embed direto no mesmo espaço que o texto indexado.

## ⚠️ Business-rule failures

| Sintoma | Causa raíz | Prevenção |
|---|---|---|
| `400 "model gemini-flash-3 not found"` | ID errado — API usa `gemini-3-flash-preview` | NUNCA usar `gemini-flash-3`; **sempre** `gemini-3-flash-preview` |
| `400 "model gemini-embedding-2 not found"` em alguma região | modelo em public preview, rollout regional | fallback p/ `gemini-embedding-001` se 404/400; logar region + retry |
| Embeddings multimodais misturando imagens retornam vetores incoerentes | usou `embed_content` text-only p/ blob de imagem | usar `inlineData.mimeType` apropriado (`image/jpeg`, `audio/ogg`, `application/pdf`) no `parts` |
| Latência de 5-10s em resposta curta WA | `thinkingLevel` em default (auto/high) | fixar `thinkingLevel: "low"` pra resposta de WhatsApp (latência <2s) |
| Embeddings retornam 3072 dims mas pgvector declarado como 768 | `outputDimensionality` não foi passado | SEMPRE passar `outputDimensionality: 768` no embedContent |
| Qualidade RAG caiu após upgrade | query e doc usaram mesmo taskType | indexar com `RETRIEVAL_DOCUMENT`, query com `RETRIEVAL_QUERY` |
| SDK antigo `@google/generative-ai` deprecated warnings | pacote errado | **migrar pra `@google/genai`** (novo, unified SDK) |
| Rate limit 429 em burst | sem retry | usar exponential backoff; Gemini rate limit é por minuto |
| Migração de `gemini-embedding-001` → `gemini-embedding-2` mistura vetores velhos/novos | espaços vetoriais **diferentes** entre versões | re-embedar TUDO ao migrar — nunca comparar vetor v1 com v2 |

## ✅ Checks

1. `process.env.GOOGLE_API_KEY` existe e é válido (ping: `ai.models.list()`)
2. `package.json` tem `@google/genai` (NÃO `@google/generative-ai`)
3. Embedding dim consistente: schema pgvector `vector(768)` ↔ SDK `outputDimensionality: 768`
4. Todos os call de embedding passam `taskType` explícito (nunca default)
5. Antes do primeiro deploy: fazer 1 embed de teste com `gemini-embedding-2`. Se 404/400, gravar `llm.embeddingModel = "gemini-embedding-001"` no `setup.config.json` (fallback) e re-rodar
6. Cada `RagChunk` grava em coluna `modelUsed` (novo campo Prisma: `embeddingModel String @default("gemini-embedding-2")`) qual modelo gerou o vetor — previne mistura futura

## 🚫 Armadilhas

- **NUNCA** usar `gemini-flash-3` — ID inexistente. Usar `gemini-3-flash-preview`
- **NUNCA** usar `@google/generative-ai` (antigo) — usar `@google/genai`
- **NUNCA** misturar taskTypes RETRIEVAL_DOCUMENT e RETRIEVAL_QUERY no mesmo embedding (degrada recall em ~15%)
- **NUNCA** comparar vetores gerados com `gemini-embedding-001` contra vetores de `gemini-embedding-2` — espaços diferentes
- **SEMPRE** `thinkingLevel: "low"` p/ respostas de WhatsApp (latência crítica)
- **SEMPRE** passar `outputDimensionality` explícito
- **SEMPRE** usar `inlineData` com `mimeType` correto p/ embedding multimodal (imagem/áudio/pdf)
- Em caso de fallback `gemini-embedding-2` → `gemini-embedding-001`, **marcar no log** qual modelo gerou cada chunk (precisa re-embedar se migrar)

## 📖 Fonte
- `gemini-3-flash-preview`: docs AI Studio `https://ai.google.dev/gemini-api/docs/models/gemini-3-flash-preview` (verificado 2026-04-21)
- `gemini-embedding-2`: blog `https://blog.google/innovation-and-ai/models-and-research/gemini-models/gemini-embedding-2/` (10-mar-2026, public preview)
- SDK NPM: `@google/genai`
- Base URL: `https://generativelanguage.googleapis.com/v1beta/`
- Auth header: `x-goog-api-key: $GEMINI_API_KEY`
- ⚠️ Context7 ainda indexando `gemini-embedding-2` — validar dimensões exatas + limite de taskTypes no primeiro call; fallback seguro é `gemini-embedding-001`
