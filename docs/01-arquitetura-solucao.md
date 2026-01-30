# Arquitetura da Solução - Sistema de Satisfação de Clientes com IA

## 1. Visão Geral

Sistema automatizado end-to-end para medição de satisfação de clientes através de agentes de IA conversacionais, integrando dados do HubSpot CRM, processamento com LLMs e canais de mensageria.

### Objetivo
Reduzir de **30 minutos para ~2 minutos** o processo de análise e contato com cada cliente, garantindo personalização, consistência e rastreabilidade completa.

---

## 2. Stack Tecnológica

### 2.1 Orquestração e Automação
**Ferramenta:** n8n (Open Source)

**Justificativa:**
- Open-source: redução de custos operacionais (ROI +40%)
- Self-hosted: controle total sobre dados sensíveis do CRM
- Integração nativa com HubSpot, APIs de IA e webhooks
- Suporte a código customizado (JavaScript/Python) para lógica complexa
- Interface visual para manutenção futura pela equipe técnica
- Sistema robusto de logs e retry em caso de falhas
- Comunidade ativa com templates prontos

**Alternativas descartadas:**
- Make: Custo ~$300/mês em escala (vs $0 n8n self-hosted)
- Zapier: Limitado para lógica multi-step complexa

---

### 2.2 Motor de IA (LLM)
**Ferramenta:** Claude 3.5 Sonnet (Anthropic API)

**Justificativa:**
- Context window de 200k tokens: analisa histórico completo de 30 dias sem truncar
- Excelente em conversas empáticas e naturais (essencial para satisfação)
- Seguimento preciso de instruções estruturadas (system prompts)
- Custo-benefício: ~$3 por milhão de tokens de input vs $10 GPT-4
- Latência baixa (~2-3s para respostas complexas)
- API estável e confiável

**Configuração sugerida:**
- Model: `claude-3-5-sonnet-20241022`
- Temperature: 0.7 (equilíbrio entre criatividade e consistência)
- Max tokens: 1024 (mensagens concisas)

**Alternativa:** GPT-4o (custo maior, mas bom fallback)

---

### 2.3 Canal de Mensageria
**Ferramenta:** Evolution API + WhatsApp Business

**Justificativa:**
- Evolution API: open-source, self-hosted, gratuita
- Integração completa com WhatsApp (não oficial mas estável)
- Suporte a webhooks bidirecionais (envio + recebimento)
- Rastreamento de status: enviado, entregue, lido
- Permite múltiplas instâncias (escalabilidade)
- API REST simples para integração com n8n

**Alternativas consideradas:**
- Twilio WhatsApp Business API: custo ~$0.005/msg + setup complexo
- Telegram Bot API: gratuito, mas menor adoção no Brasil
- SMS via Twilio: custo alto (~$0.02/SMS)

**Para voz (opcional):**
- Vapi.ai ou Bland.ai: APIs de conversação por voz com IA
- Custo: ~$0.10-0.15/minuto

---

### 2.4 Interface de Monitoramento
**Ferramenta:** Chatwoot (Open Source)

**Justificativa:**
- Plataforma madura de helpdesk/live chat com todas as features necessárias:
  - Interface de conversas em tempo real
  - Histórico completo e busca
  - Dashboard com métricas e relatórios
  - Sistema de tags e notas
  - **Intervenção manual**: gerente pode assumir conversa
  - Múltiplos usuários/equipes
  - Mobile app (iOS/Android)
- Self-hosted: $0 de custo
- API REST completa para integração com n8n
- Reduz tempo de desenvolvimento de 7-10 dias para 2-3 dias

**Alternativas descartadas:**
- Desenvolvimento custom (Next.js): 7-10 dias, alto custo
- Appsmith/Budibase: requer desenvolvimento, sem features de chat
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

**Tecnologia:** Claude 3.5 Sonnet com prompt especializado

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

**Tecnologia:** Claude 3.5 Sonnet com few-shot examples

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

