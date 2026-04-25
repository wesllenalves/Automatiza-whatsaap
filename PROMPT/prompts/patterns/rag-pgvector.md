---
id: rag-pgvector
loadWhen: cfg.tools?.rag
tokensEstimate: 1100
verifiedAt: null   # 📝 TODO
---

# RAG com pgvector

## 📋 Quando carregar
`tools.rag == true` na config. Usuário quer upload de docs pra contexto do agente.

## 🎯 O que garante
- Extensão `vector` do Postgres instalada antes do migrate
- Chunk strategy: 500 tokens com 50 overlap (texto); ajuste por modalidade (ver tabela abaixo)
- Embedding model do provider habilitado (ver `llm/<provider>.md`)
- Retrieval top-5 por cosine distance
- **Suporte multimodal** via `gemini-embedding-2` (texto + imagem + áudio + vídeo + PDF no mesmo espaço)
- Coluna `embeddingModel` + `modality` em cada chunk p/ prevenir mistura cross-model

## 🗄️ Setup pgvector

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

No Railway: rodar uma vez após Postgres provisioned:
```bash
railway run --service app -- psql $DATABASE_URL -c 'CREATE EXTENSION IF NOT EXISTS vector;'
```

No Prisma (schema.prisma):
```prisma
model RagChunk {
  id             String  @id @default(cuid())
  docId          String
  ord            Int
  text           String                                // caption do item (p/ UI + fallback se multimodal não estiver disponível)
  embedding      Unsupported("vector(768)")?
  embeddingModel String  @default("gemini-embedding-2") // fonte do vetor
  modality       String  @default("text")               // "text" | "image" | "audio" | "video" | "pdf" | "multimodal"
  mediaUrl       String?                                // S3/Railway volume URL se modality != text
  @@index([embeddingModel])
  @@index([docId, ord])
}
```

### Chunk strategy por modalidade

| Modalidade | Estratégia de chunk |
|---|---|
| `text` | 500 tokens, 50 overlap |
| `pdf`  | 1 chunk por página OU 500 tokens extraídos + 50 overlap |
| `image` | 1 chunk por imagem (sem split) |
| `audio` | 1 chunk por segmento de ≤30s (split em silêncios) |
| `video` | 1 chunk por cena (keyframe + áudio do segmento) |
| `multimodal` (ex: "slide + áudio do apresentador") | 1 chunk combinado — `parts: [text, image, audio]` num único `embedContent` |

## 🔧 Indexação

```ts
import { embedBatch } from "@/lib/llm/<provider>";

export async function indexDocument(doc: { id: string; text: string }) {
  const chunks = splitIntoChunks(doc.text, 500, 50);  // 500 tokens, 50 overlap
  const vecs = await embedBatch(chunks, "RETRIEVAL_DOCUMENT");
  for (let i = 0; i < chunks.length; i++) {
    await prisma.$executeRaw`
      INSERT INTO "RagChunk" (id, "docId", ord, text, embedding)
      VALUES (gen_random_uuid()::text, ${doc.id}, ${i}, ${chunks[i]}, ${vecs[i]}::vector)
    `;
  }
}
```

## 🔎 Retrieval

```ts
export async function retrieveTopK(query: string, agentSessionId: string, k = 5) {
  const [qVec] = await embedBatch([query], "RETRIEVAL_QUERY");
  const rows = await prisma.$queryRaw<Array<{ id: string; text: string; distance: number }>>`
    SELECT c.id, c.text, c.embedding <=> ${qVec}::vector AS distance
    FROM "RagChunk" c
    JOIN "RagDoc" d ON d.id = c."docId"
    WHERE d."agentSessionId" = ${agentSessionId}
    ORDER BY distance ASC
    LIMIT ${k}
  `;
  return rows;
}
```

## ⚠️ Failures

| Sintoma | Causa | Prevenção |
|---|---|---|
| `type "vector" does not exist` | extensão não instalada | rodar `CREATE EXTENSION` antes do migrate |
| Retrieval retorna lixo irrelevante | mesmo taskType em index e query | indexar `RETRIEVAL_DOCUMENT`, query `RETRIEVAL_QUERY` |
| Dim mismatch entre embedding e schema | provider mudou ou `outputDimensionality` não setado | validar dim em runtime; schema fixo em 768 |
| Chunk muito grande, contexto limitado | `splitIntoChunks(_, 2000)` | 500 tokens com 50 overlap é sweet spot |
| Retrieval mistura vetores de modelos diferentes (após migração) | query com v2 buscando em vetores v1 | WHERE `embeddingModel = $modelAtivo` em toda query; migrar forçando re-embed |
| Áudio do WhatsApp volta como "nada relevante" | transcrevia p/ texto antes de embedar (perde entonação, contexto) | com `gemini-embedding-2`, embedar o áudio bruto direto (`inlineData: audio/ogg`) |

## 🚫 Armadilhas
- **NUNCA** misturar taskTypes (degrada recall)
- **NUNCA** indexar documentos > 50MB no pgvector — use S3 + indexar só chunks
- **NUNCA** comparar vetores de `embeddingModel` diferentes (filtrar no WHERE)
- **SEMPRE** `CREATE EXTENSION` antes do `prisma migrate deploy`
- **SEMPRE** filtrar por `agentSessionId` no retrieval (multi-agent isolation)
- **SEMPRE** gravar `embeddingModel` + `modality` em cada chunk p/ auditoria e migração
