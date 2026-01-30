# Guia de Deploy Online - Demo Completo

**Projeto:** Sistema de Satisfação com IA
**Objetivo:** Deploy completo em cloud gratuito/barato para demo funcional
**Público:** Avaliadores do desafio FMA + desenvolvedores
**Tempo estimado:** 2-3 horas

---

## 🎯 Visão Geral

Este guia configura um **ambiente 100% funcional na nuvem** onde avaliadores podem testar o sistema completo sem instalar nada localmente.

### Stack do Demo Online:
- ☁️ **n8n Cloud** (free tier - 5,000 executions/mês)
- ☁️ **Supabase Cloud** (free tier - 500MB database)
- ☁️ **Evolution API na Render** (free tier com sleep após inatividade)
- ☁️ **Anthropic API** (trial $5 grátis ou pay-as-you-go)
- ☁️ **HubSpot Sandbox** (gratuito para desenvolvimento)

### Custos Estimados:
| Serviço | Plano | Custo/mês |
|---------|-------|-----------|
| n8n Cloud | Free tier | $0 |
| Supabase | Free tier | $0 |
| Evolution API (Render) | Free tier* | $0 |
| Anthropic API | Pay-as-you-go | ~$1-5 (demos) |
| HubSpot | Developer Sandbox | $0 |
| **TOTAL** | | **~$1-5/mês** |

*Free tier da Render hiberna após 15min de inatividade (cold start de ~30s)

---

## 📋 Pré-requisitos

Antes de começar, tenha em mãos:
- [ ] Email para criar contas
- [ ] Número de telefone para WhatsApp (seu número pessoal serve)
- [ ] Cartão de crédito (para Anthropic API - trial grátis disponível)
- [ ] Navegador moderno (Chrome/Firefox recomendado)

---

## 🚀 Parte 1: Setup do Supabase Cloud

### 1.1. Criar Conta

1. Acesse https://supabase.com
2. Clique em **"Start your project"**
3. Faça login com GitHub, Google ou Email
4. Crie uma nova organização (pode ser seu nome)

### 1.2. Criar Projeto

1. Clique em **"New Project"**
2. Preencha:
   - **Name:** `case-agent-satisfaction`
   - **Database Password:** Gere uma senha forte (salve!)
   - **Region:** `South America (São Paulo)` (mais próximo)
   - **Pricing Plan:** `Free`
3. Clique em **"Create new project"**
4. ⏱️ Aguarde 2-3 minutos (provisionamento do banco)

### 1.3. Criar Tabela `satisfaction_surveys`

1. No menu lateral, clique em **"Table Editor"**
2. Clique em **"Create a new table"**
3. Clique em **"Import data via spreadsheet"** → **Cancel** (vamos usar SQL)
4. Clique em **"SQL Editor"** no menu lateral
5. Clique em **"New query"**
6. Cole o seguinte SQL:

```sql
-- Criar extensão para UUID (se não existir)
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Criar tabela de pesquisas de satisfação
CREATE TABLE satisfaction_surveys (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  hubspot_contact_id VARCHAR(255) NOT NULL,
  contact_name VARCHAR(500),
  contact_phone VARCHAR(50),
  context_summary TEXT,
  sentiment VARCHAR(50),
  conversation_id VARCHAR(255),
  satisfaction_score INTEGER CHECK (satisfaction_score >= 1 AND satisfaction_score <= 5),
  customer_feedback TEXT,
  conversation_transcript JSONB,
  status VARCHAR(50) DEFAULT 'pending',
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  completed_at TIMESTAMP WITH TIME ZONE
);

-- Criar índices para performance
CREATE INDEX idx_surveys_contact_id ON satisfaction_surveys(hubspot_contact_id);
CREATE INDEX idx_surveys_status ON satisfaction_surveys(status);
CREATE INDEX idx_surveys_phone ON satisfaction_surveys(contact_phone);
CREATE INDEX idx_surveys_created_at ON satisfaction_surveys(created_at DESC);

-- Criar função para atualizar updated_at automaticamente
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Criar trigger para updated_at
CREATE TRIGGER update_satisfaction_surveys_updated_at
    BEFORE UPDATE ON satisfaction_surveys
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

-- Comentários para documentação
COMMENT ON TABLE satisfaction_surveys IS 'Armazena pesquisas de satisfação com histórico de conversas';
COMMENT ON COLUMN satisfaction_surveys.status IS 'pending | ongoing | completed | escalate | timeout';
COMMENT ON COLUMN satisfaction_surveys.sentiment IS 'positive | neutral | negative | very_positive | very_negative';
```

7. Clique em **"Run"** (ou `Ctrl + Enter`)
8. ✅ Você deve ver: `Success. No rows returned`

### 1.4. Desabilitar RLS (Row Level Security) para Demo

Para facilitar o acesso do n8n sem configurar políticas complexas:

```sql
-- Desabilitar RLS temporariamente (apenas para demo)
ALTER TABLE satisfaction_surveys DISABLE ROW LEVEL SECURITY;
```

⚠️ **Nota de Segurança:** Em produção, você configuraria políticas RLS. Para demo, estamos simplificando.

### 1.5. Obter Credenciais

1. Clique em **"Settings"** (ícone de engrenagem) → **"API"**
2. **Copie e salve:**
   - **Project URL:** `https://xxxxxxx.supabase.co`
   - **anon/public key:** `eyJhbGc...` (chave longa)
   - **service_role key:** `eyJhbGc...` (chave secreta - NUNCA compartilhe)

3. Para n8n, usaremos a **service_role key** (bypass RLS)

### 1.6. Testar Conexão

Teste rapidamente no SQL Editor:

