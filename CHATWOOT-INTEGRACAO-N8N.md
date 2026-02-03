# Integração Chatwoot Cloud com n8n

**Data:** 2026-02-02
**Status:** Em implementação
**Decisão:** Usar Chatwoot Cloud após Render Free tier falhar por falta de memória

---

## 🔑 Credenciais Chatwoot Cloud

```
URL: https://app.chatwoot.com
Account ID: 150655
Inbox ID: 94417
API Access Token: YfdfiZa8YPNM6rhBxpGmdK9M
Workspace: Pareto
```

**⚠️ Importante:** Estas credenciais são sensíveis. Não compartilhar publicamente.

---

## 📋 Passo 1: Configurar Credencial no n8n

### 1.1 Acessar n8n Cloud

- URL: https://albino.app.n8n.cloud
- Login

### 1.2 Criar Credencial Chatwoot

**Opção A: Usar HTTP Request (Recomendado)**

n8n não tem node nativo do Chatwoot, então usaremos HTTP Request com autenticação.

1. **Credentials → Add Credential → Header Auth**
   - Name: `Chatwoot Cloud API`
   - Header Name: `api_access_token`
   - Value: `YfdfiZa8YPNM6rhBxpGmdK9M`
   - Save

**Opção B: Usar Generic Credential Type**

1. **Credentials → Add Credential → Generic Credential Type**
   - Credential Type: `chatwootApi`
   - Name: `Chatwoot Cloud`
   - Fields:
     - `apiAccessToken`: `YfdfiZa8YPNM6rhBxpGmdK9M`
     - `accountId`: `150655`
     - `inboxId`: `94417`
   - Save

---

## 🔗 Passo 2: Endpoints da API Chatwoot

Base URL: `https://app.chatwoot.com/api/v1/accounts/150655`

### Endpoints principais:

**1. Criar/Buscar Contato:**
```
POST /contacts
GET /contacts/search?q={phone}
```

**2. Criar Conversa:**
```
POST /conversations
```

**3. Enviar Mensagem:**
```
POST /conversations/{conversation_id}/messages
```

**4. Atualizar Conversa:**
```
PATCH /conversations/{conversation_id}
```

---

## 🛠️ Passo 3: Nodes a Criar no Workflow n8n

### FLUXO 1 - Envio Proativo

#### Node 1: Create/Find Chatwoot Contact

**Tipo:** HTTP Request

**Configuração:**
```
Method: POST
URL: https://app.chatwoot.com/api/v1/accounts/150655/contacts
Authentication: Header Auth (usar credencial criada)

Headers:
- Content-Type: application/json

Body (JSON):
{
  "inbox_id": 94417,
  "name": "{{ $json.customer_name }}",
  "phone_number": "{{ $json.customer_phone }}",
  "custom_attributes": {
    "hubspot_id": "{{ $json.hubspot_contact_id }}",
    "context_summary": "{{ $json.context_summary }}"
  }
}
```

#### Node 2: Create Chatwoot Conversation

**Tipo:** HTTP Request

**Configuração:**
```
Method: POST
URL: https://app.chatwoot.com/api/v1/accounts/150655/conversations
Authentication: Header Auth

Body (JSON):
{
  "source_id": "{{ $json.customer_phone }}",
  "inbox_id": 94417,
  "contact_id": "{{ $node['Create Chatwoot Contact'].json.payload.contact.id }}",
  "status": "open",
  "custom_attributes": {
    "survey_id": "{{ $json.survey_id }}",
    "type": "satisfaction_survey"
  }
}
```

#### Node 3: Send Initial Message to Chatwoot

**Tipo:** HTTP Request

**Configuração:**
```
Method: POST
URL: https://app.chatwoot.com/api/v1/accounts/150655/conversations/{{ $node['Create Chatwoot Conversation'].json.id }}/messages
Authentication: Header Auth

Body (JSON):
{
  "content": "{{ $json.message }}",
  "message_type": "outgoing",
  "private": false
}
```

---

### FLUXO 2 - Recebimento de Respostas

#### Node 1: Send User Message to Chatwoot

**Posição:** Após "Parse Meta Message"

**Tipo:** HTTP Request

**Configuração:**
```
Method: POST
URL: https://app.chatwoot.com/api/v1/accounts/150655/conversations/{{ $json.chatwoot_conversation_id }}/messages
Authentication: Header Auth

Body (JSON):
{
  "content": "{{ $json.message_content }}",
  "message_type": "incoming",
  "private": false,
  "sender": {
    "id": "{{ $json.contact_id }}"
  }
}
```

#### Node 2: Send Bot Response to Chatwoot

**Posição:** Após "Send Meta WhatsApp"

**Tipo:** HTTP Request

**Configuração:**
```
Method: POST
URL: https://app.chatwoot.com/api/v1/accounts/150655/conversations/{{ $json.chatwoot_conversation_id }}/messages
Authentication: Header Auth

Body (JSON):
{
  "content": "{{ $node['Tess - Agent 4'].json.response }}",
  "message_type": "outgoing",
  "private": false
}
```

