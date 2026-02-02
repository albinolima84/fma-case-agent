# Guia de Configuração - Meta WhatsApp API

**Data:** 2026-02-01
**Status:** Workflow pronto para importar

---

## 📋 Informações da API Meta

```
Phone Number ID: 674094992450703
WhatsApp Business Account ID: 1255054259608433
Access Token: [Você precisa gerar um novo - veja instruções abaixo]
API Version: v21.0
```

---

## 🚀 PASSO 1: Importar Workflow no n8n

1. Acesse: https://albino.app.n8n.cloud
2. Clique em **Workflows** → **Import from File**
3. Selecione o arquivo: `workflows/satisfaction-survey-meta-api.json`
4. Clique em **Import**

---

## 🔑 PASSO 2: Criar Credencial Meta WhatsApp

1. No n8n, vá em **Credentials** → **Add Credential**
2. Busque por **"Header Auth"** ou **"HTTP Header Auth"**
3. Configure:
   - **Name:** `Meta WhatsApp Token`
   - **Header Name:** `Authorization`
   - **Value:** `Bearer SEU_ACCESS_TOKEN_AQUI`
4. Clique em **Save**

### Como gerar Access Token válido:

**Opção A: Token Temporário (24h - apenas para testes)**
1. Acesse: https://developers.facebook.com/apps
2. Selecione seu App WhatsApp
3. Vá em **WhatsApp** → **API Setup**
4. Copie o **"Temporary access token"**

**Opção B: Token Permanente (recomendado)**
1. Acesse: https://business.facebook.com/
2. Vá em **Business Settings** → **Users** → **System Users**
3. Clique em **Add** para criar novo System User (ou selecione existente)
4. Nome: `n8n-whatsapp-integration`
5. Role: **Admin**
6. Clique em **Add Assets** → **Apps**
7. Selecione seu App WhatsApp → **Full Control**
8. Clique em **Generate New Token**
9. Selecione o App
10. Marque as permissões:
    - ✅ `whatsapp_business_messaging`
    - ✅ `whatsapp_business_management`
11. Expiration: **Never** (nunca expira)
12. Clique em **Generate Token**
13. **COPIE O TOKEN** (não será mostrado novamente!)

---

## 🔗 PASSO 3: Associar Credencial aos Nodes

No workflow importado, configure a credencial nos seguintes nodes:

### Node: "Send Meta WhatsApp"
1. Abra o node
2. Em **Authentication**, selecione: `Generic Credential Type`
3. Em **Generic Auth Type**, selecione: `Header Auth`
4. Em **Credential for Header Auth**, selecione: `Meta WhatsApp Token`
5. Salve

---

## 📡 PASSO 4: Configurar Webhook na Meta

1. Acesse: https://developers.facebook.com/apps
2. Selecione seu App WhatsApp
3. Vá em **WhatsApp** → **Configuration**
4. Na seção **Webhooks**, clique em **Edit**

Configure:
```
Callback URL: https://albino.app.n8n.cloud/webhook/whatsapp-incoming
Verify Token: satisfaction-survey-2026
```

5. Clique em **Verify and Save**
   - ✅ Deve aparecer "Success!"

6. **Subscribe to webhook fields:**
   - ✅ Marque: `messages`

7. Clique em **Save**

---

## ✅ PASSO 5: Ativar Workflow

1. No n8n, abra o workflow importado
2. Clique no toggle **"Active"** (canto superior direito)
3. Deve aparecer: **"Workflow activated"**

---

## 🧪 PASSO 6: Testar FLUXO 2 (Recebimento de Mensagens)

### Pré-requisito: Ter survey ativo no banco

Execute o FLUXO 1 primeiro (veja seção abaixo) para criar um survey ativo.

### Teste:

1. **DE OUTRO NÚMERO** (não do número conectado), envie mensagem WhatsApp para:
   ```
   +55 21 98144-4992
   ```

2. Exemplo de mensagem:
   ```
   5 - Excelente atendimento!
   ```

3. **Verifique no n8n:**
   - Deve aparecer nova execução do workflow
   - Todos os nodes devem executar com sucesso ✅

4. **Você deve receber resposta do bot** processada pelo Agent 4

---

## 📤 FLUXO 1: Envio Proativo (Templates)

⚠️ **IMPORTANTE:** A Meta exige usar **Template Messages** para enviar mensagens proativas (primeira mensagem para o cliente).

### Criar Template na Meta:

1. Acesse: https://business.facebook.com/wa/manage/message-templates/
2. Clique em **Create Template**
3. Configure:
   - **Category:** `UTILITY`
   - **Name:** `satisfaction_survey`
   - **Languages:** Portuguese (BR)

