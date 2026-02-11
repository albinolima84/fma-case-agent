# Workflow n8n - Sistema de Pesquisa de Satisfação

Este diretório contém o workflow completo do n8n que implementa o sistema automatizado de pesquisa de satisfação com agentes de IA.

## 📁 Arquivos

- `satisfaction-survey-workflow.json` - Workflow completo exportado do n8n
- `.env.example` - Template de variáveis de ambiente
- `workflow-diagram.md` - Diagrama visual explicativo
- `test-example.md` - Exemplo de teste com dados simulados

---

## 🚀 Como Importar o Workflow

### 1. Pré-requisitos

- n8n instalado (Docker recomendado)
- Contas e API keys:
  - HubSpot API Key
  - Tess AI API Key (gpt-4o-mini)
  - Supabase (project URL + anon key)
  - Evolution API (instância configurada)
  - Chatwoot (account + API token)

### 2. Instalação do n8n via Docker

```bash
# Clone o repositório
git clone <repo-url>
cd case-agent-dev

# Copie o template de variáveis de ambiente
cp workflows/.env.example .env

# Edite o .env com suas credenciais
nano .env

# Inicie o n8n com Docker Compose
docker-compose up -d n8n
```

### 3. Importar o Workflow

1. Acesse n8n em `http://localhost:5678`
2. Faça login ou crie uma conta
3. Clique em **"Add workflow" → "Import from File"**
4. Selecione o arquivo `satisfaction-survey-workflow.json`
5. O workflow será importado com todos os nodes configurados

### 4. Configurar Credenciais

Após importar, você precisa configurar as credenciais para cada integração:

#### HubSpot API
1. Vá em **Settings → Credentials**
2. Clique em **"Add Credential" → "Header Auth"**
3. Nome: `HubSpot API Key`
4. Configure:
   - **Header Name:** `Authorization`
   - **Header Value:** `Bearer YOUR_HUBSPOT_API_KEY`

#### Tess AI API (gpt-4o-mini)
As chamadas à Tess AI estão configuradas como HTTP Request genérico usando variáveis de ambiente:
- Configure `TESS_API_KEY` no arquivo `.env`

#### Supabase
1. Vá em **Settings → Credentials**
2. Clique em **"Add Credential" → "Supabase"**
3. Configure:
   - **Host:** `https://your-project.supabase.co`
   - **Service Role Secret:** `your-supabase-anon-key`

#### Chatwoot API
1. Vá em **Settings → Credentials**
2. Clique em **"Add Credential" → "Header Auth"**
3. Nome: `Chatwoot API Token`
4. Configure:
   - **Header Name:** `api_access_token`
   - **Header Value:** `YOUR_CHATWOOT_TOKEN`

---

## 🔧 Configuração de Prompts

O workflow lê os prompts dos agentes de arquivos externos. Você precisa disponibilizar os arquivos de prompts:

### Opção 1: Montar Volume no Docker

```yaml
# docker-compose.yml
services:
  n8n:
    volumes:
      - ./prompts:/data/prompts
```

### Opção 2: Usar Variáveis de Ambiente

Altere os nodes `Read Prompt - Agent X` para usar variáveis de ambiente em vez de ler arquivos:

```javascript
// Substitua o node de leitura de arquivo por:
return {
  json: {
    system_prompt: process.env.AGENT_2_PROMPT
  }
};
```

E configure no `.env`:
```bash
AGENT_2_PROMPT="Você é um Analista de Relacionamento..."
AGENT_3_PROMPT="Você é uma Gerente de Relacionamento..."
AGENT_4_PROMPT="Você é Sofia, Gerente de Relacionamento..."
```

---

## 🎯 Execução do Workflow

### Execução Manual (Teste)

1. Abra o workflow no n8n
2. No node `Set Contact ID`, clique em **"Execute Node"**
3. No painel de execução, insira um `contact_id` de teste:
   ```json
   {
     "contact_id": "12345"
   }
   ```
4. Clique em **"Execute Workflow"**
5. Acompanhe a execução node por node

### Execução Automática (Produção)

O workflow está configurado com dois triggers:

#### 1. Schedule Trigger (Automático)
- **Frequência:** Diariamente às 10h da manhã
- **Cron:** `0 10 * * *`
- **Função:** Processa lista de clientes agendados
- **Ativação:** Toggle "Active" no node Schedule Trigger

#### 2. Webhook Trigger (Respostas)
- **URL:** `http://localhost:5678/webhook/webhook-whatsapp-response`
- **Método:** POST
- **Função:** Recebe respostas dos clientes via Evolution API
- **Ativação:** Sempre ativo quando workflow está ativo

### Configurar Evolution API para Enviar Webhooks

Configure o webhook na Evolution API para enviar respostas para o n8n:

```bash
# Configurar webhook na Evolution API
curl -X POST http://evolution-api:8080/webhook/set/instance \
  -H "Content-Type: application/json" \
  -d '{
    "url": "http://n8n:5678/webhook/webhook-whatsapp-response",
    "webhook_by_events": true,
    "webhook_base64": false,
    "events": ["messages.upsert"]
  }'
```

