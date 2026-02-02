# Guia de Setup - n8n Cloud

**Data:** 2026-01-30
**Arquivo do Workflow:** `workflows/satisfaction-survey.json`

---

## 🎯 Visão Geral

Este workflow implementa o sistema completo de pesquisa de satisfação com 2 fluxos integrados:

### **FLUXO 1: ENVIO PROATIVO** (13 nodes)
Schedule → HubSpot → Agente 2 (análise) → Agente 3 (mensagem) → WhatsApp → Create Survey

### **FLUXO 2: RECEBER RESPOSTAS** (10 nodes)
Webhook WhatsApp → Get Survey → Agente 4 (conversa) → WhatsApp → Update Survey

**Total:** 23 nodes funcionais

---

## 📐 Arquitetura Visual

```
┌─────────────────────────────────────────────────────────────┐
│ FLUXO 1: ENVIO PROATIVO (linha superior)                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  [Schedule]                                                 │
│      ↓                                                      │
│  [Set Contact ID]                                           │
│      ↓                                                      │
│  [HubSpot: Get Contact, Emails, Deals, Tickets] (paralelo) │
│      ↓                                                      │
│  [Consolidate Data] ← AGENTE 1 inline                       │
│      ↓                                                      │
│  [Tess Agente 2] → Context Analyzer                         │
│      ↓                                                      │
│  [Parse Analysis]                                           │
│      ↓                                                      │
│  [Tess Agente 3] → Message Generator                        │
│      ↓                                                      │
│  [Parse Message]                                            │
│      ↓                                                      │
│  [Send Initial WhatsApp]                                    │
│      ↓                                                      │
│  [Create Survey in Supabase]                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ FLUXO 2: RECEBER RESPOSTAS (linha inferior)                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  [Webhook WhatsApp]                                         │
│      ↓                                                      │
│  [Parse Message]                                            │
│      ↓                                                      │
│  [Filter - Customer Only]                                   │
│      ↓                                                      │
│  [Get Survey from Supabase]                                 │
│      ↓                                                      │
│  [Prepare Tess Input]                                       │
│      ↓                                                      │
│  [Tess Agente 4] → Conversation Handler                     │
│      ↓                                                      │
│  [Parse Response]                                           │
│      ↓                                                      │
│  [Send WhatsApp Response]                                   │
│      ↓                                                      │
│  [Update Survey]                                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔧 Passo 1: Importar Workflow

1. No n8n Cloud, clique em **Workflows > Import from File**
2. Selecione: `satisfaction-survey-COMPLETO.json`
3. Workflow será importado com 23 nodes
4. Organize visualmente (use auto-arrange se disponível)

---

## 🔐 Passo 2: Configurar Credenciais

### 2.1 - Tess API Token

**Nome:** `Tess API Token`
**Tipo:** Header Auth

```
Header Name: Authorization
Header Value: Bearer [SEU_TOKEN_TESS]
```

**Aplicar em:**
- Tess - Agente 2 (Análise)
- Tess - Agente 3 (Mensagem)
- Tess - Agente 4 (Conversa)

---

### 2.2 - HubSpot API Key

**Nome:** `HubSpot API Key`
**Tipo:** Header Auth

```
Header Name: Authorization
Header Value: Bearer [SUA_API_KEY_HUBSPOT]
```

**Onde encontrar:**
- HubSpot > Settings > Integrations > Private Apps
- Criar novo Private App se necessário
- Permissões necessárias: `crm.objects.contacts.read`, `crm.objects.deals.read`, `crm.objects.tickets.read`

**Aplicar em:**
- HubSpot - Get Contact
- HubSpot - Get Emails
- HubSpot - Get Deals
- HubSpot - Get Tickets

---

### 2.3 - Evolution API Key

**Nome:** `Evolution API Key`
**Tipo:** Header Auth

```
Header Name: apikey
Header Value: worshiper-stagnate-spelling-handwork-aloha-sardine
```

**Aplicar em:**
- Send Initial WhatsApp
- Send WhatsApp Response

---

### 2.4 - Supabase PostgreSQL

**Nome:** `Supabase PostgreSQL`
**Tipo:** Postgres

```
Host: aws-1-sa-east-1.pooler.supabase.com
Port: 5432
Database: postgres
Schema: public
User: postgres.bvjvfqktmgegwwztrece
Password: TvU*&7FCw013xJhXNndX
SSL: Enabled (require)
```

**Aplicar em:**
- Supabase - Create Survey
- Supabase - Get Survey
- Supabase - Update Survey

---

## 🗄️ Passo 3: Criar Tabela no Supabase

Execute este SQL no Supabase (Query Editor):

```sql
-- Criar tabela de pesquisas
CREATE TABLE IF NOT EXISTS surveys (
  id SERIAL PRIMARY KEY,
  customer_phone VARCHAR(20) NOT NULL,
  customer_name VARCHAR(100),
  context_summary TEXT,
  conversation_transcript JSONB DEFAULT '[]',
  satisfaction_score INTEGER CHECK (satisfaction_score BETWEEN 1 AND 5),
  main_feedback TEXT,
  sentiment VARCHAR(20) CHECK (sentiment IN ('positive', 'neutral', 'negative')),
  status VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'completed', 'escalated')),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Criar índice para performance
