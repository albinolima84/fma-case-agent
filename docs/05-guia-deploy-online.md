# Guia de Deploy Online - Demo Completo

**Projeto:** Sistema de Satisfação com IA
**Objetivo:** Deploy completo em cloud gratuito/barato para demo funcional
**Público:** Avaliadores do desafio FMA + desenvolvedores

---

## 🎯 Visão Geral

Este guia configura um **ambiente 100% funcional na nuvem** onde avaliadores podem testar o sistema completo sem instalar nada localmente.

### Stack do Demo Online:
- ☁️ **n8n Cloud** (trial 14 dias, depois Starter R$150/mês)
- ☁️ **Supabase Cloud** (free tier - 500MB database)
- ☁️ **Meta WhatsApp Business API** (oficial — sem deploy próprio)
- ☁️ **Tess AI** (gpt-4o-mini — agentes pré-configurados)
- ☁️ **Chatwoot Cloud** (Plano Hacker — gratuito para piloto)
- ☁️ **HubSpot** (conta existente ou Sandbox de desenvolvedor)

### Custos Estimados:
| Serviço | Plano | Custo/mês (Piloto 14d) | Custo/mês (Produção) |
|---------|-------|------------------------|----------------------|
| n8n Cloud | Trial → Starter | $0 (14 dias) | R$150 |
| Supabase | Free tier | $0 | $0 |
| Meta WhatsApp API | Plataforma | variável | variável |
| Tess AI | Créditos | ~$5-10 | ~R$30 |
| Chatwoot Cloud | Hacker (free) | $0 | R$105 (Pro) |
| HubSpot | Existente/Sandbox | $0 | $0 |
| **TOTAL (Piloto)** | | **~$5-10/mês** | - |
| **TOTAL (Produção)** | | - | **~R$285/mês** |

**Importante:** O n8n Cloud oferece 14 dias de trial gratuito. Após isso, o plano mínimo é o Starter (R$150/mês). Para validação do MVP durante o trial, o custo é zero.

---

## 📋 Pré-requisitos