---

## 📊 Estrutura do Workflow

### Parte 1: Envio Inicial da Pesquisa

```
Schedule Trigger
    ↓
Set Contact ID
    ↓
[HubSpot APIs - Paralelo]
├─ Get Contact
├─ Get Emails
├─ Get Deals
├─ Get Tickets
└─ Get Notes
    ↓
AGENTE 1: Data Fetcher (consolida dados)
    ↓
AGENTE 2: Context Analyzer (Tess AI - análise)
    ↓
Validate Analysis (validação + fallback)
    ↓
AGENTE 3: Message Generator (Tess AI - mensagem)
    ↓
Prepare Message
    ↓
[Envio Paralelo]
├─ Send WhatsApp Message (Evolution API)
├─ Supabase - Insert Survey (registra pesquisa)
└─ Chatwoot - Create Conversation (cria conversa)
```

### Parte 2: Gerenciamento de Respostas

```
Webhook - WhatsApp Response (recebe resposta)
    ↓
Parse Webhook (extrai dados)
    ↓
Supabase - Get Survey (busca pesquisa ativa)
    ↓
Build Conversation History (monta histórico)
    ↓
AGENTE 4: Conversation Handler (Tess AI - resposta)
    ↓
Process Conversation Response (processa e decide)
    ↓
IF - Continue Conversation?
    ├─ SIM (should_continue = true)
    │   ├─ Send WhatsApp Response (envia resposta)
    │   └─ Supabase - Update Ongoing (atualiza DB)
    │       └─ [LOOP - aguarda próxima resposta]
    │
    └─ NÃO (should_continue = false)
        ↓
    Supabase - Update Completed (finaliza pesquisa)
        ↓
    IF - Escalate to Human?
        ├─ SIM (status = escalate)
        │   └─ Chatwoot - Assign to Human
        │
        └─ NÃO (status = completed)
            └─ FIM
```

---

## 🧪 Testando o Workflow

### Teste 1: Envio Inicial (Sem APIs Reais)

Use o node `Set Contact ID` com dados mockados:

```json
{
  "contact_id": "test-12345",
  "mock_mode": true
}
```

Adicione um node `IF` após `Set Contact ID` para detectar `mock_mode` e retornar dados simulados.

### Teste 2: Resposta do Cliente (Webhook)

Simule uma resposta do cliente enviando um POST para o webhook:

```bash
curl -X POST http://localhost:5678/webhook/webhook-whatsapp-response \
  -H "Content-Type: application/json" \
  -d '{
    "event": "messages.upsert",
    "data": {
      "key": {
        "remoteJid": "5511999999999@s.whatsapp.net"
      },
      "message": {
        "conversation": "5! Estou muito satisfeito"
      },
      "messageTimestamp": "1706400000"
    }
  }'
```

### Teste 3: Fluxo Completo End-to-End

Para testar o fluxo completo com APIs reais:

1. Configure todas as credenciais
2. Use um contato de teste no HubSpot
3. Use seu próprio número de WhatsApp
4. Execute o workflow manualmente
5. Responda as mensagens no WhatsApp
6. Verifique os dados no Supabase e Chatwoot

---

## 📈 Monitoramento e Logs

### Executions (Histórico de Execuções)

No n8n, acesse **Executions** para ver todas as execuções do workflow:
- ✅ Sucesso (verde)
- ❌ Erro (vermelho)
- ⏸️ Aguardando (amarelo)

Clique em qualquer execução para ver:
- Dados de entrada/saída de cada node
- Tempo de execução
- Erros e stack traces

### Logs do Container

Para ver logs em tempo real:

```bash
# Logs do n8n
docker-compose logs -f n8n

# Logs de um workflow específico
docker exec n8n cat /home/node/.n8n/logs/n8n.log
```

### Alertas

Configure webhooks de erro no n8n:

1. Vá em **Settings → Workflows → Error Workflow**
2. Crie um workflow que:
   - Recebe erros de outros workflows
   - Envia notificação via Slack/Email
   - Registra no Supabase

---

## 🔒 Segurança

### Variáveis de Ambiente

Nunca commite o arquivo `.env` com credenciais reais:

```bash
# .gitignore
.env
workflows/.env
```

### Webhooks Públicos

O webhook `webhook-whatsapp-response` é público por padrão. Para produção:

1. **Opção A:** Adicione autenticação básica:
   ```javascript
   // No node "Webhook - WhatsApp Response"
   // Adicione validação de token
   const receivedToken = $json.headers['x-webhook-token'];
   if (receivedToken !== process.env.WEBHOOK_SECRET) {
     throw new Error('Unauthorized');
   }
   ```

2. **Opção B:** Use IP whitelist no n8n

3. **Opção C:** Configure webhook privado via VPN/Tunnel

### Credenciais no n8n

O n8n criptografa todas as credenciais armazenadas. Certifique-se de:
- Usar senhas fortes no n8n
- Habilitar 2FA se disponível
- Fazer backup das credenciais separadamente

---

## 📊 Métricas e Performance

### Tempo de Execução Esperado

