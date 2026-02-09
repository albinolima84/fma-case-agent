# Arquitetura da Solução - Sistema de Satisfação de Clientes com IA

## 1. Visão Geral

Sistema automatizado end-to-end para medição de satisfação de clientes através de agentes de IA conversacionais, integrando dados do HubSpot CRM, processamento com LLMs e canais de mensageria.

### Objetivo
Reduzir de **30 minutos para ~2 minutos** o processo de análise e contato com cada cliente, garantindo personalização, consistência e rastreabilidade completa.

---

## 2. Stack Tecnológica

### 2.1 Orquestração e Automação
**Ferramenta:** n8n Cloud

**Justificativa:**
- Integração nativa com HubSpot, APIs de IA e webhooks
- Suporte a código customizado (JavaScript) para lógica complexa
- Interface visual para manutenção futura pela equipe técnica
- Sistema robusto de logs e retry em caso de falhas
- Webhook URL pública sem necessidade de servidor próprio
- Tier gratuito suficiente para o volume atual

**Alternativas descartadas:**
- Make: Custo ~$300/mês em escala
- Zapier: Limitado para lógica multi-step complexa

---

### 2.2 Motor de IA (LLM)
**Ferramenta:** Tess AI (gpt-4o-mini)

**Justificativa:**
- Modelo gpt-4o-mini: excelente custo-benefício para tarefas de análise e geração de texto
- Suporte a agentes especializados com prompts configuráveis na plataforma Tess
- Latência baixa (~1-3s por resposta)
- API REST simples: execute via endpoint `/agents/{id}/execute`
- Integração direta com n8n via HTTP Request

**Configuração (por agente):**
- Agente 2 (Context Analyzer): ID `38717`, Temperature `1`
- Agente 3 (Message Generator): ID `38728`, Temperature `1`
- Agente 4 (Conversation Handler): ID `38733`, Temperature `1`

**Formato de resposta:** `responses[0].output` — string que pode conter marcações ````json``` ` (tratamento de sanitização aplicado no workflow)

---

### 2.3 Canal de Mensageria
**Ferramenta:** Meta WhatsApp Business API (oficial)

**Justificativa:**
- API oficial da Meta: máxima estabilidade e confiabilidade
- Webhooks 100% confiáveis para recebimento de mensagens
- Envio de mensagens de texto livre (sem necessidade de templates pré-aprovados para sessões abertas)
- Infraestrutura robusta da Meta com SLA garantido
- Sem custos por mensagem no volume atual (taxa de plataforma aplicável)

**Credenciais do ambiente:**
- Phone Number ID: `674094992450703`
- Número do Bot: `+55 11 5286-8259`
- API Version: `v21.0`
- Webhook: `https://albino.app.n8n.cloud/webhook/whatsapp-meta`

**Alternativas descartadas:**
- Evolution API: não oficial, risco de bloqueio pela Meta
- Twilio WhatsApp: custo adicional por mensagem (~$0.005/msg)
- Telegram Bot API: menor adoção no Brasil

---

### 2.4 Interface de Monitoramento
**Ferramenta:** Chatwoot Cloud (Plano Hacker — gratuito)

**Justificativa:**
- Plataforma madura de helpdesk/live chat com todas as features necessárias:
  - Interface de conversas em tempo real
  - Histórico completo e busca
  - Dashboard com métricas e relatórios
  - Sistema de tags e notas privadas
  - **Intervenção manual**: gerente pode assumir conversa a qualquer momento
  - Múltiplos usuários/equipes
  - Mobile app (iOS/Android)
- API REST completa para integração com n8n
- Sem necessidade de deploy próprio (Cloud gerenciado)
- Reduz tempo de desenvolvimento de 7-10 dias para integração direta

**Limitações do Plano Hacker (gratuito):** 1 inbox e 2 agentes. Suficiente para o MVP atual (apenas 1 inbox utilizado, sem agentes humanos — todas as operações são feitas via API). Upgrade para plano pago necessário se houver necessidade de mais inboxes ou agentes.

**Alternativas descartadas:**
- Chatwoot self-hosted: heap out of memory no tier gratuito (512MB RAM insuficiente)
- Desenvolvimento custom (Next.js): 7-10 dias, alto custo
- n8n interface nativa: muito técnica para gerentes de qualidade

---