CREATE INDEX idx_surveys_phone_status ON surveys(customer_phone, status);
CREATE INDEX idx_surveys_created_at ON surveys(created_at DESC);
```

---

## 🔗 Passo 4: Configurar Webhook

### 4.1 - Ativar Webhook no n8n

1. No workflow, clique no node **"Webhook WhatsApp"**
2. Ative o workflow (ou clique em "Listen for Test Event")
3. Copie a **URL do webhook** gerada

### 4.2 - Configurar na Evolution API

```bash
curl -X POST https://evolution-api-demo.onrender.com/webhook/set/satisfaction-survey \
  -H "Content-Type: application/json" \
  -H "apikey: worshiper-stagnate-spelling-handwork-aloha-sardine" \
  -d '{
    "url": "https://seu-username.app.n8n.cloud/webhook/whatsapp-incoming",
    "webhook_by_events": true,
    "events": ["MESSAGES_UPSERT"]
  }'
```

---

## 🧪 Passo 5: Testar FLUXO 1 (Envio Proativo)

### Configuração Inicial

1. **No node "Set Contact ID"**, edite o valor:
   ```json
   {
     "hubspot_contact_id": "12345"
   }
   ```
   Substitua `12345` por um ID real de contato do HubSpot.

2. **OU mockar dados HubSpot** (se não tiver API key ainda):
   - Desative nodes do HubSpot
   - No node "Consolidate HubSpot Data", mockar dados:
   ```javascript
   return {
     json: {
       hubspot_data: {
         contact: {
           id: "mock123",
           name: "João Silva",
           email: "joao@teste.com",
           phone: "5511999999999",
           company: "Empresa Teste",
           created_date: "2024-01-01"
         },
         summary: {
           total_emails: 5,
           total_deals: 1,
           total_tickets: 0
         },
         deals: [{
           name: "Upgrade Plano Pro",
           stage: "closedwon",
           amount: "1000"
         }],
         emails: [],
         tickets: []
       },
       customer_phone: "5511999999999",
       customer_name: "João Silva"
     }
   };
   ```

### Executar Teste

1. **Clique em "Execute Workflow"** no node "Schedule - Enviar Pesquisas"
2. Acompanhe a execução:
   - ✅ Get HubSpot data
   - ✅ Consolidate data
   - ✅ Tess Agente 2 analisa contexto
   - ✅ Tess Agente 3 gera mensagem
   - ✅ Evolution API envia WhatsApp
   - ✅ Supabase cria survey
3. **Verifique WhatsApp:** Deve receber mensagem personalizada
4. **Verifique Supabase:**
   ```sql
   SELECT * FROM surveys ORDER BY created_at DESC LIMIT 1;
   ```

---

## 🧪 Passo 6: Testar FLUXO 2 (Receber Respostas)

Agora que você recebeu a mensagem inicial, responda:

### Teste Completo

1. **Você (WhatsApp):** "5"
   - ✅ Webhook recebe
   - ✅ Get Survey
   - ✅ Tess Agente 4 processa
   - ✅ Responde: "Que ótimo! O que você mais gostou?"
   - ✅ Update Survey (score: 5)

2. **Você (WhatsApp):** "Muito fácil de usar!"
   - ✅ Agente 4 processa feedback
   - ✅ Responde: "Obrigada! Ficamos felizes..."
   - ✅ Update Survey (feedback: "Muito fácil de usar", status: completed)

3. **Verificar no Supabase:**
   ```sql
   SELECT
     customer_name,
     satisfaction_score,
     main_feedback,
     sentiment,
     status,
     conversation_transcript
   FROM surveys
   WHERE customer_phone = '5511999999999'
   ORDER BY created_at DESC
   LIMIT 1;
   ```

Deve mostrar:
- `satisfaction_score: 5`
- `main_feedback: "Muito fácil de usar!"`
- `sentiment: "positive"`
- `status: "completed"`
- `conversation_transcript`: Array com todas as mensagens

---

## 🔄 Fluxo Completo End-to-End

```
DIA 1 - 10h (FLUXO 1):
1. Schedule trigger executa
2. Busca dados do cliente no HubSpot
3. Agente 2 analisa contexto → "Cliente com upgrade recente, satisfeito"
4. Agente 3 gera mensagem → "Oi João! Vi que você fez upgrade. Como está sendo?"
5. Envia via WhatsApp
6. Cria survey no Supabase (status: active)