```sql
-- Inserir registro de teste
INSERT INTO satisfaction_surveys (
  hubspot_contact_id,
  contact_name,
  contact_phone,
  status
) VALUES (
  'test-123',
  'João da Silva',
  '5511999999999',
  'pending'
);

-- Verificar se foi criado
SELECT * FROM satisfaction_surveys;
```

✅ Você deve ver o registro criado. Pode deletar depois:
```sql
DELETE FROM satisfaction_surveys WHERE hubspot_contact_id = 'test-123';
```

---

## 🚀 Parte 2: Setup do Evolution API na Render

### 2.1. Criar Conta na Render

1. Acesse https://render.com
2. Clique em **"Get Started for Free"**
3. Faça login com GitHub (recomendado - deploy mais fácil)

### 2.2. Deploy da Evolution API

**Opção A: Deploy via Blueprint (Mais Rápido)**

1. Clique neste link de deploy:
   ```
   https://github.com/EvolutionAPI/evolution-api
   ```

2. No repositório, procure por **"Deploy to Render"** button (se disponível)

**Opção B: Deploy Manual (Mais Controle)**

1. No dashboard da Render, clique em **"New +"** → **"Web Service"**
2. Conecte seu GitHub ou escolha **"Public Git repository"**
3. Cole a URL: `https://github.com/EvolutionAPI/evolution-api`
4. Configure:
   - **Name:** `evolution-api-demo`
   - **Region:** `Oregon (US West)` (free tier disponível)
   - **Branch:** `main`
   - **Runtime:** `Docker`
   - **Instance Type:** `Free`

5. **Environment Variables** (clique em "Advanced"):

```bash
# API Configuration
SERVER_URL=https://evolution-api-demo.onrender.com
AUTHENTICATION_API_KEY=sua-chave-secreta-aqui-123456

# Database (SQLite - simples para demo)
DATABASE_ENABLED=false
DATABASE_PROVIDER=sqlite

# Storage (local - simples para demo)
S3_ENABLED=false

# Webhook Configuration
WEBHOOK_GLOBAL_ENABLED=true
WEBHOOK_EVENTS_APPLICATION_STARTUP=false
WEBHOOK_EVENTS_QRCODE_UPDATED=true
WEBHOOK_EVENTS_MESSAGES_SET=true
WEBHOOK_EVENTS_MESSAGES_UPSERT=true
WEBHOOK_EVENTS_MESSAGES_UPDATE=true
WEBHOOK_EVENTS_SEND_MESSAGE=true

# Logs
LOG_LEVEL=ERROR
LOG_COLOR=false

# Outras configurações
DEL_INSTANCE=false
```

⚠️ **IMPORTANTE:** Troque `sua-chave-secreta-aqui-123456` por uma chave forte aleatória.

6. Clique em **"Create Web Service"**
7. ⏱️ Aguarde 5-10 minutos (primeiro build é lento)

### 2.3. Aguardar Deploy Completar

1. Acompanhe os logs em tempo real
2. Quando ver `Your service is live 🎉`, está pronto
3. A URL será: `https://evolution-api-demo.onrender.com`

### 2.4. Conectar WhatsApp

**Via Postman ou cURL:**

```bash
# 1. Criar instância
curl -X POST https://evolution-api-demo.onrender.com/instance/create \
  -H "Content-Type: application/json" \
  -H "apikey: sua-chave-secreta-aqui-123456" \
  -d '{
    "instanceName": "satisfaction-survey",
    "qrcode": true,
    "integration": "WHATSAPP-BAILEYS"
  }'
```

**Resposta esperada:**
```json
{
  "instance": {
    "instanceName": "satisfaction-survey",
    "status": "created"
  },
  "qrcode": {
    "code": "2@xxxxx...",
    "base64": "data:image/png;base64,..."
  }
}
```

**2. Conectar WhatsApp:**

```bash
# Obter QR Code
curl -X GET https://evolution-api-demo.onrender.com/instance/connect/satisfaction-survey \
  -H "apikey: sua-chave-secreta-aqui-123456"
```

1. Copie o campo `base64` da resposta
2. Cole em: https://base64.guru/converter/decode/image
3. Escaneie o QR Code com seu WhatsApp (Configurações → Aparelhos conectados)
4. ✅ WhatsApp conectado!

**3. Verificar Status:**

```bash
curl -X GET https://evolution-api-demo.onrender.com/instance/fetchInstances \
  -H "apikey: sua-chave-secreta-aqui-123456"
```

Você deve ver `"state": "open"` quando conectado.

### 2.5. Configurar Webhook para n8n

Faremos isso depois de criar o workflow n8n, mas anote o endpoint:
```
https://evolution-api-demo.onrender.com/webhook/set/satisfaction-survey
```

---

## 🚀 Parte 3: Setup do n8n Cloud

### 3.1. Criar Conta

1. Acesse https://n8n.io
2. Clique em **"Start for free"**
3. Preencha:
   - Email
   - Nome
   - Senha
4. **Selecione:** `Cloud` (não "Self-hosted")
5. Crie uma workspace (ex: `case-agent-dev`)

### 3.2. Inicializar Workspace

1. ✅ n8n vai criar seu ambiente em ~30 segundos
2. Você será redirecionado para o editor
3. URL do seu n8n: `https://sua-workspace.app.n8n.cloud`

### 3.3. Configurar Credenciais

Antes de importar o workflow, vamos configurar todas as credenciais.

#### 3.3.1. Supabase

1. Menu lateral → **"Credentials"** → **"Add Credential"**
2. Busque: `Supabase`
3. Preencha:
   - **Credential Name:** `Supabase - Satisfaction DB`
   - **Host:** `https://xxxxxxx.supabase.co` (sua Project URL)
   - **Service Role Secret:** `eyJhbGc...` (service_role key do Supabase)