Antes de começar, tenha em mãos:
- [ ] Email para criar contas
- [ ] Conta Meta Business com App criado no [Meta for Developers](https://developers.facebook.com)
- [ ] Número de telefone WhatsApp Business conectado ao App Meta
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

### 1.3. Criar Tabela `surveys`

1. No menu lateral, clique em **"SQL Editor"**
2. Clique em **"New query"**
3. Cole o seguinte SQL:

```sql
-- Criar tabela de pesquisas de satisfação
CREATE TABLE surveys (
  id SERIAL PRIMARY KEY,
  customer_phone VARCHAR(20) NOT NULL,
  customer_name VARCHAR(255),
  context_summary TEXT,
  conversation_transcript JSONB DEFAULT '[]',
  satisfaction_score INTEGER CHECK (satisfaction_score >= 1 AND satisfaction_score <= 5),
  main_feedback TEXT,
  sentiment VARCHAR(20),
  status VARCHAR(20) DEFAULT 'active',
  chatwoot_conversation_id INTEGER,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Criar índices para performance
CREATE INDEX idx_surveys_phone ON surveys(customer_phone);
CREATE INDEX idx_surveys_status ON surveys(status);
CREATE INDEX idx_surveys_created_at ON surveys(created_at DESC);

-- Criar função para atualizar updated_at automaticamente
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Criar trigger para updated_at
CREATE TRIGGER update_surveys_updated_at
    BEFORE UPDATE ON surveys
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

4. Clique em **"Run"** (ou `Ctrl + Enter`)
5. ✅ Você deve ver: `Success. No rows returned`

### 1.4. Desabilitar RLS (Row Level Security) para Demo

Para facilitar o acesso do n8n sem configurar políticas complexas:

```sql
-- Desabilitar RLS temporariamente (apenas para demo)
ALTER TABLE surveys DISABLE ROW LEVEL SECURITY;
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
INSERT INTO surveys (
  customer_phone,
  customer_name,
  status
) VALUES (
  '5511999999999',
  'João da Silva',
  'active'
);

-- Verificar se foi criado
SELECT * FROM surveys;
```

✅ Você deve ver o registro criado. Pode deletar depois:
```sql
DELETE FROM surveys WHERE customer_name = 'João da Silva';
```

---

## 🚀 Parte 2: Setup do Chatwoot Cloud

### 2.1. Criar Conta

1. Acesse https://app.chatwoot.com
2. Clique em **"Sign up"**
3. Preencha nome, email e senha
4. Selecione plano **Hacker** (gratuito)

### 2.2. Criar Inbox (Canal API)

1. No dashboard, vá em **Settings → Inboxes → Add Inbox**
2. Selecione **"API"** como tipo de canal
3. Preencha:
   - **Name:** `satisfaction-survey`
4. Clique em **"Create Inbox"**
5. ✅ Anote o **Inbox ID** mostrado nas configurações

### 2.3. Obter Credenciais

1. Vá em **Settings → General**
2. Copie e salve:
   - **Account ID** — visível na URL do dashboard (ex: `/app/accounts/XXXXX`)
   - **API Access Token** — gerado em Settings → Account Settings → Access Token
   - **Inbox ID** — da etapa 2.2

> **Limitações do Plano Hacker:** 1 inbox e 2 agentes. Suficiente para o piloto — apenas 1 inbox é utilizado e todas as operações são feitas via API (sem agentes humanos no Chatwoot). Upgrade para Plano Pro necessário na produção (retenção de dados > 30 dias).

---

## 🚀 Parte 3: Setup da Meta WhatsApp Business API

### 3.1. Pré-requisitos Meta

- Conta no [Meta Business Suite](https://business.facebook.com)
- App criado no [Meta for Developers](https://developers.facebook.com)
- Número de telefone WhatsApp Business conectado ao App

### 3.2. Informações Necessárias

No Meta for Developers → seu App → **WhatsApp → Configuration**, copie:

- **Phone Number ID** — identificador do número conectado
- **Permanent Access Token** — token de acesso permanente (use token de longa duração para evitar expiração)
- **API Version** — use `v21.0`

### 3.3. Webhook (configura após criar o n8n)

O webhook será configurado após obter a URL do n8n Cloud (Parte 5). Anote para uso posterior:
- **Verify Token:** um valor aleatório que você escolhe (ex: `satisfaction-survey-2026`)
- **Callback URL:** será a URL do webhook do n8n (preenchemos na Parte 5)
- **Subscription Fields:** `messages` (em WhatsApp)

---

## 🚀 Parte 4: Setup do n8n Cloud

### 4.1. Criar Conta

1. Acesse https://n8n.io
2. Clique em **"Start for free"**
3. Preencha email, nome e senha
4. **Selecione:** `Cloud` (não "Self-hosted")
5. Crie uma workspace (ex: `case-agent-dev`)

### 4.2. Inicializar Workspace

1. ✅ n8n vai criar seu ambiente em ~30 segundos
2. Você será redirecionado para o editor
3. URL do seu n8n: `https://sua-workspace.app.n8n.cloud`

### 4.3. Configurar Credenciais

Antes de importar o workflow, configure todas as credenciais.

#### 4.3.1. Supabase (Postgres)

1. Menu lateral → **"Credentials"** → **"Add Credential"**
2. Busque: `Postgres`
3. Preencha:
   - **Credential Name:** `Supabase - Satisfaction DB`
   - **Host:** `db.xxxxxxx.supabase.co`
   - **Database:** `postgres`
   - **User:** `postgres`
   - **Password:** senha criada no Supabase
   - **Port:** `5432`
4. Clique em **"Save"** → teste a conexão

#### 4.3.2. Tess AI

1. **"Add Credential"** → **"Header Auth"**
2. Preencha:
   - **Credential Name:** `Tess AI`
   - **Name:** `Authorization`
   - **Value:** `Bearer {SEU_TOKEN_TESS_AI}`
3. Clique em **"Save"**

#### 4.3.3. Meta WhatsApp API

1. **"Add Credential"** → **"Header Auth"**
2. Preencha:
   - **Credential Name:** `Meta WhatsApp API`
   - **Name:** `Authorization`
   - **Value:** `Bearer {SEU_ACCESS_TOKEN_META}`
3. Clique em **"Save"**

#### 4.3.4. Chatwoot Cloud API

1. **"Add Credential"** → **"Header Auth"**
2. Preencha:
   - **Credential Name:** `Chatwoot Cloud API`
   - **Name:** `api_access_token`
   - **Value:** `{SEU_API_ACCESS_TOKEN_CHATWOOT}`
3. Clique em **"Save"**

#### 4.3.5. HubSpot API

1. **"Add Credential"** → **"Header Auth"**
2. Preencha:
   - **Credential Name:** `HubSpot API`
   - **Name:** `Authorization`
   - **Value:** `Bearer {SEU_TOKEN_HUBSPOT}`
3. Clique em **"Save"**

### 4.4. Importar o Workflow

1. No n8n, clique no **menu "≡"** (canto superior esquerdo)
2. Clique em **"Import from File"**
3. Selecione: `workflows/satisfaction-survey-workflow.json`
4. O workflow será importado com todos os nodes

### 4.5. Ajustar Credenciais no Workflow

Após importar, conecte as credenciais aos nodes:

1. **Nodes Supabase** (`Supabase - Insert Survey`, `Supabase - Get Survey`, `Supabase - Update Ongoing`, `Supabase - Update Completed`):
   - Clique em cada → aba "Credentials" → selecione `Supabase - Satisfaction DB`

2. **Nodes Tess AI** (`AGENTE 2: Context Analyzer`, `AGENTE 3: Message Generator`, `AGENTE 4: Conversation Handler`):
   - São HTTP Request nodes apontando para `https://api.tess.im/agents/{id}/execute`
   - Clique em cada → selecione `Tess AI`

3. **Nodes Meta WhatsApp API** (`Send WhatsApp Message`, `Send WhatsApp Response`):
   - São HTTP Request nodes apontando para `https://graph.facebook.com/v21.0/{phone_number_id}/messages`
   - Clique em cada → selecione `Meta WhatsApp API`

4. **Nodes Chatwoot** (Search Contact, Create Contact, Create Conversation, Send Message, Add Private Note, Mark Resolved):
   - São HTTP Request nodes apontando para `https://app.chatwoot.com/api/v1/accounts/{account_id}/...`
   - Clique em cada → selecione `Chatwoot Cloud API`

5. **Nodes HubSpot** (`Get Contact`, `Get Emails`, `Get Deals`, `Get Tickets`):
   - Clique em cada → selecione `HubSpot API`

### 4.6. Ativar o Workflow

1. No canto superior direito, toggle **"Inactive"** → **"Active"**
2. ✅ Workflow está ativo!

### 4.7. Obter URL do Webhook

1. Abra o node **"Webhook - WhatsApp Meta"** (início do FLUXO 2)
2. Copie a **Production URL** (será algo como):
   ```
   https://sua-workspace.app.n8n.cloud/webhook/whatsapp-meta
   ```
3. Guarde essa URL — usaremos na Parte 5 para configurar o webhook Meta

---

## 🚀 Parte 5: Configurar Webhook Meta → n8n

### 5.1. Configurar no Meta for Developers

1. Acesse seu App no [Meta for Developers](https://developers.facebook.com)
2. Vá em **WhatsApp → Configuration → Webhooks**
3. Preencha:
   - **Callback URL:** `https://sua-workspace.app.n8n.cloud/webhook/whatsapp-meta`
   - **Verify Token:** o valor definido na Parte 3 (ex: `satisfaction-survey-2026`)
4. Clique em **"Verify and Save"**
5. ✅ A Meta vai fazer uma chamada GET para validar o webhook

### 5.2. Subscribir nos Eventos

1. Na mesma página, clique em **"Manage Subscription Fields"**
2. Habilite: **messages** (sob WhatsApp)
3. Clique em **"Save"**

---

## 🚀 Parte 6: Agentes no Tess AI

Os agentes são configurados diretamente na plataforma Tess AI — não no workflow n8n. O workflow apenas chama o endpoint da API de cada agente.

### 6.1. Verificar Agentes Configurados

No [Tess AI](https://tess.im), verifique que os 3 agentes existem e estão ativos:

| Agente | Nome | ID | Modelo | Temperatura |
|--------|------|----|--------|-------------|
| 2 | Context Analyzer | 38717 | gpt-4o-mini | 1 |
| 3 | Message Generator | 38728 | gpt-4o-mini | 1 |
| 4 | Conversation Handler V2.0 | 38733 | gpt-4o-mini | 1 |

### 6.2. Prompts dos Agentes

Os prompts de cada agente estão no diretório `prompts/` do repositório:
- `prompts/agent-2-context-analyzer.txt`
- `prompts/agent-3-message-generator.txt`
- `prompts/agent-4-conversation-handler.txt`

Se precisar atualizar um prompt, edite o arquivo correspondente e atualize o system prompt do agente na plataforma Tess.

### 6.3. Como o Workflow Chama os Agentes

Cada chamada segue esse padrão:

```
POST https://api.tess.im/agents/{agent_id}/execute

Headers:
  Authorization: Bearer {token}

Body:
  { "input": "{dados do cliente ou mensagem}" }

Response:
  { "responses": [{ "output": "resposta do agente" }] }
```

O workflow faz o parsing de `responses[0].output` e aplica sanitização de JSON quando necessário (remove marcações ` ```json ``` ` que o modelo pode retornar).

---

## 🧪 Parte 7: Testar o Sistema End-to-End

### 7.1. Criar Dados de Teste no HubSpot (ou Mockar)

**Opção A: Mockar HubSpot (Mais Simples para Demo)**

1. No workflow, adicione um node **"Code"** antes dos nodes HubSpot
2. Nomeie: `Mock HubSpot Data`
3. Cole o código:

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
    }
  }
}];
```

4. Conecte `Set Contact ID` → `Mock HubSpot Data` → nó de consolidação
5. Desconecte os 4 nodes de HubSpot API real (ou deixe em paralelo para usar depois)

**Opção B: Usar HubSpot Real**

Se você tem uma conta HubSpot com contatos, pode usar os nodes reais diretamente. Certifique-se de que o token da credencial `HubSpot API` é válido.

### 7.2. Teste Manual - Envio Inicial

1. No workflow, clique no node **"Set Contact ID"**
2. Clique em **"Execute Node"**
3. No painel, cole:
```json
{
  "contact_id": "demo-12345"
}
```
4. Clique em **"Execute Node"**
5. 🎯 Acompanhe a execução percorrer todos os nodes:
   - Get Contact / Mock Data ✅
   - Get Emails / Get Deals / Get Tickets ✅
   - Consolidate Data ✅
   - AGENTE 2: Context Analyzer (Tess AI) ✅
   - AGENTE 3: Message Generator (Tess AI) ✅
   - Send WhatsApp (Meta API) ✅
   - Supabase Insert + Chatwoot Create (paralelo) ✅

6. **Verificar WhatsApp:**
   - Você deve receber uma mensagem no número conectado à Meta WhatsApp Business API
   - Exemplo: *"Oi João! Vi que você fez upgrade para o Plano Pro recentemente. Como está sendo a experiência? De 1 a 5, como você avalia nosso serviço?"*

7. **Verificar Supabase:**
   - Vá no Supabase → Table Editor → `surveys`
   - Deve ter 1 registro novo com `status = 'active'`

8. **Verificar Chatwoot:**
   - No Chatwoot dashboard, deve aparecer uma nova conversa com o contato
   - A mensagem enviada via WhatsApp deve estar registrada

### 7.3. Teste de Conversa - Responder no WhatsApp

1. Pegue seu celular (ou outro número para simular o cliente)
2. Responda à mensagem do bot, por exemplo:
   ```
   5! Estou muito satisfeito, a plataforma é excelente!
   ```

3. O webhook Meta vai disparar automaticamente e:
   - Webhook recebe a mensagem
   - Busca a survey ativa no Supabase (`status = 'active'`)
   - Monta histórico de conversa
   - AGENTE 4 (Tess AI) processa e gera resposta
   - Resposta enviada via Meta WhatsApp API
   - Chatwoot atualizado com a mensagem

4. Continue a conversa:
   - Bot pode perguntar: *"Que ótimo saber! O que você mais gosta?"*
   - Você responde: *"A facilidade de usar as automações"*
   - Bot encerra: *"Obrigado pelo feedback, João! Vamos continuar melhorando 😊"*

5. **Verificar finalização:**
   - Supabase → `surveys` → `status = 'completed'`
   - `satisfaction_score = 5`
   - `conversation_transcript` com todo o histórico
   - Chatwoot → conversa marcada como resolvida + nota privada com o score

### 7.4. Verificar Logs Completos

1. No n8n, clique em **"Executions"** (menu lateral)
2. Você verá todas as execuções:
   - Primeira execução (trigger manual — envio inicial)
   - Segunda execução (webhook — primeira resposta do cliente)
   - Terceira execução (webhook — segunda resposta)
   - Etc.

3. Clique em qualquer execução para ver:
   - Dados de entrada/saída de cada node
   - Tempo de execução
   - Erros (se houver)

---

## 🚨 Troubleshooting

### Problema: Webhook Meta não recebe mensagens

**Sintoma:** Mensagem enviada pelo cliente, mas workflow não dispara

**Checklist:**
1. ✅ Webhook verificado no Meta for Developers? (Callback URL + Verify Token)
2. ✅ Subscription field `messages` está habilitado?
3. ✅ Workflow está "Active" no n8n?
4. ✅ A URL do webhook no Meta corresponde à Production URL do node Webhook no n8n?

**Debug:**
- No Meta for Developers → WhatsApp → Configuration → Webhooks, verifique o status
- No n8n → Executions, verifique se há execuções recentes do webhook

### Problema: Tess AI retorna erro ou JSON inválido

**Sintoma:** Agente retorna texto com markdown ou formato errado

**Solução:**
1. O workflow já possui sanitização: remove marcações ` ```json ``` ` da resposta
2. Verifique se o agente existe no Tess AI com o ID correto (38717, 38728, 38733)
3. Verifique se o token da credencial `Tess AI` é válido
4. Conferir créditos disponíveis na conta Tess AI

### Problema: Chatwoot não aparece a conversa

**Sintoma:** Mensagem enviada, Supabase atualizado, mas Chatwoot vazio

**Checklist:**
1. ✅ Credencial `Chatwoot Cloud API` com token correto?
2. ✅ Account ID e Inbox ID nos nodes Chatwoot são os corretos?
3. ✅ O contato existe no Chatwoot? (o workflow faz Search → Create se não existir)

### Problema: Supabase rejeita INSERT (RLS)

**Sintoma:** Erro 403 ou policy violation

**Solução:**
```sql
-- Verificar status do RLS
SELECT tablename, rowsecurity
FROM pg_tables
WHERE tablename = 'surveys';