4. **Message Body:**
   ```
   Olá {{1}}, tudo bem? 😊

   Percebemos que você abriu alguns tickets recentemente. Como está sua experiência com nossos serviços?

   Poderia avaliar de 1 a 5 seu nível de satisfação?
   ```

5. **Variables:**
   - {{1}} = Nome do cliente

6. Clique em **Submit**
7. Aguarde aprovação (~15 minutos)

### Enviar Template via API:

```bash
curl -X POST "https://graph.facebook.com/v21.0/674094992450703/messages" \
  -H "Authorization: Bearer SEU_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "messaging_product": "whatsapp",
    "to": "5521981444992",
    "type": "template",
    "template": {
      "name": "satisfaction_survey",
      "language": {
        "code": "pt_BR"
      },
      "components": [
        {
          "type": "body",
          "parameters": [
            {
              "type": "text",
              "text": "Carlos Mendes"
            }
          ]
        }
      ]
    }
  }'
```

---

## 🔍 Troubleshooting

### Erro: "The callback URL or verify token couldn't be validated"

**Causa:** Webhook não está respondendo corretamente ao desafio da Meta

**Solução:**
1. Certifique-se que o workflow está **ATIVO**
2. Verifique se o node "Check Verification" está configurado corretamente
3. Verifique se o node "Respond Verification" retorna: `{{ $json.query['hub.challenge'] }}`
4. Teste manualmente:
   ```bash
   curl "https://albino.app.n8n.cloud/webhook/whatsapp-incoming?hub.mode=subscribe&hub.verify_token=satisfaction-survey-2026&hub.challenge=12345"
   ```
   Deve retornar: `12345`

---

### Erro: "The access token could not be decrypted"

**Causa:** Access Token inválido ou expirado

**Solução:**
1. Gere novo Access Token (veja PASSO 2)
2. Atualize a credencial no n8n
3. Teste novamente

---

### Erro: "(#131030) Recipient phone number not in allowed list"

**Causa:** Número não está na lista de testes (apenas em ambiente de desenvolvimento)

**Solução:**
1. Acesse: https://developers.facebook.com/apps
2. Vá em **WhatsApp** → **API Setup**
3. Em **"To"**, clique em **"Manage phone number list"**
4. Adicione o número de teste
5. Ou solicite aprovação do App para produção

---

### Webhook não dispara ao enviar mensagem

**Verificações:**

1. **Webhook está subscrito?**
   - Vá em WhatsApp → Configuration → Webhooks
   - Verifique se `messages` está marcado

2. **Workflow está ativo?**
   - Toggle "Active" deve estar ON no n8n

3. **Survey existe no banco?**
   - Execute query: `SELECT * FROM surveys WHERE customer_phone = '+5521981444992' AND status = 'active'`
   - Deve retornar pelo menos 1 registro

4. **Está enviando do número correto?**
   - Não envie do próprio número conectado
   - Use outro WhatsApp

---

## 📊 Estrutura do Webhook da Meta

### Verificação (GET):
```
GET /webhook/whatsapp-incoming?hub.mode=subscribe&hub.verify_token=TOKEN&hub.challenge=CHALLENGE
```

Resposta esperada: `CHALLENGE`

### Mensagem Recebida (POST):
```json
{
  "object": "whatsapp_business_account",
  "entry": [{
    "id": "WHATSAPP_BUSINESS_ACCOUNT_ID",
    "changes": [{
      "value": {
        "messaging_product": "whatsapp",
        "metadata": {
          "display_phone_number": "PHONE_NUMBER",
          "phone_number_id": "PHONE_NUMBER_ID"
        },
        "contacts": [{
          "profile": {
            "name": "NAME"
          },
          "wa_id": "WHATSAPP_ID"
        }],
        "messages": [{
          "from": "SENDER_PHONE_NUMBER",
          "id": "MESSAGE_ID",
          "timestamp": "TIMESTAMP",
          "text": {
            "body": "MESSAGE_TEXT"
          },
          "type": "text"
        }]
      },
      "field": "messages"
    }]
  }]
}
```

---

## ✅ Checklist Final

Antes de ir para produção:

- [ ] Access Token permanente criado
- [ ] Credencial configurada no n8n
- [ ] Webhook verificado e salvo na Meta
- [ ] Webhook subscrito ao campo `messages`
- [ ] Workflow ativo no n8n
- [ ] Template aprovado pela Meta (para FLUXO 1)
- [ ] Survey de teste criado no Supabase
- [ ] Teste de envio e recebimento funcionando

---

**Última atualização:** 2026-02-01
**Arquivo workflow:** `workflows/satisfaction-survey-meta-api.json`