4. Clique em **"Save"**

#### 3.3.2. Anthropic (Claude API)

1. Primeiro, obtenha sua API key:
   - Acesse https://console.anthropic.com
   - Crie conta (trial $5 grátis disponível)
   - Vá em **"API Keys"**
   - Clique em **"Create Key"**
   - Copie a key (começa com `sk-ant-...`)

2. No n8n:
   - **"Credentials"** → **"Add Credential"**
   - Busque: `HTTP Request` → **"Header Auth"**
   - **Credential Name:** `Anthropic API Key`
   - **Name:** `x-api-key`
   - **Value:** `sk-ant-api03-xxxxx` (sua chave)
   - Clique em **"Save"**

#### 3.3.3. Evolution API (WhatsApp)

1. **"Add Credential"** → **"Header Auth"**
2. Preencha:
   - **Credential Name:** `Evolution API Key`
   - **Name:** `apikey`
   - **Value:** `sua-chave-secreta-aqui-123456` (a mesma do Render)
3. Clique em **"Save"**

#### 3.3.4. HubSpot API (Opcional - podemos mockar)

**Opção A: Sandbox Real (Recomendado)**

1. Acesse https://developers.hubspot.com
2. Crie uma conta de desenvolvedor (gratuita)
3. Crie um "Test Account" (sandbox com dados fake)
4. Vá em **"Apps"** → **"Create app"**
5. Configure scopes necessários:
   - `crm.objects.contacts.read`
   - `crm.objects.deals.read`
   - `crm.objects.companies.read`
   - `sales-email-read`
   - `tickets`
6. Obtenha o **Access Token**
7. No n8n:
   - **"Add Credential"** → **"Header Auth"**
   - **Name:** `Authorization`
   - **Value:** `Bearer YOUR_HUBSPOT_TOKEN`

**Opção B: Mockar Dados (Mais Simples para Demo)**

Vamos configurar um node no n8n que retorna dados fixos (faremos isso no workflow).

### 3.4. Importar o Workflow

1. No n8n, clique no **menu "≡"** (canto superior esquerdo)
2. Clique em **"Import from File"**
3. Selecione: `/home/albino/projects/case-agent-dev/workflows/satisfaction-survey-workflow.json`
4. O workflow será importado com todos os 35 nodes

### 3.5. Ajustar Credenciais no Workflow

Após importar, você precisa conectar as credenciais criadas aos nodes:

1. **Nodes do Supabase:**
   - `Supabase - Insert Survey`
   - `Supabase - Get Survey`
   - `Supabase - Update Ongoing`
   - `Supabase - Update Completed`

   Para cada um: clique → aba "Credentials" → selecione `Supabase - Satisfaction DB`

2. **Nodes de Claude API (HTTP Request):**
   - `AGENTE 2: Context Analyzer`
   - `AGENTE 3: Message Generator`
   - `AGENTE 4: Conversation Handler`

   Para cada um: clique → aba "Credentials" → selecione `Anthropic API Key`

3. **Nodes Evolution API:**
   - `Send WhatsApp Message`
   - `Send WhatsApp Response`

   Para cada um: clique → aba "Header Auth" → selecione `Evolution API Key`

### 3.6. Configurar Environment Variables (Prompts)

Como n8n Cloud não tem acesso ao filesystem local, vamos usar uma das abordagens:

**Opção A: Variáveis de Ambiente (Recomendado)**

1. Clique no menu "≡" → **"Settings"** → **"Environment Variables"**
2. Adicione:

```bash
ANTHROPIC_API_KEY=sk-ant-api03-xxxxx
EVOLUTION_API_URL=https://evolution-api-demo.onrender.com
EVOLUTION_INSTANCE_NAME=satisfaction-survey
```

**Opção B: Substituir nodes "Read Prompt" por Code Nodes**

Vou criar versões dos prompts inline para facilitar. Vamos fazer isso depois de testar a estrutura básica.

### 3.7. Ativar o Workflow

1. No canto superior direito, toggle **"Inactive"** → **"Active"**
2. ✅ Workflow está ativo!

### 3.8. Configurar Webhook URL

1. Abra o node **"Webhook - WhatsApp Response"**
2. Copie a **Production URL** (será algo como):
   ```
   https://sua-workspace.app.n8n.cloud/webhook/webhook-whatsapp-response
   ```
3. Guarde essa URL (vamos configurar no Evolution API)

---

## 🚀 Parte 4: Configurar Webhook Evolution → n8n

Agora vamos conectar Evolution API para enviar respostas do WhatsApp para o n8n.

```bash
curl -X POST https://evolution-api-demo.onrender.com/webhook/set/satisfaction-survey \
  -H "Content-Type: application/json" \
  -H "apikey: sua-chave-secreta-aqui-123456" \
  -d '{
    "url": "https://sua-workspace.app.n8n.cloud/webhook/webhook-whatsapp-response",
    "webhook_by_events": true,
    "webhook_base64": false,
    "events": [
      "messages.upsert"
    ]
  }'
```

✅ Resposta esperada:
```json
{
  "webhook": {
    "url": "https://sua-workspace.app.n8n.cloud/webhook/webhook-whatsapp-response",
    "enabled": true
  }
}
```

---

## 🚀 Parte 5: Preparar Prompts dos Agentes

Como n8n Cloud não tem acesso aos arquivos `.txt` locais, vamos embutir os prompts diretamente no workflow.

### 5.1. Atualizar Node "Read Prompt - Agent 2"

1. Abra o node **"Read Prompt - Agent 2"**
2. Mude o tipo de `File` para `Code`
3. Delete o código atual
4. Cole:

```javascript
return [{
  json: {
    system_prompt: `Você é um Analista de Relacionamento de uma empresa B2B SaaS, especializado em analisar dados de clientes do HubSpot para preparar pesquisas de satisfação personalizadas.

Seu trabalho é receber dados brutos sobre um cliente (histórico de emails, deals, tickets e notas) e gerar uma análise estruturada focada em:
1. Resumo do relacionamento do cliente
2. Sentimento geral (positivo, neutro, negativo)
3. Eventos-chave que devem ser mencionados
4. Tom sugerido para abordagem
5. Red flags ou pontos de atenção
6. Pontos de personalização

IMPORTANTE: Seu output DEVE ser um JSON válido, sem markdown, sem explicações extras, APENAS o JSON puro.

Formato de Output (JSON):
{
  "summary": "Resumo de 50-200 palavras sobre o cliente e seu relacionamento",
  "sentiment": "positive|neutral|negative|very_positive|very_negative",
  "key_events": ["evento1", "evento2", "evento3"],
  "suggested_tone": "celebratory|appreciative|curious|empathetic|proactive",
  "red_flags": ["problema1", "problema2"] ou [],
  "personalization_points": ["ponto1", "ponto2", "ponto3"]
}

Regras:
- summary: 50-200 palavras, objetivo, focado em satisfação
- sentiment: apenas um dos 5 valores permitidos
- key_events: 2-5 eventos relevantes (upgrade, problema resolvido, feature usada, etc)
- suggested_tone: baseado no sentimento e contexto
- red_flags: APENAS se houver problemas reais (pode ser array vazio)
- personalization_points: 2-4 pontos específicos para mencionar na mensagem

Exemplos de key_events:
- "Fez upgrade para Plano Pro há 2 semanas"
- "Abriu ticket sobre integração que foi resolvido rapidamente"
- "Participou de webinar sobre automações"
- "Não utiliza feature X que poderia ajudá-lo"

Tom sugerido por contexto:
- celebratory: cliente teve sucesso recente, milestone alcançado
- appreciative: cliente leal, sem problemas, relacionamento estável
- curious: pouco engajamento, queremos entender melhor
- empathetic: teve problemas recentes, queremos mostrar cuidado
- proactive: oportunidade de oferecer valor adicional