### 2.5 Banco de Dados
**Ferramenta:** Supabase (PostgreSQL gerenciado)

**Justificativa:**
- PostgreSQL confiável para dados estruturados
- API REST automática (gerada a partir do schema)
- Realtime subscriptions (útil para dashboard ao vivo)
- Autenticação e RLS (Row Level Security) integrados
- Tier gratuito generoso: até 500MB storage + 2GB bandwidth
- Backup automático e point-in-time recovery
- SDK JavaScript/Python para fácil integração

**Schema principal:**
```sql
-- Tabela de pesquisas de satisfação
CREATE TABLE satisfaction_surveys (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),

  -- Dados do cliente
  hubspot_contact_id VARCHAR NOT NULL,
  contact_name VARCHAR,
  contact_email VARCHAR,
  contact_phone VARCHAR,

  -- Contexto gerado pela IA
  context_summary TEXT,
  period_analyzed VARCHAR, -- "últimos 30 dias"

  -- Conversa
  conversation_id VARCHAR, -- ID do Chatwoot
  channel VARCHAR, -- "whatsapp", "telegram", "voice"

  -- Resultados
  satisfaction_score INTEGER CHECK (satisfaction_score >= 1 AND satisfaction_score <= 5),
  customer_feedback TEXT,
  conversation_transcript JSONB,

  -- Metadados
  sent_at TIMESTAMP WITH TIME ZONE,
  delivered_at TIMESTAMP WITH TIME ZONE,
  read_at TIMESTAMP WITH TIME ZONE,
  completed_at TIMESTAMP WITH TIME ZONE,

  -- Status tracking
  status VARCHAR DEFAULT 'pending', -- pending, sent, in_progress, completed, failed

  -- Logs
  error_log TEXT,
  execution_time_seconds INTEGER
);

-- Índices para performance
CREATE INDEX idx_surveys_contact ON satisfaction_surveys(hubspot_contact_id);
CREATE INDEX idx_surveys_status ON satisfaction_surveys(status);
CREATE INDEX idx_surveys_created ON satisfaction_surveys(created_at DESC);
```

---

## 3. Arquitetura de Agentes de IA

### Padrão Multi-Agent System
O sistema utiliza **4 agentes especializados** trabalhando em pipeline sequencial, cada um com uma responsabilidade clara.

### Agente 1: Data Fetcher (Coletor de Dados)
**Responsabilidade:** Buscar e consolidar histórico do cliente no HubSpot

**Dados coletados:**
- Informações básicas do contato (nome, email, telefone, empresa)
- Emails trocados nos últimos 30 dias
- Deals/negócios (criados, ganhos, perdidos)
- Tickets abertos (especialmente churn/downgrade)
- Notas e atividades registradas
- Produtos/serviços contratados
- Última interação

**Tecnologia:** Node HTTP Request via n8n + HubSpot API v3

**Output:** JSON estruturado com todos os dados relevantes

---

### Agente 2: Context Analyzer (Analisador de Contexto)
**Responsabilidade:** Analisar dados brutos e gerar insights focados em satisfação

**Processamento:**
- Resumir eventos importantes do período
- Identificar sinais de satisfação/insatisfação
- Destacar conquistas/marcos positivos
- Flaggar problemas ou reclamações
- Avaliar nível de engajamento
- Sugerir tom da mensagem (empático, celebratório, investigativo)

**Tecnologia:** Tess AI — Agente 2 (Context Analyzer, ID `38717`)

**Output:**
```json
{
  "summary": "Cliente contratou plano Pro há 20 dias, abriu 2 tickets de suporte que foram resolvidos em < 24h...",
  "sentiment": "positive",
  "key_events": ["upgrade_to_pro", "support_ticket_resolved"],
  "suggested_tone": "celebratory_and_curious",
  "red_flags": []
}
```

---

### Agente 3: Message Generator (Gerador de Mensagens)
**Responsabilidade:** Criar mensagem personalizada de abertura

**Características:**
- Tom conversacional e humano
- Personalização baseada no contexto específico
- Referências a eventos reais do período
- Call-to-action claro (responder sobre satisfação)
- Brevidade (máximo 3-4 linhas)

**Tecnologia:** Tess AI — Agente 3 (Message Generator, ID `38728`)

**Output:** Mensagem pronta para envio (texto)