CLIENTE RESPONDE (FLUXO 2):
7. Cliente: "5"
8. Webhook recebe
9. Get survey ativo
10. Agente 4 processa → "Que ótimo! O que mais gostou?"
11. Envia resposta
12. Update survey (score: 5, sentiment: positive)

CLIENTE CONTINUA:
13. Cliente: "As novas funcionalidades!"
14. Agente 4 → "Obrigada pelo feedback!"
15. Update survey (feedback: "novas funcionalidades", status: completed)
```

---

## 📊 Monitoramento

### Métricas no Supabase

```sql
-- Dashboard de pesquisas
SELECT
  COUNT(*) as total_pesquisas,
  COUNT(*) FILTER (WHERE status = 'completed') as completas,
  COUNT(*) FILTER (WHERE status = 'active') as em_andamento,
  ROUND(AVG(satisfaction_score), 2) as nota_media,
  COUNT(*) FILTER (WHERE sentiment = 'positive') as positivos,
  COUNT(*) FILTER (WHERE sentiment = 'negative') as negativos,
  COUNT(*) FILTER (WHERE sentiment = 'neutral') as neutros
FROM surveys
WHERE created_at >= NOW() - INTERVAL '30 days';

-- NPS (Net Promoter Score)
WITH nps_calc AS (
  SELECT
    CASE
      WHEN satisfaction_score >= 4 THEN 'promoter'
      WHEN satisfaction_score >= 3 THEN 'passive'
      ELSE 'detractor'
    END as nps_category
  FROM surveys
  WHERE satisfaction_score IS NOT NULL
    AND created_at >= NOW() - INTERVAL '30 days'
)
SELECT
  ROUND(
    (COUNT(*) FILTER (WHERE nps_category = 'promoter') * 100.0 / COUNT(*)) -
    (COUNT(*) FILTER (WHERE nps_category = 'detractor') * 100.0 / COUNT(*))
  ) as nps_score
FROM nps_calc;

-- Top feedbacks positivos
SELECT
  customer_name,
  satisfaction_score,
  main_feedback,
  created_at
FROM surveys
WHERE sentiment = 'positive'
  AND main_feedback IS NOT NULL
ORDER BY satisfaction_score DESC, created_at DESC
LIMIT 10;

-- Alertas negativos
SELECT
  customer_name,
  customer_phone,
  satisfaction_score,
  main_feedback,
  context_summary,
  created_at
FROM surveys
WHERE sentiment = 'negative'
  OR satisfaction_score <= 2
ORDER BY created_at DESC;
```

### Custos Tess

```sql
-- Estimar consumo de créditos
-- (assumindo ~0.002-0.005 por agente)
SELECT
  COUNT(*) * 0.003 as creditos_agente_2,
  COUNT(*) * 0.003 as creditos_agente_3,
  COUNT(*) * 3 * 0.003 as creditos_agente_4_medio,
  COUNT(*) * (0.003 + 0.003 + 0.009) as total_estimado