CRÍTICO: Retorne APENAS o JSON, sem \`\`\`json, sem texto antes ou depois.`
  }
}];
```

5. Clique em **"Save"**

### 5.2. Atualizar Node "Read Prompt - Agent 3"

Repita o processo para o Agent 3:

```javascript
return [{
  json: {
    system_prompt: `Você é Sofia, Gerente de Relacionamento de uma empresa B2B SaaS, responsável por criar mensagens personalizadas de abertura para pesquisas de satisfação via WhatsApp.

Você receberá:
1. Nome do cliente
2. Análise de contexto (summary, sentiment, personalization_points, suggested_tone)

Seu trabalho é criar uma mensagem curta, natural e personalizada que:
- Cumprimente o cliente pelo nome
- Mencione algo específico do relacionamento dele (use personalization_points)
- Peça feedback de forma leve e conversacional
- Inclua a escala de 1 a 5
- Seja autêntica e humanizada (não pareça bot)

FORMATO DE OUTPUT:
Retorne APENAS o texto da mensagem, sem JSON, sem aspas, sem formatação markdown.

REGRAS:
- Máximo 400 caracteres (WhatsApp friendly)
- Tom conversacional, como se fosse uma pessoa real
- SEMPRE incluir "de 1 a 5" ou "1-5" na pergunta
- Usar emoji APENAS se o tom for celebratory ou muito positivo (máximo 1 emoji)
- Não usar jargão técnico excessivo
- Fazer UMA pergunta clara sobre satisfação

TONS E ABORDAGENS:

1. celebratory (cliente teve sucesso):
Exemplo: "Oi [Nome]! Vi que você [conquista recente]! 🎉 Como está sendo a experiência? De 1 a 5, como você avalia nosso suporte nessa jornada?"

2. appreciative (cliente leal):
Exemplo: "Oi [Nome]! Você é cliente há [tempo] e isso é muito importante para nós. De 1 a 5, como você avalia nossa plataforma?"

3. curious (pouco engajamento):
Exemplo: "Oi [Nome]! Notei que você tem usado [feature X]. Como tem sido? De 1 a 5, o quanto você está satisfeito?"

4. empathetic (teve problemas):
Exemplo: "Oi [Nome], vi que você teve [problema] recentemente. Espero que esteja tudo resolvido agora. De 1 a 5, como avalia nosso atendimento?"

5. proactive (oportunidade):
Exemplo: "Oi [Nome]! Como vai? Vi que você usa bastante [feature Y]. De 1 a 5, como está sendo a experiência geral com nossa plataforma?"

IMPORTANTE:
- Adapte a mensagem ao contexto específico do cliente
- Seja genuíno, não genérico
- Máximo 300 caracteres é ideal (400 é o limite absoluto)
- Retorne APENAS o texto da mensagem, nada mais`
  }
}];
```

### 5.3. Atualizar Node "Read Prompt - Agent 4"

E finalmente o Agent 4:

```javascript
return [{
  json: {
    system_prompt: `Você é Sofia, Gerente de Relacionamento, conduzindo uma conversa de pesquisa de satisfação via WhatsApp.

Seu objetivo é:
1. Manter conversa natural e empática
2. Extrair nota de satisfação (1-5) e feedback
3. Entender o "porquê" da nota
4. Decidir quando encerrar ou escalar para humano

IMPORTANTE: Seu output DEVE ser um JSON válido, sem markdown, APENAS o JSON puro.

INPUT que você recebe:
{
  "conversation_history": [
    {"role": "assistant", "content": "mensagem inicial"},
    {"role": "user", "content": "resposta do cliente"},
    ...
  ],
  "turn_count": número_do_turno,
  "context_summary": "resumo do cliente"
}

OUTPUT esperado (JSON):
{
  "response": "sua resposta ao cliente (máx 500 caracteres)",
  "status": "continue|completed|escalate",
  "extracted_data": {
    "satisfaction_score": 1-5 ou null,
    "main_feedback": "resumo do feedback" ou null,
    "sentiment": "positive|neutral|negative" ou null
  }
}

REGRAS DE DECISÃO:

1. status = "continue" quando:
   - Cliente ainda não deu nota clara
   - Cliente deu nota mas você quer entender melhor o porquê
   - Menos de 5 turnos de conversa
   - Conversa fluindo naturalmente

2. status = "completed" quando:
   - Nota foi extraída E feedback foi dado
   - Cliente claramente encerrou a conversa ("obrigado", "só isso", etc)
   - 5 turnos completados

3. status = "escalate" quando:
   - Cliente pede para falar com humano
   - Cliente relata problema urgente/grave
   - Cliente demonstra frustração alta

DETECÇÃO DE NOTA:
Extraia satisfaction_score de formatos variados:
- Numérico direto: "5", "4/5", "nota 3"
- Textual: "muito satisfeito" = 5, "satisfeito" = 4, "neutro" = 3, "insatisfeito" = 2, "muito insatisfeito" = 1
- Emojis: 😍🤩 = 5, 😊🙂 = 4, 😐 = 3, 😕 = 2, 😠😡 = 1
- Qualitativo: "excelente" = 5, "bom" = 4, "ok/médio" = 3, "ruim" = 2, "péssimo" = 1

TOM DA CONVERSA:
- Natural e empático
- Agradeça feedback positivo com entusiasmo genuíno
- Demonstre preocupação real com feedback negativo
- Faça perguntas abertas ("O que mais poderíamos melhorar?")
- Não seja repetitivo
- Máximo 500 caracteres por resposta (WhatsApp friendly)

EXEMPLOS DE RESPOSTAS:

Após nota 5:
"Que ótimo saber que você está satisfeito! 😊 O que você mais gosta na plataforma?"

Após nota 3:
"Entendo. O que poderíamos fazer para melhorar sua experiência?"

Após nota 1-2:
"Sinto muito pela experiência ruim. Pode me contar o que aconteceu? Queremos resolver isso."

Cliente pede humano:
"Claro! Vou conectar você com um gerente agora mesmo."

CRÍTICO:
- Retorne APENAS o JSON, sem \`\`\`json, sem texto extra
- response deve ser string simples, sem quebras de linha complexas
- satisfaction_score deve ser number (1-5) ou null, não string`
  }
}];
```

### 5.4. Salvar o Workflow

1. Após atualizar os 3 nodes de prompts
2. Clique em **"Save"** (canto superior direito)
3. ✅ Workflow atualizado com prompts inline!

---

## 🧪 Parte 6: Testar o Sistema End-to-End

Agora vamos testar o fluxo completo!

### 6.1. Criar Dados de Teste no HubSpot (ou Mockar)

**Opção A: Mockar HubSpot (Mais Simples para Demo)**

1. Abra o node **"Set Contact ID"**
2. Adicione um novo node **"Code"** logo após ele
3. Nomeie: `Mock HubSpot Data`
4. Cole o código:

```javascript
const contactId = $input.first().json.contact_id;

