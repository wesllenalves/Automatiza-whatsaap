---
id: ai-rules
loadWhen: always   # baseline — canal habilitado exige rules
tokensEstimate: 1400
verifiedAt: 2026-04-21
---

# AI Activation Rules — pipeline de decisão

## 📋 Quando carregar
**Sempre** (baseline). Todo canal precisa decidir SE/QUANDO responder.

## 🎯 O que garante
- 8 modes suportados (`always_on`, `contact_whitelist/blacklist`, `keyword_trigger/pause`, `schedule`, `manual_approval`, `human_takeover`)
- 4 actions (`respond`, `drop`, `handoff`, `static_reply`)
- Priority-ordered evaluation com short-circuit
- Integração no pipeline DEPOIS do debounce flush, ANTES do LLM
- Override manual por conversa (`conversation.aiEnabled`)

## 🧠 Dados (Prisma — ver §4.0 main)
- `AgentSession` (id, name, systemPrompt, model, provider, memoryMode, tools, isDefault)
- `AIRule` (id, agentSessionId, priority, enabled, mode, params JSON, action, staticReply)
- `Conversation.aiEnabled` — override manual do atendente

## 🧮 Evaluation — pure function

```ts
// apps/web/src/lib/ai-rules/evaluate.ts
export type Decision =
  | { action: "respond"; agentSession: AgentSession }
  | { action: "drop"; reason: string }
  | { action: "handoff"; reason: string }
  | { action: "static_reply"; text: string };

export async function decideAction(
  msgs: ChannelMessage[],
  conv: Conversation,
): Promise<Decision> {
  if (!conv.aiEnabled) return { action: "drop", reason: "conversation.aiEnabled=false" };

  const agentSession = conv.agentSessionId
    ? await prisma.agentSession.findUnique({ where: { id: conv.agentSessionId } })
    : await prisma.agentSession.findFirst({ where: { isDefault: true } });
  if (!agentSession) return { action: "drop", reason: "no_default_agent_session" };

  const rules = await prisma.aIRule.findMany({
    where: { agentSessionId: agentSession.id, enabled: true },
    orderBy: { priority: "asc" },
  });

  for (const r of rules) {
    if (!evaluateRule(r, msgs, conv)) continue;
    if (r.action === "respond")      return { action: "respond", agentSession };
    if (r.action === "drop")         return { action: "drop", reason: `rule:${r.id}` };
    if (r.action === "handoff")      return { action: "handoff", reason: `rule:${r.id}` };
    if (r.action === "static_reply") return { action: "static_reply", text: r.staticReply ?? "" };
  }
  return { action: "respond", agentSession };  // default: always_on implícito
}

function evaluateRule(rule: AIRule, msgs: ChannelMessage[], conv: Conversation): boolean {
  const p = rule.params as any;
  switch (rule.mode) {
    case "always_on":          return true;
    case "contact_whitelist":  return matchesContactList(p, conv.contactId);
    case "contact_blacklist":  return matchesContactList(p, conv.contactId);
    case "keyword_trigger":    return matchesKeywords(p, msgs);
    case "keyword_pause":      return matchesKeywords(p, msgs);
    case "schedule":           return insideSchedule(p, new Date());
    case "human_takeover":     return recentHumanMessage(conv.id, p.silenceHours ?? 24);
    case "manual_approval":    return true;
    default: return false;
  }
}
```

## 🎛️ Modes (referência)

| Mode | `params` shape |
|---|---|
| `always_on` | `{}` |
| `contact_whitelist` | `{ contactIds?: string[], tags?: string[] }` |
| `contact_blacklist` | `{ contactIds?: string[], tags?: string[] }` |
| `keyword_trigger` | `{ keywords: string[], matchMode: "any"\|"all", caseSensitive?: boolean }` |
| `keyword_pause` | `{ keywords: string[] }` |
| `schedule` | `{ timezone: string, days: number[], start: "HH:MM", end: "HH:MM" }` |
| `manual_approval` | `{ notifyVia: "panel"\|"slack" }` |
| `human_takeover` | `{ silenceHours: number }` |

## ⚠️ Failures previstos

| Sintoma | Causa | Prevenção |
|---|---|---|
| IA responde mesmo com rule `keyword_pause` ativa | priority errado (rule `always_on` veio antes) | `keyword_pause` deve ter priority MENOR (avaliado antes); validar em rules editor |
| IA calou pra sempre após 1 msg do humano | `human_takeover` sem janela | definir `silenceHours` (default 24); reset ao humano não mandar em N horas |
| `manual_approval` chama LLM na hora | bug: deve apenas abrir ticket | ação especial: não invoca `flushToLLM` — enfileira p/ review humana |
| Regras de contato não casam | comparou `contactId` externo com id interno | sempre normalizar: `conversation.contactId` é id da tabela Contact, não `phone@c.us` |

## 🚫 Armadilhas
- **NUNCA** avaliar rules SEM respeitar `conversation.aiEnabled=false` primeiro (é override manual, mais forte)
- **NUNCA** short-circuitar antes de checar priority (ordem importa)
- **NUNCA** invocar LLM em `manual_approval` (deve parar e notificar)
- **SEMPRE** logar `{ruleId, decision, reason}` pra auditoria
- **SEMPRE** cache decision por `(conversationId, hash(msgs))` se latência importar

## 📖 Fonte
- §5.5 do main spec (detalhado)
- UI: `/settings/agents/[id]/rules` — drag-to-reorder priority