-- Desabilitar se necessário (apenas para demo)
ALTER TABLE surveys DISABLE ROW LEVEL SECURITY;
```

### Problema: Meta WhatsApp API retorna erro 401

**Sintoma:** Erro na hora de enviar mensagem via WhatsApp

**Solução:**
1. Verifique se o Access Token não expirou (tokens de curta duração expiram em 1h)
2. Use um **Permanent Access Token** ou token de longa duração
3. Verifique se o Phone Number ID está correto nas chamadas API

---

## 🔗 Recursos e Links Úteis

- **n8n Cloud Docs:** https://docs.n8n.io/hosting/cloud/
- **Supabase Docs:** https://supabase.com/docs
- **Tess AI:** https://tess.im
- **Meta WhatsApp Business API Docs:** https://developers.facebook.com/docs/whatsapp/cloud-api
- **Chatwoot Docs:** https://www.chatwoot.com/docs/
- **HubSpot API Docs:** https://developers.hubspot.com/

---

## 💬 Suporte

Para problemas durante o deploy:

1. Revisar seção de Troubleshooting deste guia
2. Consultar documentação oficial das plataformas
3. Verificar logs em cada serviço:
   - n8n: Executions → detalhes
   - Supabase: Logs (menu lateral)
   - Chatwoot: Settings → Audit Log

---

**Última atualização:** 2026-02-04
**Versão:** 2.0 (Meta WhatsApp API + Tess AI + Chatwoot Cloud)
**Status:** Pronto para deploy