return [{
  json: {
    contact: {
      id: contactId,
      properties: {
        firstname: "João",
        lastname: "Silva",
        email: "joao.silva@example.com",
        phone: "5511999887766",
        lifecyclestage: "customer",
        hs_lead_status: "CUSTOMER"
      }
    },
    emails: {
      results: [
        {
          properties: {
            hs_email_subject: "Boas-vindas ao nosso sistema!",
            hs_email_text: "Olá João, bem-vindo!",
            hs_timestamp: "2025-12-01T10:00:00Z"
          }
        },
        {
          properties: {
            hs_email_subject: "Seu upgrade foi concluído",
            hs_email_text: "Parabéns pelo upgrade para o Plano Pro!",
            hs_timestamp: "2026-01-15T14:30:00Z"
          }
        }
      ]
    },
    deals: {
      results: [
        {
          properties: {
            dealname: "Upgrade para Plano Pro",
            dealstage: "closedwon",
            amount: "499",
            closedate: "2026-01-15"
          }
        }
      ]
    },
    tickets: {
      results: [
        {
          properties: {
            subject: "Dúvida sobre integração API",
            hs_ticket_priority: "HIGH",
            hs_pipeline_stage: "4",
            createdate: "2026-01-20T09:00:00Z"
          }
        }
      ]
    },
    notes: {
      results: [
        {
          properties: {
            hs_note_body: "Cliente muito engajado, usa features avançadas regularmente.",
            hs_timestamp: "2026-01-22T15:00:00Z"
          }
        }
      ]
    }
  }
}];
```

5. Conecte `Set Contact ID` → `Mock HubSpot Data` → `AGENTE 1: Data Fetcher`
6. Desconecte os 5 nodes de HubSpot API real (ou deixe em paralelo para usar depois)

**Opção B: Usar HubSpot Sandbox Real**

Se você configurou HubSpot sandbox com credenciais, pode usar os nodes reais. Apenas certifique-se de ter contatos de teste criados.

### 6.2. Teste Manual - Envio Inicial

1. No workflow, clique no node **"Set Contact ID"**
2. Clique em **"Execute Node"**
3. No painel que abrir, cole:
```json
{
  "contact_id": "demo-12345"
}
```
4. Clique em **"Execute Node"**
5. 🎯 Acompanhe a execução percorrer todos os nodes:
   - Mock/Get data from HubSpot ✅
   - AGENTE 1: Data Fetcher ✅
   - AGENTE 2: Context Analyzer ✅ (chamada Claude API)
   - Validate Analysis ✅
   - AGENTE 3: Message Generator ✅ (chamada Claude API)
   - Prepare Message ✅
   - Send WhatsApp + Supabase + Chatwoot (paralelo) ✅

6. **Verificar WhatsApp:**
   - Você deve receber uma mensagem no número conectado na Evolution API
   - Exemplo: "Oi João! Vi que você fez upgrade para o Plano Pro recentemente. Como está sendo a experiência? De 1 a 5, como você avalia nosso serviço?"

7. **Verificar Supabase:**
   - Vá no Supabase → Table Editor → satisfaction_surveys
   - Deve ter 1 registro novo com status = "pending"

### 6.3. Teste de Conversa - Responder no WhatsApp

1. Pegue seu celular
2. Responda à mensagem do bot, por exemplo:
   ```
   5! Estou muito satisfeito, a plataforma é excelente!
   ```

3. O webhook vai disparar automaticamente e:
   - Webhook recebe sua mensagem
   - Busca a survey ativa no Supabase
   - Monta histórico de conversa
   - AGENTE 4 processa e responde
   - Você recebe nova mensagem

4. Continue a conversa:
   - Bot pode perguntar: "Que ótimo saber! O que você mais gosta?"
   - Você responde: "A facilidade de usar as automações"
   - Bot encerra: "Obrigado pelo feedback, João! Vamos continuar melhorando 😊"

5. **Verificar finalização:**
   - Supabase → satisfaction_surveys → status = "completed"
   - satisfaction_score = 5
   - conversation_transcript com todo o histórico em JSON

### 6.4. Verificar Logs Completos

1. No n8n, clique em **"Executions"** (menu lateral)
2. Você verá todas as execuções:
   - Primeira execução (trigger manual - envio inicial)
   - Segunda execução (webhook - primeira resposta)
   - Terceira execução (webhook - segunda resposta)
   - Etc.

3. Clique em qualquer execução para ver:
   - Dados de entrada/saída de cada node
   - Tempo de execução
   - Erros (se houver)

4. **Exportar logs para o PDF:**
   - Tire screenshots das execuções
   - Copie JSONs relevantes
   - Guarde para incluir no documento final

---

## 📸 Parte 7: Coletar Evidências para o PDF

### 7.1. Screenshots Necessários

Tire prints de tela de:

1. **n8n Workflow Overview**
   - Vista geral do workflow com todos os 35 nodes
   - Zoom em cada parte (Envio Inicial e Gerenciamento de Respostas)

2. **n8n Execution Logs**
   - Execução bem-sucedida do envio inicial
   - Execução do loop de conversa
   - Detalhes de cada agente (input/output)

3. **Supabase Database**
   - Tabela satisfaction_surveys com registros de teste
   - SQL queries mostrando dados

4. **WhatsApp Conversa Real**
   - Print da conversa completa cliente ↔ bot
   - Mostrar naturalidade das respostas

5. **Evolution API Dashboard**
   - Status da instância conectada
   - Webhooks configurados

6. **Anthropic Console**
   - Dashboard mostrando uso de API (tokens, custos)

### 7.2. Exportar Dados para Análise

**Transcrição de Conversa:**

No Supabase SQL Editor:

```sql
SELECT
  contact_name,
  satisfaction_score,
  customer_feedback,
  conversation_transcript,
  created_at,
  completed_at
FROM satisfaction_surveys
WHERE status = 'completed'
ORDER BY completed_at DESC
LIMIT 5;
```

Copie o resultado e formate para incluir no PDF como exemplo real.

**Métricas de Performance:**

```sql
-- Tempo médio de conclusão
SELECT
  AVG(EXTRACT(EPOCH FROM (completed_at - created_at))) / 60 as avg_minutes,
  COUNT(*) as total_surveys,
  AVG(satisfaction_score) as avg_score
FROM satisfaction_surveys
WHERE status = 'completed';
```

### 7.3. Exemplo de Transcrição Formatada

Crie um arquivo `/docs/exemplo-conversa-real.md`:

```markdown
# Exemplo de Conversa Real - Pesquisa de Satisfação

**Cliente:** João Silva
**Data:** 2026-01-28 14:30
**Duração:** 3 minutos
**Nota Final:** 5/5
**Status:** Completed

---

## Transcrição

**[14:30] Sofia (Bot):**
> Oi João! Vi que você fez upgrade para o Plano Pro recentemente. Como está sendo a experiência com as funcionalidades avançadas? De 1 a 5, como você avalia nosso serviço?

**[14:31] João Silva:**
> 5! Estou muito satisfeito, a plataforma é excelente!

**[14:31] Sofia (Bot):**
> Que ótimo saber que você está satisfeito! 😊 O que você mais gosta na plataforma?

**[14:32] João Silva:**
> A facilidade de usar as automações e o suporte rápido quando precisei

**[14:32] Sofia (Bot):**
> Perfeito! Ficamos muito felizes em saber que conseguimos te ajudar. Obrigado pelo feedback, João! Vamos continuar melhorando para você 🙂

---

## Análise Automática

**Sentiment Detected:** very_positive
**Satisfaction Score:** 5
**Key Feedback Points:**
- Facilidade de uso das automações
- Qualidade do suporte (rápido)
- Upgrade para Plano Pro foi positivo