**Exemplo:**
```
Olá João! 👋

Vi que você migrou para o plano Pro há 3 semanas e já está aproveitando as novas funcionalidades. Como tem sido sua experiência até agora?

Adoraria saber sua opinião! De 1 a 5, como você avalia nosso serviço?
```

---

### Agente 4: Conversation Handler (Gerenciador de Conversa)
**Responsabilidade:** Conduzir conversa bidirecional e extrair nota de satisfação

**Capacidades:**
- Responder perguntas de clarificação
- Aprofundar em pontos específicos (se cliente mencionar problema)
- Agradecer feedback positivo
- Solicitar nota de 1-5 de forma natural
- Detectar quando conversa foi concluída
- Saber quando escalar para humano (se solicitado)

**Tecnologia:** Tess AI — Agente 4 (Conversation Handler V2.0, ID `38733`)

**Output:**
- Nota de satisfação (1-5)
- Transcrição completa
- Insights adicionais do feedback

---

## 4. Fluxo de Dados End-to-End

```
┌─────────────────────────────────────────────────────────────────┐
│                      TRIGGER (Diário ou Manual)                  │
│         - Cron schedule (ex: todo dia 10h)                      │
│         - Webhook manual (lista de contatos)                    │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│            CONFIG NODE (Configuração Centralizada)              │
│                                                                  │
│  Define variáveis usadas no workflow:                           │
│  - CHATWOOT_ACCOUNT_ID: ID da conta Chatwoot                   │
│  - CHATWOOT_INBOX_ID: ID do inbox                              │
│  - META_PHONE_NUMBER_ID: ID do número WhatsApp Business        │
│                                                                  │
│  Padrão de acesso: $node["Config"].json.VARIABLE_NAME          │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│              1. AGENTE DATA FETCHER (n8n workflow)              │
│                                                                  │
│  ┌──────────────┐    API     ┌─────────────────────┐           │
│  │ Lista de     │──────────→ │  HubSpot API v3     │           │
│  │ Contatos     │            │  - GET contact       │           │
│  └──────────────┘            │  - GET emails        │           │
│                              │  - GET deals         │           │
│                              │  - GET tickets       │           │
│                              │  - GET notes         │           │
│                              └──────────┬──────────┘           │
│                                         │                       │
│                                         ↓ JSON                  │
│                              ┌──────────────────────┐           │
│                              │ Raw Customer Data    │           │
│                              └──────────────────────┘           │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│           2. AGENTE CONTEXT ANALYZER (Tess AI — Agent 2)       │
│                                                                  │
│  Input: Raw customer data (JSON, até 20kb)                     │
│                                                                  │
│  Prompt System:                                                 │
│  "Você é um analista de relacionamento com clientes.           │
│   Analise o histórico e gere insights focados em satisfação."  │
│                                                                  │
│  Output:                                                        │
│  {                                                              │
│    summary: "...",                                              │
│    sentiment: "positive/neutral/negative",                      │
│    key_events: [...],                                           │
│    suggested_tone: "..."                                        │
│  }                                                              │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│        3. AGENTE MESSAGE GENERATOR (Tess AI — Agent 3)         │
│                                                                  │
│  Input: Context analysis + contact name                         │
│                                                                  │
│  Prompt System:                                                 │
│  "Crie uma mensagem personalizada, empática e concisa           │
│   para iniciar conversa sobre satisfação."                      │
│                                                                  │
│  Output: "Olá {nome}! [contexto personalizado]...              │
│           Como você avalia nosso serviço de 1 a 5?"            │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│         4. ENVIO VIA META WHATSAPP API (oficial)                │
│                                                                  │
│  POST /v21.0/{phone_number_id}/messages                         │
│  {                                                              │
│    "messaging_product": "whatsapp",                             │
│    "recipient_type": "individual",                              │
│    "to": "5521981444992",                                       │
│    "type": "text",                                              │
│    "text": { "body": "[mensagem gerada]" }                     │
│  }                                                              │
│                                                                  │
│  ✓ Mensagem enviada                                            │
│  ✓ Webhook dispara na resposta do cliente                      │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│                 5. REGISTRO NO CHATWOOT + SUPABASE             │
│                                                                  │
│  Chatwoot:                                                      │
│  - Cria nova conversa                                           │
│  - Adiciona mensagem enviada                                    │
│  - Atribui tag "satisfacao-survey"                             │
│                                                                  │
│  Supabase:                                                      │
│  INSERT INTO satisfaction_surveys                               │
│  (contact_id, context_summary, sent_at, status: 'sent')        │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓ (cliente responde)
┌─────────────────────────────────────────────────────────────────┐
│          6. WEBHOOK RECEBE RESPOSTA (Meta WhatsApp API)         │
│                                                                  │
│  POST /webhook/whatsapp-meta                                    │
│  {                                                              │
│    "entry": [{                                                  │
│      "changes": [{                                              │
│        "value": {                                               │
│          "messages": [{                                         │
│            "from": "5521981444992",                             │
│            "text": { "body": "Muito bom! Dou nota 5!" }       │
│          }]                                                     │
│        }                                                        │
│      }]                                                         │
│    }]                                                           │
│  }                                                              │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│      7. AGENTE CONVERSATION HANDLER (Tess AI — Agent 4 V2.0)   │
│                                                                  │
│  Input:                                                         │
│  - Contexto anterior (histórico da conversa)                   │
│  - Nova mensagem do cliente                                     │
│                                                                  │
│  Lógica:                                                        │
│  - Se mensagem contém nota (1-5): extrair e agradecer          │
│  - Se mensagem tem feedback: aprofundar ou agradecer           │
│  - Se mensagem é dúvida: responder e voltar ao objetivo        │
│  - Se cliente pede humano: notificar gerente (Chatwoot)        │
│                                                                  │
│  Output:                                                        │
│  - Resposta para enviar                                        │
│  - Status: "continue" | "completed" | "escalate"               │
│  - Extracted data: { score: 5, feedback: "..." }              │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│            8. LOOP: Enviar resposta + aguardar nova             │
│                                                                  │
│  Repete steps 4-7 até status = "completed" ou "escalate"       │
│                                                                  │
│  Max rodadas: 5 (evita loops infinitos)                        │
│  Timeout: 24h de inatividade = marca como "incomplete"         │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓ (quando completed)
┌─────────────────────────────────────────────────────────────────┐
│               9. FINALIZAÇÃO E REGISTRO                         │
│                                                                  │
│  Supabase:                                                      │
│  UPDATE satisfaction_surveys                                    │
│  SET                                                            │
│    satisfaction_score = 5,                                      │
│    customer_feedback = "...",                                   │
│    conversation_transcript = [...],                             │
│    completed_at = NOW(),                                        │
│    status = 'completed'                                         │
│                                                                  │
│  Chatwoot:                                                      │
│  - Marca conversa como "resolvida"                             │
│  - Adiciona nota privada com score                             │
│                                                                  │
│  HubSpot (opcional):                                            │
│  - Atualiza custom property "ultima_pesquisa_satisfacao"       │
│  - Cria nota com resultado                                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. Interface de Monitoramento (Chatwoot)

### 5.1 Funcionalidades para Gerentes de Qualidade

**Dashboard Principal:**
- Lista de conversas ativas/pendentes/concluídas
- Filtros por status, data, score
- Busca por nome do cliente
- Métricas em tempo real:
  - Total de pesquisas enviadas (hoje/semana/mês)
  - Taxa de resposta
  - Distribuição de scores (gráfico)
  - Tempo médio de resposta

**Visualização de Conversa Individual:**
- Histórico completo da conversa
- Contexto do HubSpot (summary gerado pela IA)
- Status: enviado/lido/respondendo/concluído
- Score de satisfação (destaque visual)
- Botão "Assumir conversa" (intervenção manual)

**Intervenção Manual:**
- Gerente pode assumir conversa a qualquer momento
- Gerente vê todo o contexto antes de assumir
- **Nota:** a IA não para automaticamente quando o gerente intervém — o bot continua processando mensagens do cliente via webhook. Parar a IA na intervenção é uma proposta futura (ver doc 03, Propostas Futuras item 4)

**Relatórios:**
- Export CSV com todas as pesquisas
- Gráficos de tendência (score ao longo do tempo)
- Comparação por produto/segmento
- Identificação de clientes críticos (score 1-2)

### 5.2 Configuração Técnica

**Integração n8n ↔ Chatwoot:**

```javascript
// n8n node: Chatwoot - Create Conversation
POST https://app.chatwoot.com/api/v1/accounts/{{$node["Config"].json.CHATWOOT_ACCOUNT_ID}}/conversations
Headers: {
  "api_access_token": "{CHATWOOT_API_TOKEN}"
}
Body: {
  "inbox_id": {{$node["Config"].json.CHATWOOT_INBOX_ID}},
  "contact_id": "{contact_id}",
  "source_id": "{customer_phone}",
  "additional_attributes": {
    "hubspot_contact_id": "{hubspot_id}",
    "context_summary": "{context_summary}",
    "survey_id": "{survey_id}"
  }
}
```

**📝 Nota:** Os IDs do Chatwoot são gerenciados centralmente via **node "Config"** (Set node). O Config define `CHATWOOT_ACCOUNT_ID`, `CHATWOOT_INBOX_ID` e `META_PHONE_NUMBER_ID`, facilitando a portabilidade do workflow entre ambientes — basta alterar os valores no node Config.

**Sincronização de mensagens:**
- Webhook Meta → n8n → Chatwoot API
- Cada mensagem do cliente registrada como `incoming`
- Cada resposta do bot registrada como `outgoing`
- Nota privada automática com score/sentiment/feedback quando survey é concluído
- Conversa marcada como resolvida automaticamente após conclusão

---

## 6. Infraestrutura e Deploy

### 6.1 Serviços Utilizados (todos Cloud/gerenciados)

| Serviço | Plano | Custo (Piloto) | Custo (Produção) |
|---------|-------|----------------|------------------|
| n8n Cloud | Trial 14d → Starter | $0 | R$150/mês |
| Supabase | Free tier | $0 | $0 |
| Chatwoot Cloud | Hacker → Pro | $0 | R$105/mês |
| Tess AI | Créditos | variável | ~R$165/mês |
| Meta WhatsApp API | Plataforma | variável | ~R$30/mês |
| HubSpot | Existente | $0 adicional | $0 adicional |

**Vantagem:** Zero servidores próprios. Todo o sistema opera em serviços gerenciados na nuvem.

**Nota:** n8n Cloud oferece 14 dias de trial gratuito, ideal para validação do MVP. Após o trial, o plano mínimo é o Starter (R$150/mês).

### 6.2 Credenciais no n8n

As credenciais são configuradas diretamente no n8n Cloud:
- **Chatwoot Cloud API** — Header Auth com `api_access_token`
- **Tess AI** — HTTP Request com bearer token
- **Supabase** — Postgres Execute com connection string
- **Meta WhatsApp API** — HTTP Request com token de acesso permanente
- **HubSpot** — HTTP Request com API key

### 6.3 Webhook Meta → n8n

```
URL: https://albino.app.n8n.cloud/webhook/whatsapp-meta
Verify Token: satisfaction-survey-2026
Subscription: messages
Status: Verificado e ativo
```

---

## 7. Segurança e Compliance

### 7.1 Dados Sensíveis
- Dados de clientes do HubSpot (LGPD/GDPR)
- Conversas privadas via WhatsApp
- Informações comerciais (deals, valores)

### 7.2 Medidas de Segurança

**Criptografia:**
- TLS/SSL em todas as comunicações (HTTPS)
- Banco de dados com encryption at rest
- Variáveis de ambiente para secrets (nunca hardcoded)

**Acesso:**
- n8n protegido com autenticação básica + IP whitelist (opcional)
- Chatwoot com SSO/SAML (se disponível) ou senhas fortes
- API tokens com rotação periódica (90 dias)
- Logs de acesso e auditoria

**LGPD Compliance:**
- Consentimento: cliente pode solicitar exclusão de dados
- Minimização: apenas dados necessários são coletados
- Transparência: cliente sabe que está falando com IA
- Retenção: dados deletados após 90 dias (configurável)

**Backup:**
- Backup diário do PostgreSQL (7 dias de retenção)
- Export semanal para storage externo (S3/GCS)
- Testes de restore mensais

---

## 8. Monitoramento e Observabilidade

### 8.1 Métricas de Negócio (Chatwoot Dashboard)
- Total de pesquisas enviadas/dia
- Taxa de resposta (% clientes que responderam)
- Distribuição de scores (1-5)
- NPS calculado automaticamente
- Tempo médio de conversa
- Taxa de escalação para humano

### 8.2 Métricas Técnicas (n8n + Logs)
- Taxa de sucesso de workflows (%)
- Latência média de cada agente
- Erros de API (HubSpot, Meta WhatsApp, Tess AI)
- Uptime dos serviços
- Uso de créditos Tess AI (controle de custo)

### 8.3 Alertas *(não implementado — proposta futura)*
Alertas automáticos não existem no MVP atual. Possibilidades para implementação futura:
- Email/Slack se taxa de erro > 5%
- Notificação se cliente dá score 1-2
- Alerta se webhook Meta parar de funcionar
- Warning se créditos Tess AI ficarem baixos

---

## 9. Escalabilidade

### Capacidade Inicial (Piloto)
- 50-100 pesquisas/dia
- ~200 mensagens/dia (incluindo respostas)
- 1 gerente de qualidade monitorando

### Escalabilidade Técnica
- n8n Cloud: suporta 1000+ execuções/dia
- Meta WhatsApp API: rate limit alto, suporta múltiplos números de negócio
- Tess AI: escalabilidade gerenciada pela plataforma
- Chatwoot Cloud: suporta 10k+ conversas sem degradação
- Supabase: escala automática do banco de dados

### Plano de Escala (se piloto funcionar)
1. Adicionar mais números WhatsApp no Meta Business
2. Upgrade Supabase para Pro ($25/mês)
3. Upgrade n8n Cloud para plano com mais execuções
4. Implementar cache para dados do HubSpot
5. Dashboard analytics via Grafana + Supabase

---

## 10. Estimativa de Custos (Mensal)

### Piloto (50 pesquisas durante trial de 14 dias)

| Item | Custo |
|------|-------|
| n8n Cloud (Trial 14 dias) | R$0 |
| Supabase (free tier) | R$0 |
| Chatwoot Cloud (Hacker) | R$0 |
| Tess AI (créditos) | ~R$30 |
| Meta WhatsApp API | ~R$15 |
| **Total (14 dias trial)** | **~R$45** |

### Produção (200 pesquisas/mês)

| Item | Custo |
|------|-------|
| n8n Cloud (Starter) | R$150 |
| Supabase (free tier) | R$0 |
| Chatwoot Cloud (Pro) | R$105 |
| Tess AI (créditos) | ~R$165 |
| Meta WhatsApp API | ~R$30 |
| **Total** | **~R$450/mês** |

**Nota sobre n8n Cloud:** Trial de 14 dias gratuito permite validar o MVP sem custo. Após o trial, o plano mínimo é o Starter (R$150/mês). Todos os serviços são gerenciados na nuvem — sem servidores próprios ou VPS necessários.

---

## 11. Diferenciais Competitivos desta Arquitetura

1. **Cloud-first:** Zero servidores próprios, custo operacional mínimo (~$6-11/mês piloto)
2. **Multi-agent pattern:** 4 agentes especializados via Tess AI = melhor qualidade e manutenibilidade
3. **API oficial da Meta:** WhatsApp Business API oficial — máxima estabilidade e sem risco de bloqueio
4. **Rastreabilidade completa:** Histórico em PostgreSQL (Supabase) + monitoramento em tempo real (Chatwoot)
5. **Interface profissional:** Chatwoot Cloud = UX enterprise com intervenção manual pelo gerente
6. **Escalável:** De 50 para 1000 pesquisas/dia sem reescrever código
7. **Flexível:** Fácil adicionar novos canais (SMS, Telegram, voz)
8. **Manutenível:** Visual workflow (n8n) + prompts versionados (git)

---

## Próximos Passos

1. ✅ Arquitetura definida e implementada
2. ✅ Fluxogramas AS-IS e TO-BE criados
3. ✅ Prompts dos 4 agentes desenvolvidos e otimizados
4. ✅ Workflow n8n funcional (FLUXO 1 + FLUXO 2)
5. ✅ Ambiente configurado (Meta API, Chatwoot Cloud, Supabase)
6. ✅ Pesquisa piloto executada e validada end-to-end
7. ✅ Resultados e ROI documentados
8. ⏳ Dashboard analytics (Fase 3 — opcional)

---

**Documento elaborado para:** Case Agent Dev - FMA/Pareto/IA Leader
**Data:** Fevereiro 2026
**Versão:** 2.0
