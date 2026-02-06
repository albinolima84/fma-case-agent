# 🚀 PROMPT PARA RETOMAR TRABALHO

**Atualização:** 2026-02-05 (sessão 5)
**Progresso:** Sistema 100% + FLUXO 1 automatizado ✅ | Documentação ~98% | Entrega final pendente
**Checkpoint:** `CHECKPOINT-2026-02-05.md`

---

## 🎯 ONDE PARÁMOS — RETOMAR AQUI

O sistema está **100% funcional com FLUXO 1 automatizado**. Na sessão 5 implementamos a busca automática de clientes elegíveis (Proposta Futura #1 do doc 03). Pendente: testes extensivos, export do workflow atualizado e entrega final.

### Tarefas imediatas (ordem de prioridade)

1. **Testar FLUXO 1 completo end-to-end** — FLUXO 1 foi modificado para buscar clientes automaticamente. Testar com mais contatos (10, depois 50) e validar:
   - Loop processa todos os elegíveis
   - Surveys criadas no Supabase
   - Conversas criadas no Chatwoot
   - Nenhum timeout ou trava

2. **Validar FLUXO 2 ainda funciona** — webhook recebendo respostas, Agent 4 processando, Chatwoot atualizando.

3. **Exportar workflow atualizado** — n8n → Export JSON → substituir `workflows/satisfaction-survey-workflow.json` → commitar.

4. **Revisar e commitar README.md** — arquivo tem correções aplicadas mas ainda não commitado.

5. **Validar docs 03 e 04** — foram corrigidos na sessão 4, pendente aprovação do usuário.

6. **Screenshots** (manual):
   - n8n: FLUXO 1 e FLUXO 2
   - Chatwoot: conversa do teste com Carlos Mendes
   - Supabase: tabela `surveys` com dados do teste
   - WhatsApp: conversa no celular (opcional)

5. **Vídeo demo** (manual) — instrução detalhada no `docs/06-guia-entrega.md`, Parte 5.

6. **Montar PDF final** — agregar docs, diagramas (PNG), screenshots e vídeo.

---

## 📋 ESTADO DOS DOCUMENTOS

| Doc | Versão | Git | Revisado? |
|-----|--------|-----|-----------|
| 01-arquitetura-solucao.md | 2.0+ | ✅ Commitado | ✅ |
| 02-agentes-ia-detalhamento.md | 2.0 | ✅ Commitado | ✅ |
| 03-processo-as-is-to-be.md | 2.0 | ✅ Commitado | ⏳ Pendente validação (Proposta #1 implementada) |
| 04-plano-projeto-roi.md | Corrigido | ✅ Commitado | ⏳ Pendente validação |
| 05-guia-deploy-online.md | 2.0 | ✅ Commitado (b2c538f) | ✅ |
| 06-guia-entrega.md | 1.1 | ✅ Commitado (9070fd9) | ✅ |
| workflows/satisfaction-survey-workflow.json | Desatualizado | ⏳ Precisa export | Modificado no n8n |

---

## 📋 CONTEXTO DO SISTEMA

Sistema de **Pesquisa de Satisfação via WhatsApp**:

- **Meta WhatsApp Business API** (oficial, v21.0)
- **n8n Cloud** (automação — `albino.app.n8n.cloud`)
- **Tess AI** (agentes gpt-4o-mini — IDs: 38717, 38728, 38733)
- **Supabase Cloud** (PostgreSQL — tabela: `surveys`)
- **HubSpot** (CRM — contato teste: Carlos Mendes, ID 198003468904)
- **Chatwoot Cloud** (monitoramento — Account 150655, Inbox 94417)

### Preço Chatwoot
- **Piloto (Fases 1–2):** Plano Hacker — gratuito (1 inbox, 2 agentes, retenção 30 dias)
- **Produção (Fase 3+):** Plano Pro — R$105/mês (retenção > 30 dias necessária)

### Valores financeiros (produção — 200 pesquisas/mês)
- OPEX Infra R$105 | OPEX Total R$300 | TO-BE R$4.510/mês
- Economia R$7.590/mês (63%) | Anual R$91.080
- ROI Ano 1: 66% | Ano 2: 166% | 3 anos: 497%
- Payback 8 meses (conservador) | Break-even Mês 8

---

## ✅ O QUE ESTÁ FUNCIONANDO

### FLUXO 1 — Envio Proativo (100% + AUTOMATIZADO 🆕)
```
Schedule → HubSpot Search (atividade 30d) → Extract Results → Loop →
    Extract Contact Data → Supabase Check (survey 30d) → Mark Eligibility →
    IF elegível → HubSpot Get Contact + Emails + Deals + Tickets →
    Agent 1 (Data Fetcher inline) → Agent 2 (Context — Tess 38717) →
    Agent 3 (Message — Tess 38728) → Send WhatsApp (Meta API) →
    Supabase INSERT (status = active) →
    Search/Create Contact (Chatwoot) → Create Conversation → Send Message →
    volta ao Loop (próximo contato)
```

**Novidade (sessão 5):** Sistema agora busca automaticamente contatos com atividade nos últimos 30 dias, filtra quem já recebeu survey recente, e processa apenas os elegíveis em loop. **Não precisa mais hardcoded contact ID!**

### FLUXO 2 — Respostas via Webhook (100%)
```
Webhook Meta → Parse Message → Filter Valid
    → Supabase GET (survey ativa)
    → Send User Message to Chatwoot (incoming)
    → Agent 4 (Conversation Handler V2.0 — Tess 38733)
    → Send WhatsApp Response (Meta API)
    → Send Bot Response to Chatwoot (outgoing)
    → IF status == completed → Add Private Note → Mark Resolved
    → Supabase UPDATE (score, feedback, transcript, status)
    → Respond Webhook
```

---

## 📟 CREDENCIAIS

### Meta WhatsApp API
- Phone Number ID: `674094992450703`
- Número do Bot: `+55 11 5286-8259`
- Cliente teste: `+55 21 98144-4992` (Carlos Mendes)

### n8n Cloud
- URL: `https://albino.app.n8n.cloud`
- Workflow: "Pesquisa Satisfação - Meta API (FINAL)"

### Tess AI
- Agent 2 (Context Analyzer): ID `38717`
- Agent 3 (Message Generator): ID `38728`
- Agent 4 (Conversation Handler V2.0): ID `38733`
- Endpoint: `POST https://api.tess.im/agents/{id}/execute`
- Response: `responses[0].output` (sanitizar marcações ` ```json ``` `)

### Supabase
- Tabela: `surveys`
- Colunas: id, customer_phone, customer_name, context_summary, conversation_transcript (JSONB), satisfaction_score, main_feedback, sentiment, status, chatwoot_conversation_id (INTEGER), created_at, updated_at

### Chatwoot Cloud
- Account ID: `150655` | Inbox ID: `94417`
- API Access Token: `YfdfiZa8YPNM6rhBxpGmdK9M`
- Base URL: `https://app.chatwoot.com/api/v1/accounts/150655`

### HubSpot
- Contato teste: Carlos Mendes (ID: `198003468904`) | Phone: +5521981444992

---

## 🔍 TROUBLESHOOTING (Problemas já resolvidos)

1. **"Phone number has already been taken"** → Search Contact antes de Create Contact
2. **`$json.campo` undefined** → `$('Nome do Node').first().json.campo`
3. **Tess AI output com ` ```json ``` `** → `.replace(/^```json\n?/, '').replace(/\n?```$/, '').trim()` antes de `JSON.parse()`
4. **`source_id` no Create Conversation** → Usar phone do cliente
5. **SQL injection com aspas simples** → `.replace(/'/g, "''")` nos campos `main_feedback` e `conversation_transcript`
6. **n8n Merge após IF trava** → Usar modo **Append** (não Combine)
7. **JSON body com quebra de linha** → Usar apenas `\n` escape
8. **contact_id após IF + Merge** → Code node com lógica condicional

---

## 💬 PROMPT DE EXEMPLO PARA RETOMAR AMANHÃ

Copie e cole no início da nova sessão:

```
Retomando o projeto case-agent-dev. Contexto no arquivo PROMPT-RETOMADA.md
e log detalhado no CHECKPOINT-2026-02-05.md.

Hoje preciso:
1. Testar FLUXO 1 automatizado end-to-end (aumentar para 10, depois 50 contatos)
2. Validar que FLUXO 2 (webhook) ainda funciona
3. Exportar workflow atualizado do n8n e commitar
4. Revisar e aprovar docs 03, 04 e README.md
5. Iniciar coleta de evidências para entrega final (doc 06)
```

---

**Arquivos-chave do repo:**
- `docs/` — documentação (01 a 06)
- `prompts/` — prompts dos agentes (agent-2, agent-3, agent-4)
- `workflows/satisfaction-survey-workflow.json` — workflow n8n exportado
- `diagrams/` — mermaid + PNG renderizados