**Tecnologia:** Claude 3.5 Sonnet em modo conversacional (mantém contexto)

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
│           2. AGENTE CONTEXT ANALYZER (Claude 3.5 Sonnet)        │
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
│        3. AGENTE MESSAGE GENERATOR (Claude 3.5 Sonnet)          │
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
│              4. ENVIO VIA EVOLUTION API (WhatsApp)              │
│                                                                  │
│  POST /message/sendText                                         │
│  {                                                              │
│    "number": "5511999999999",                                   │
│    "textMessage": {                                             │
│      "text": "[mensagem gerada]"                                │
│    }                                                            │
│  }                                                              │
│                                                                  │
│  ✓ Mensagem enviada                                            │
│  ✓ Status: delivered / read (webhook)                          │
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
│           6. WEBHOOK RECEBE RESPOSTA (Evolution API)            │
│                                                                  │
│  POST /webhook/n8n-receiver                                     │
│  {                                                              │
│    "event": "messages.upsert",                                  │
│    "data": {                                                    │
│      "key": { "remoteJid": "5511999999999@s.whatsapp.net" },   │
│      "message": { "conversation": "Muito bom! Dou nota 5!" }   │
│    }                                                            │
│  }                                                              │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│      7. AGENTE CONVERSATION HANDLER (Claude 3.5 Sonnet)         │
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
- IA detecta e para de responder automaticamente
- Gerente vê todo o contexto antes de assumir
- Pode retornar para IA depois (se desejar)

**Relatórios:**
- Export CSV com todas as pesquisas
- Gráficos de tendência (score ao longo do tempo)
- Comparação por produto/segmento
- Identificação de clientes críticos (score 1-2)

### 5.2 Configuração Técnica

**Integração n8n ↔ Chatwoot:**

```javascript
// n8n node: Chatwoot - Create Conversation
POST https://chatwoot.seudominio.com/api/v1/accounts/{account_id}/conversations
Headers: {
  "api_access_token": "{{$env.CHATWOOT_API_TOKEN}}"
}
Body: {
  "inbox_id": 1,
  "contact_id": "{{$node.CreateContact.json.id}}",
  "additional_attributes": {
    "hubspot_contact_id": "{{$json.hubspot_id}}",
    "context_summary": "{{$json.context.summary}}",
    "survey_id": "{{$json.survey_id}}"
  }
}
```

**Sincronização de mensagens:**
- Webhook do Evolution API → n8n → Chatwoot API
- Cada mensagem (enviada/recebida) registrada no Chatwoot
- Tags automáticas: `ai-agent`, `satisfacao`, `score-{1-5}`

---

## 6. Infraestrutura e Deploy

### 6.1 Servidores Necessários

**Opção 1: VPS Único (Recomendado para piloto)**
- Provedor: DigitalOcean / Hetzner / Contabo
- Specs: 4 vCPU, 8GB RAM, 160GB SSD
- Custo: ~$20-40/mês
- Serviços instalados:
  - n8n (Docker)
  - Evolution API (Docker)
  - Chatwoot (Docker)
  - PostgreSQL (Docker)
  - Nginx (reverse proxy)

**Opção 2: Serviços Gerenciados (Produção)**
- n8n: n8n Cloud ($50/mês) ou self-hosted
- Supabase: Tier gratuito ou Pro ($25/mês)
- Evolution API: Self-hosted
- Chatwoot: Self-hosted ou Cloud ($19/mês)
- Custo total: ~$50-100/mês

### 6.2 Docker Compose Setup

```yaml
# docker-compose.yml (simplificado)
version: '3.8'

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: n8n
      POSTGRES_USER: n8n
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data

  n8n:
    image: n8nio/n8n:latest
    ports:
      - "5678:5678"
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - N8N_BASIC_AUTH_USER=${N8N_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_PASSWORD}
      - WEBHOOK_URL=https://n8n.seudominio.com/
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      - postgres

  evolution-api:
    image: atendai/evolution-api:latest
    ports:
      - "8080:8080"
    environment:
      - DATABASE_ENABLED=true
      - DATABASE_PROVIDER=postgresql
      - DATABASE_CONNECTION_URI=postgresql://postgres:${DB_PASSWORD}@postgres:5432/evolution
    volumes:
      - evolution_data:/evolution/instances

  chatwoot:
    image: chatwoot/chatwoot:latest
    ports:
      - "3000:3000"
    environment:
      - POSTGRES_HOST=postgres
      - POSTGRES_DATABASE=chatwoot
      - POSTGRES_USERNAME=chatwoot
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - REDIS_URL=redis://redis:6379
      - SECRET_KEY_BASE=${CHATWOOT_SECRET}
    depends_on:
      - postgres
      - redis

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  n8n_data:
  evolution_data:
  redis_data:
```