**Action Taken:** Survey completed, data saved to Supabase
**Next Steps:** None (customer highly satisfied, no escalation needed)
```

---

## 🔗 Parte 8: Links Públicos para o PDF

### 8.1. Tornar Workflow Público (Template)

No n8n Cloud:

1. Abra o workflow
2. Clique no menu "..." (três pontos) → **"Share"**
3. Escolha **"Create Template"**
4. Preencha:
   - **Name:** `Sistema de Satisfação com IA - FMA Case`
   - **Description:** `Workflow completo para pesquisa de satisfação automatizada com 4 agentes de IA`
5. Marque **"Public"**
6. Copie o link público gerado (ex: `https://n8n.io/workflows/12345`)

### 8.2. Compartilhar Credenciais de Demo Supabase

**IMPORTANTE:** Nunca compartilhe service_role key em público!

Para demo, você pode:

**Opção A: Criar View Read-Only**

```sql
-- Criar view pública (somente leitura)
CREATE VIEW public_satisfaction_surveys AS
SELECT
  id,
  contact_name,
  satisfaction_score,
  customer_feedback,
  status,
  created_at,
  completed_at
FROM satisfaction_surveys;

-- Permitir acesso público à view
GRANT SELECT ON public_satisfaction_surveys TO anon;
```

Depois compartilhe:
- Project URL: `https://xxxxxxx.supabase.co`
- Anon key: `eyJhbGc...` (essa pode ser pública)
- Tabela: `public_satisfaction_surveys` (view read-only)

**Opção B: Screenshots Apenas**

Mais seguro: não compartilhar acesso, apenas screenshots no PDF.

### 8.3. Link da Evolution API

A Evolution API na Render é pública, mas protegida por API key.

No PDF, você pode incluir:
- URL base: `https://evolution-api-demo.onrender.com`
- Documentação: Como configurar (sem expor a API key)
- Screenshots do QR Code e status

### 8.4. Resumo de Links para o PDF

Crie uma seção no PDF:

```markdown
## 🔗 Links e Recursos

### Workflow n8n (Público)
- **Template n8n:** https://n8n.io/workflows/xxxxx
- **Instruções:** Clique em "Use Template" para importar

### Banco de Dados (Read-Only Demo)
- **Supabase Project:** https://xxxxxxx.supabase.co
- **Tabela Pública:** `public_satisfaction_surveys`
- **Query de Exemplo:**
  ```sql
  SELECT * FROM public_satisfaction_surveys ORDER BY created_at DESC LIMIT 10;
  ```

### Repositório GitHub (Opcional)
Se você criar um repositório com a documentação:
- **GitHub:** https://github.com/seu-usuario/case-agent-satisfaction

### APIs Utilizadas
- **n8n Cloud:** https://n8n.io
- **Supabase:** https://supabase.com
- **Anthropic Claude:** https://console.anthropic.com
- **Evolution API:** https://github.com/EvolutionAPI/evolution-api
```

---

## 📊 Parte 9: Métricas e Validação

### 9.1. Testes de Validação

Execute os seguintes testes e documente resultados:

| Teste | Objetivo | Resultado Esperado |
|-------|----------|-------------------|
| 1. Envio inicial | Verificar integração completa | Mensagem enviada, DB atualizado |
| 2. Conversa 1 turno | Testar detecção de nota | Score extraído corretamente |
| 3. Conversa 3 turnos | Testar loop conversacional | Conversa natural, encerramento OK |
| 4. Escalação | Testar pedido de humano | Status = "escalate" |
| 5. Timeout | Testar limite de turnos | Max 5 turnos respeitado |
| 6. Nota ambígua | Testar parsing variado | "muito bom" = 4 ou 5 |

### 9.2. Métricas de Performance

Documente:

**Tempo de Execução:**
- Envio inicial (todos os agentes): X segundos
- Resposta conversacional (Agente 4): Y segundos
- Total por pesquisa completa: Z segundos

**Custos Reais:**
- Tokens usados por Agente 2: ~X tokens
- Tokens usados por Agente 3: ~Y tokens
- Tokens usados por Agente 4 (por turno): ~Z tokens
- Custo total por pesquisa: $XX

**Taxa de Sucesso:**
- Pesquisas completadas: X%
- Notas extraídas corretamente: Y%
- Escalações necessárias: Z%

### 9.3. Análise de Qualidade

**Naturalidade das Respostas:**
- Peça para 2-3 pessoas testarem
- Elas perceberam que era bot?
- Feedback sobre tom e empatia

**Precisão da Análise:**
- Agente 2 detectou sentiment corretamente?
- Personalization points foram relevantes?
- Tone sugerido foi apropriado?

---

## 🚨 Troubleshooting

### Problema: Evolution API (Render) hiberna (free tier)

**Sintoma:** Webhook não recebe mensagens após 15min de inatividade

**Solução:**
1. Render free tier hiberna após inatividade
2. Primeiro acesso após hibernar demora ~30s (cold start)
3. Para demo: fazer uma chamada de "wake up" antes de testar

```bash
# Wake up da Evolution API
curl https://evolution-api-demo.onrender.com/instance/fetchInstances \
  -H "apikey: sua-chave"
```

**Alternativa (se quiser evitar hibernação):**
- Upgrade para plano pago da Render ($7/mês)
- Ou configurar cron job que faz ping a cada 10 minutos

### Problema: Claude API retorna erro 429 (rate limit)

**Sintoma:** Muitos testes seguidos, API bloqueia

**Solução:**
1. Trial tem limite de 50 requests/minuto
2. Espere 1 minuto entre testes
3. Ou adicione retry logic no n8n (já configurado no workflow)

### Problema: Webhook n8n não recebe dados

**Sintoma:** WhatsApp enviado, mas workflow não dispara