#### Node 3: Update Conversation on Completion

**Posição:** Após detectar status "completed"

**Tipo:** HTTP Request

**Configuração:**
```
Method: PATCH
URL: https://app.chatwoot.com/api/v1/accounts/150655/conversations/{{ $json.chatwoot_conversation_id }}
Authentication: Header Auth

Body (JSON):
{
  "status": "resolved",
  "custom_attributes": {
    "satisfaction_score": "{{ $json.satisfaction_score }}",
    "sentiment": "{{ $json.sentiment }}",
    "completed_at": "{{ $now }}"
  }
}

Adicionar nota privada (separado):
POST /conversations/{{ $json.chatwoot_conversation_id }}/messages
{
  "content": "Survey concluído - Nota: {{ $json.satisfaction_score }}/5 - Sentiment: {{ $json.sentiment }}",
  "message_type": "outgoing",
  "private": true
}
```

---

## 🔄 Passo 4: Modificações no Supabase

Adicionar coluna para armazenar Chatwoot conversation ID:

```sql
ALTER TABLE surveys
ADD COLUMN chatwoot_conversation_id INTEGER;

CREATE INDEX idx_surveys_chatwoot ON surveys(chatwoot_conversation_id);
```

---

## 🧪 Passo 5: Teste de Integração

### Teste 1: Criar Contato

**Endpoint:**
```bash
curl -X POST https://app.chatwoot.com/api/v1/accounts/150655/contacts \
  -H "api_access_token: YfdfiZa8YPNM6rhBxpGmdK9M" \
  -H "Content-Type: application/json" \
  -d '{
    "inbox_id": 94417,
    "name": "Teste n8n",
    "phone_number": "+5521981444992"
  }'
```

**Resultado esperado:**
```json
{
  "payload": {
    "contact": {
      "id": 123456,
      "name": "Teste n8n",
      "phone_number": "+5521981444992"
    }
  }
}
```

### Teste 2: Criar Conversa

```bash
curl -X POST https://app.chatwoot.com/api/v1/accounts/150655/conversations \
  -H "api_access_token: YfdfiZa8YPNM6rhBxpGmdK9M" \
  -H "Content-Type: application/json" \
  -d '{
    "inbox_id": 94417,
    "contact_id": 123456,
    "status": "open"
  }'
```

### Teste 3: Enviar Mensagem

```bash
curl -X POST https://app.chatwoot.com/api/v1/accounts/150655/conversations/{conversation_id}/messages \
  -H "api_access_token: YfdfiZa8YPNM6rhBxpGmdK9M" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "Mensagem de teste do n8n",
    "message_type": "outgoing"
  }'
```

---

## 📊 Passo 6: Visualizar no Chatwoot

1. **Acessar:** https://app.chatwoot.com
2. **Inbox:** Satisfação WhatsApp
3. **Ver conversas:** Todas as mensagens devem aparecer em tempo real
4. **Intervir manualmente:** Clicar em conversa → Assumir → Enviar mensagem

---

## ⚠️ Error Handling

Todos os requests ao Chatwoot devem ter error handling:

```javascript
// Em Code nodes do n8n
try {
  // Salvar no Supabase (SEMPRE PRIMEIRO - crítico)
  await supabase.insert(survey_data);

  // Tentar enviar para Chatwoot (best effort)
  try {
    const response = await chatwoot.createMessage(data);
    return { chatwoot_id: response.id };
  } catch (chatwootError) {
    console.log('Chatwoot falhou, mas dados salvos no Supabase');
    return { chatwoot_id: null, error: chatwootError.message };
  }
} catch (supabaseError) {
  // Só falha workflow se Supabase falhar
  throw supabaseError;
}
```

---

## 📝 Checklist de Implementação

- [ ] Criar credencial Header Auth no n8n
- [ ] Adicionar coluna `chatwoot_conversation_id` no Supabase
- [ ] Criar node "Create Chatwoot Contact" no FLUXO 1
- [ ] Criar node "Create Chatwoot Conversation" no FLUXO 1
- [ ] Criar node "Send Initial Message" no FLUXO 1
- [ ] Criar node "Send User Message" no FLUXO 2
- [ ] Criar node "Send Bot Response" no FLUXO 2
- [ ] Criar node "Update Conversation" no FLUXO 2 (finalização)
- [ ] Testar criação de contato
- [ ] Testar criação de conversa
- [ ] Testar envio de mensagem
- [ ] Testar fluxo completo end-to-end
- [ ] Validar visualização no Chatwoot dashboard

---

## 🔗 Documentação de Referência

- **API Docs:** https://www.chatwoot.com/developers/api/
- **n8n HTTP Request:** https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/
- **Chatwoot Inbox:** https://app.chatwoot.com/app/accounts/150655/inbox/94417

---

**Próximo passo:** Configurar credencial no n8n e criar primeiro node de teste