FROM surveys
WHERE created_at >= NOW() - INTERVAL '30 days';
```

---

## 🐛 Troubleshooting

### FLUXO 1: HubSpot retorna 401
**Causa:** API key inválida ou sem permissões
**Solução:**
- Verificar Private App no HubSpot
- Permissões: contacts.read, deals.read, tickets.read, emails.read
- Regenerar token se necessário

---

### FLUXO 1: Agente 2 ou 3 não retorna JSON
**Causa:** Prompt do agente não está instruindo retornar JSON
**Solução:**
- Verificar prompts na Tess
- Agente 2 deve retornar: `{"summary": "...", "sentiment": "...", ...}`
- Agente 3 deve retornar: texto puro (não JSON)

---

### FLUXO 1: WhatsApp não envia
**Causa:** Número de telefone em formato incorreto ou instância offline
**Solução:**
- Formato correto: `5511999999999` (DDI + DDD + número)
- Verificar se Evolution API está online
- Verificar se instância está conectada

---

### FLUXO 2: Webhook não recebe
**Causa:** URL incorreta ou Evolution API não configurada
**Solução:**
- Reconfigurar webhook (Passo 4.2)
- Verificar: `curl GET /webhook/find/satisfaction-survey`

---

### FLUXO 2: "No survey found"
**Causa:** Não existe pesquisa ativa para o número
**Solução:**
- Executar FLUXO 1 primeiro (cria o survey)
- OU inserir manual: `INSERT INTO surveys ...`

---

### Schedule não executa automaticamente
**Causa:** Workflow não está ativo ou plano n8n não permite
**Solução:**
- Ativar workflow (toggle no topo)
- n8n Cloud free tier: verificar limites
- Testar manual: clicar "Execute Workflow"

---

## 🚀 Otimizações Futuras

### 1. Múltiplos Contatos

Substituir "Set Contact ID" por:
```
Get Eligible Contacts from Supabase/HubSpot
   ↓
Split Into Batches (10 por vez)
   ↓
For Each Contact → Execute FLUXO 1
```

### 2. Filtros Inteligentes

Adicionar IF nodes:
- Só enviar se não tiver pesquisa nos últimos 30 dias
- Pular se cliente tiver ticket crítico aberto
- Priorizar clientes com upgrades recentes

### 3. Escalation Automática

Quando `sentiment = "negative"` ou `score <= 2`:
```
IF Negative → Send Slack Alert → Create Ticket → Assign to Manager
```

### 4. A/B Testing

Duplicar FLUXO 1 com variações:
- Teste A: Tom "celebratory"
- Teste B: Tom "curious"
- Medir taxa de resposta

---

## 📋 Checklist de Produção

Antes de usar em produção:

- [ ] **Credenciais configuradas:**
  - [ ] Tess API Token
  - [ ] HubSpot API Key
  - [ ] Evolution API Key
  - [ ] Supabase PostgreSQL

- [ ] **Tabela Supabase criada e testada**

- [ ] **Webhook configurado e testado:**
  - [ ] URL correta
  - [ ] Events: MESSAGES_UPSERT
  - [ ] Testado recebimento

- [ ] **FLUXO 1 testado:**
  - [ ] Schedule ou trigger manual
  - [ ] HubSpot retorna dados (ou mock funciona)
  - [ ] Agente 2 analisa corretamente
  - [ ] Agente 3 gera mensagem natural
  - [ ] WhatsApp envia
  - [ ] Survey criado no Supabase

- [ ] **FLUXO 2 testado:**
  - [ ] Webhook recebe mensagens
  - [ ] Agente 4 responde naturalmente
  - [ ] Detecta notas corretamente
  - [ ] Extrai feedback
  - [ ] Finaliza conversa
  - [ ] Update Survey correto

- [ ] **Monitoramento configurado:**
  - [ ] Dashboard Supabase (queries acima)
  - [ ] Alertas para notas baixas
  - [ ] Acompanhamento de créditos Tess

- [ ] **Documentação atualizada:**
  - [ ] IDs dos agentes Tess corretos
  - [ ] Números de telefone de teste
  - [ ] Contatos HubSpot de teste

---

## 💡 Dicas Finais

### Performance
- FLUXO 1 leva ~10-15 segundos (HubSpot + 2 agentes)
- FLUXO 2 leva ~3-5 segundos por turno
- Total por pesquisa completa: ~30-60 segundos

### Custos
- Tess: ~0.015 créditos por pesquisa completa
- n8n Cloud: Conta execuções (free tier: 5000/mês)
- Evolution API: Grátis (self-hosted) ou pago (cloud)

### Escalabilidade
- 100 pesquisas/dia = 3000/mês
- n8n free tier: suficiente
- Tess: ~45 créditos/mês
- Considerar upgrade se > 500 pesquisas/dia

---

**Status:** Workflow completo pronto para produção
**Última atualização:** 2026-01-30
**Próxima ação:** Configurar credenciais e testar os 2 fluxos