**Checklist:**
1. ✅ Webhook está ativo no n8n?
2. ✅ Workflow está "Active"?
3. ✅ Evolution API tem a URL correta do webhook?
4. ✅ Evolution API está online? (verifique Render dashboard)
5. ✅ WhatsApp está conectado? (check instance status)

**Debug:**
```bash
# Verificar webhook configurado
curl https://evolution-api-demo.onrender.com/webhook/find/satisfaction-survey \
  -H "apikey: sua-chave"
```

### Problema: Supabase rejeita INSERT (RLS)

**Sintoma:** Erro 403 ou policy violation

**Solução:**
1. Verificar se RLS está desabilitado:
```sql
SELECT tablename, rowsecurity
FROM pg_tables
WHERE tablename = 'satisfaction_surveys';
```

Se `rowsecurity = true`:
```sql
ALTER TABLE satisfaction_surveys DISABLE ROW LEVEL SECURITY;
```

### Problema: Prompts retornam JSON inválido

**Sintoma:** Agente retorna texto com markdown ou formato errado

**Solução:**
1. Claude às vezes retorna ```json ... ``` apesar da instrução
2. Adicione parsing no node "Validate Analysis":

```javascript
let analysis;
try {
  // Tentar parse direto
  analysis = JSON.parse($json.content[0].text);
} catch (e) {
  // Remover markdown se houver
  const text = $json.content[0].text;
  const jsonMatch = text.match(/```json\n([\s\S]*?)\n```/) ||
                    text.match(/```\n([\s\S]*?)\n```/);

  if (jsonMatch) {
    analysis = JSON.parse(jsonMatch[1]);
  } else {
    throw new Error('Failed to parse JSON from Claude response');
  }
}

return { json: analysis };
```

---

## ✅ Checklist Final - Antes de Documentar no PDF

Antes de compilar o PDF final, certifique-se:

### Infraestrutura
- [ ] n8n Cloud ativo e workflow funcionando
- [ ] Supabase com tabela criada e dados de teste
- [ ] Evolution API na Render online e WhatsApp conectado
- [ ] Anthropic API key válida com créditos
- [ ] Webhook Evolution → n8n configurado

### Testes Realizados
- [ ] Envio inicial testado com sucesso (mensagem recebida)
- [ ] Conversa completa testada (mínimo 3 turnos)
- [ ] Nota de satisfação extraída corretamente
- [ ] Dados salvos no Supabase corretamente
- [ ] Logs de execução capturados

### Evidências Coletadas
- [ ] Screenshots do workflow n8n (overview + detalhes)
- [ ] Screenshots de execuções bem-sucedidas
- [ ] Print da conversa real no WhatsApp
- [ ] Print do Supabase com dados salvos
- [ ] Transcrição formatada da conversa
- [ ] Métricas de performance documentadas

### Links Públicos
- [ ] Template n8n publicado (link público)
- [ ] Supabase view read-only criada (opcional)
- [ ] Documentação de acesso preparada
- [ ] Links testados e funcionando

### Documentação
- [ ] Guia de deploy completo (este arquivo)
- [ ] README atualizado com links
- [ ] Exemplo de conversa documentado
- [ ] Troubleshooting documentado

---

## 🎁 Extras para Impressionar Avaliadores

### 1. Vídeo Demo (Opcional mas Impactante)

Grave um vídeo de 3-5 minutos mostrando:
1. Acesso ao n8n Cloud (workflow overview)
2. Executar manualmente um envio
3. Receber mensagem no WhatsApp
4. Responder e ver bot conversando
5. Mostrar dados salvos no Supabase
6. Mostrar logs de execução

**Ferramentas:**
- Loom (gratuito, 5min suficiente)
- OBS Studio (gratuito, sem limites)

### 2. Dashboard de Métricas

Criar um dashboard no Supabase ou Metabase mostrando:
- Total de pesquisas realizadas
- Distribuição de notas (1-5)
- Taxa de conclusão
- Tempo médio de conversa
- Sentiment analysis agregado

### 3. API Pública de Consulta

Criar um endpoint público (via n8n webhook) que retorna estatísticas:

```
GET https://sua-workspace.app.n8n.cloud/webhook/stats

Response:
{
  "total_surveys": 42,
  "average_score": 4.2,
  "completion_rate": "87%",
  "average_duration_minutes": 3.5
}
```

---

## 📝 Próximos Passos

Após completar este guia:

1. ✅ Testar tudo extensivamente
2. ✅ Coletar todas as evidências (screenshots, logs, transcrições)
3. ✅ Documentar métricas reais de performance
4. ✅ Criar exemplo de conversa real formatado
5. ✅ Gerar links públicos para compartilhar
6. 🎯 **Compilar documento final PDF** (Tarefa #9)

---

## 🔗 Recursos e Links Úteis

- **n8n Cloud Docs:** https://docs.n8n.io/hosting/cloud/
- **Supabase Docs:** https://supabase.com/docs
- **Evolution API Docs:** https://doc.evolution-api.com/
- **Anthropic API Docs:** https://docs.anthropic.com/claude/reference/
- **Render Docs:** https://render.com/docs

---

## 💬 Suporte

Para problemas durante o deploy:

1. Revisar seção de Troubleshooting deste guia
2. Consultar documentação oficial das plataformas
3. Verificar logs em cada serviço:
   - n8n: Executions → detalhes
   - Supabase: Logs (menu lateral)
   - Render: Logs da aplicação
   - Evolution: Logs via API

---

**Última atualização:** 2026-01-28
**Versão:** 1.0
**Status:** Pronto para deploy
**Tempo estimado de setup:** 2-3 horas
**Custo mensal:** $1-5 (apenas Anthropic API)

**Boa sorte com o deploy! 🚀**