| Etapa | Tempo Médio |
|-------|-------------|
| HubSpot APIs (paralelo) | 3-5s |
| Agente 1 (Data Fetcher) | 0.5s |
| Agente 2 (Context Analyzer) | 2-4s |
| Agente 3 (Message Generator) | 1-2s |
| Envio WhatsApp + DB | 1-2s |
| **Total (envio inicial)** | **8-14s** |
| Agente 4 (por turno) | 2-3s |
| **Total por conversa (3 turnos)** | **~25-35s** |

### Custos de API

Por pesquisa completa (200 pesquisas/mês):

| API | Custo/Pesquisa | Custo/Mês |
|-----|----------------|-----------|
| HubSpot | $0 (incluído) | $0 |
| Tess AI/gpt-4o-mini (Agente 2) | $0.006 | $1.20 |
| Tess AI/gpt-4o-mini (Agente 3) | $0.003 | $0.60 |
| Tess AI/gpt-4o-mini (Agente 4, 3 turnos) | $0.010 | $2.00 |
| **Total** | **$0.019** | **$3.80** |

### Limites e Throttling

- **HubSpot:** 100 req/10s (API Enterprise)
- **Tess AI:** 50 req/min
- **Evolution API:** Sem limite (self-hosted)

Configurar retry e backoff exponencial nos nodes HTTP Request:
```json
{
  "options": {
    "timeout": 30000,
    "retry": {
      "maxRetries": 3,
      "retryOnHttpStatus": [429, 500, 502, 503, 504]
    }
  }
}
```

---

## 🐛 Troubleshooting

### Erro: "HubSpot API Key Invalid"

**Solução:**
1. Verifique se a API key está correta no `.env`
2. Confirme que a key tem permissões para ler contatos, emails, deals, tickets, notes
3. Teste a key manualmente:
   ```bash
   curl https://api.hubapi.com/crm/v3/objects/contacts \
     -H "Authorization: Bearer YOUR_KEY"
   ```

### Erro: "Tess AI API 401 Unauthorized"

**Solução:**
1. Verifique a API key da Tess AI
2. Confirme que a key está ativa e tem créditos
3. Teste manualmente:
   ```bash
   curl -X POST https://api.tess.im/agents/38717/execute \
     -H "Authorization: Bearer YOUR_KEY" \
     -H "Content-Type: application/json" \
     -d '{"input": "teste"}'
   ```

### Erro: "Evolution API Connection Refused"

**Solução:**
1. Verifique se o container está rodando:
   ```bash
   docker ps | grep evolution
   ```
2. Teste a API:
   ```bash
   curl http://localhost:8080/instance/fetchInstances
   ```
3. Reconecte o WhatsApp se necessário

### Erro: "Webhook Not Receiving Responses"

**Solução:**
1. Verifique se o webhook está ativo no n8n
2. Teste manualmente com curl (ver seção de testes)
3. Verifique os logs do Evolution API:
   ```bash
   docker-compose logs evolution-api | grep webhook
   ```
4. Confirme que a URL do webhook está correta na Evolution API

### Erro: "Supabase RLS (Row Level Security)"

**Solução:**
1. Desabilite RLS temporariamente para testes:
   ```sql
   ALTER TABLE satisfaction_surveys DISABLE ROW LEVEL SECURITY;
   ```
2. Ou configure políticas corretas:
   ```sql
   CREATE POLICY "Allow n8n service role"
   ON satisfaction_surveys
   FOR ALL
   TO service_role
   USING (true);
   ```

---

## 🔄 Atualizações e Manutenção

### Versionamento do Workflow

Sempre exporte uma nova versão após mudanças:

```bash
# Exporte via UI: Workflow → Export → Download
# Ou via CLI (se disponível)
n8n export:workflow --id=<workflow-id> --output=./workflows/
```

Nomeie com versionamento:
- `satisfaction-survey-workflow-v1.0.json`
- `satisfaction-survey-workflow-v1.1.json`

### Changelog

Mantenha um registro de mudanças:

```markdown
## v1.1 (2026-02-01)
- Adicionado retry automático em falhas de API
- Melhorado tratamento de erros no Agente 4
- Otimizado paralelização de chamadas HubSpot

## v1.0 (2026-01-28)
- Versão inicial do workflow
- 4 agentes funcionais
- Integração completa HubSpot → Tess AI → WhatsApp
```

---

## 📚 Recursos Adicionais

- [Documentação oficial do n8n](https://docs.n8n.io/)
- [HubSpot API Reference](https://developers.hubspot.com/docs/api/overview)
- [Tess AI Documentation](https://tess.im/pt-BR/docs)
- [Evolution API Docs](https://doc.evolution-api.com/)
- [Chatwoot API Docs](https://www.chatwoot.com/developers/api/)

---

## 💬 Suporte

Para dúvidas ou problemas:
1. Revise esta documentação
2. Consulte os logs do n8n
3. Verifique o PROJECT_STATUS.md do projeto
4. Entre em contato com o time técnico

---

**Última atualização:** 2026-01-28
**Versão do workflow:** 1.0
**Compatível com:** n8n v1.0+