### 6.3 Variáveis de Ambiente Necessárias

```bash
# .env
DB_PASSWORD=senha_segura_aqui
N8N_USER=admin
N8N_PASSWORD=senha_n8n
CHATWOOT_SECRET=$(openssl rand -hex 64)

# APIs externas
HUBSPOT_API_KEY=pat-na-...
ANTHROPIC_API_KEY=sk-ant-...
SUPABASE_URL=https://xxx.supabase.co
SUPABASE_ANON_KEY=eyJ...
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
- Erros de API (HubSpot, Claude, Evolution)
- Uptime dos serviços
- Uso de tokens da API Claude (controle de custo)

### 8.3 Alertas
- Email/Slack se taxa de erro > 5%
- Notificação se cliente dá score 1-2 (NPS detractor)
- Alerta se Evolution API cair (WhatsApp offline)
- Warning se uso de tokens Claude exceder budget diário

---

## 9. Escalabilidade

### Capacidade Inicial (Piloto)
- 50-100 pesquisas/dia
- ~200 mensagens/dia (incluindo respostas)
- 1 gerente de qualidade monitorando

### Escalabilidade Técnica
- n8n: suporta 1000+ execuções/dia em VPS básica
- Evolution API: múltiplas instâncias do WhatsApp (1 por número)
- Claude API: rate limit 50 req/min (3000/hora)
- Chatwoot: suporta 10k+ conversas sem degradação

### Plano de Escala (se piloto funcionar)
1. Adicionar mais números de WhatsApp (Evolution multi-instance)
2. Upgrade VPS (8 vCPU, 16GB RAM)
3. Separar serviços em VMs dedicadas
4. Implementar cache (Redis) para dados do HubSpot
5. Load balancer para n8n (múltiplos workers)

---

## 10. Estimativa de Custos (Mensal)

### Piloto (50 pesquisas/dia)

| Item | Custo |
|------|-------|
| VPS (4vCPU, 8GB) | $30 |
| Claude API (150k tokens/dia) | $15 |
| Domínio + SSL | $2 |
| Backup (S3) | $3 |
| **Total** | **$50/mês** |

### Produção (200 pesquisas/dia)

| Item | Custo |
|------|-------|
| VPS (8vCPU, 16GB) | $60 |
| Claude API (600k tokens/dia) | $60 |
| Supabase Pro | $25 |
| Domínio + SSL | $2 |
| Backup + CDN | $10 |
| **Total** | **$157/mês** |

**Nota:** Twilio WhatsApp seria +$30-60/mês. Evolution API = $0.

---

## 11. Diferenciais Competitivos desta Arquitetura

1. **Open-source first:** Reduz custos em 70% vs soluções proprietárias
2. **Multi-agent pattern:** Cada agente especializado = melhor qualidade
3. **Rastreabilidade completa:** Tudo registrado (compliance + insights)
4. **Interface profissional:** Chatwoot = UX enterprise sem custo de dev
5. **Escalável:** De 50 para 1000 pesquisas/dia sem reescrever código
6. **Flexível:** Fácil adicionar novos canais (SMS, Telegram, voz)
7. **Manutenível:** Visual workflow (n8n) + código versionado (git)

---

## Próximos Passos

1. ✅ Arquitetura definida
2. ⏳ Criar fluxogramas AS-IS e TO-BE
3. ⏳ Desenvolver prompts dos 4 agentes
4. ⏳ Implementar workflow n8n funcional
5. ⏳ Configurar ambiente de teste
6. ⏳ Executar pesquisa piloto com 1 cliente
7. ⏳ Documentar resultados e ROI

---

**Documento elaborado para:** Case Agent Dev - FMA/Pareto/IA Leader
**Data:** Janeiro 2026
**Versão:** 1.0
