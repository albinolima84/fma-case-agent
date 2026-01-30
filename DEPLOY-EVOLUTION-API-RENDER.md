# Deploy Evolution API na Render - Guia Rápido

**Objetivo:** Subir Evolution API na Render (free tier) para usar no projeto

---

## 🚀 Passo 1: Criar Conta na Render

1. Acesse https://render.com
2. Clique em **"Get Started for Free"**
3. Faça login com **GitHub** (recomendado)

---

## 🚀 Passo 2: Deploy da Evolution API

### Opção A: Deploy via GitHub Fork (Recomendado)

1. **Fork o repositório:**
   - Acesse: https://github.com/EvolutionAPI/evolution-api
   - Clique em **"Fork"** (canto superior direito)
   - Confirme o fork para sua conta

2. **No Render Dashboard:**
   - Clique em **"New +"** → **"Web Service"**
   - Selecione **"Build and deploy from a Git repository"**
   - Clique em **"Connect GitHub"** (se não conectou ainda)
   - Selecione o repositório **"evolution-api"** que você acabou de forkar

3. **Configure o serviço:**
   - **Name:** `evolution-api-demo`
   - **Region:** `Oregon (US West)` (free tier disponível)
   - **Branch:** `main`
   - **Runtime:** Detectado automaticamente como `Docker`
   - **Instance Type:** **Free**

4. **Environment Variables** (clique em "Advanced"):

⚠️ **ATENÇÃO:** Evolution API **REQUER** banco de dados PostgreSQL ou MySQL. Não é possível desabilitá-lo.

**Você precisará das credenciais do Supabase** (ou outro PostgreSQL gratuito). Se ainda não tem, vá para Passo 2.1 abaixo.

```bash
# API Configuration
SERVER_URL=https://evolution-api-demo.onrender.com
AUTHENTICATION_API_KEY=MinhaChaveSecreta123456

# Database - OBRIGATÓRIO (use suas credenciais Supabase)
DATABASE_PROVIDER=postgresql
DATABASE_CONNECTION_URI=postgresql://postgres:SUA_SENHA_SUPABASE@db.SEU_PROJECT_REF.supabase.co:5432/postgres
DATABASE_CONNECTION_CLIENT_NAME=evolution_api

# Optional - Control what gets saved (default true)
DATABASE_SAVE_DATA_INSTANCE=true
DATABASE_SAVE_DATA_NEW_MESSAGE=true
DATABASE_SAVE_MESSAGE_UPDATE=true
DATABASE_SAVE_DATA_CONTACTS=true
DATABASE_SAVE_DATA_CHATS=true

# Storage
S3_ENABLED=false

# Webhook
WEBHOOK_GLOBAL_ENABLED=true
WEBHOOK_EVENTS_QRCODE_UPDATED=true
WEBHOOK_EVENTS_MESSAGES_UPSERT=true
WEBHOOK_EVENTS_SEND_MESSAGE=true

# Logs
LOG_LEVEL=ERROR
LOG_COLOR=false
```

⚠️ **IMPORTANTE:**
- Troque `MinhaChaveSecreta123456` por uma senha forte
- Substitua `SUA_SENHA_SUPABASE` pela senha do Supabase
- Substitua `SEU_PROJECT_REF` pelo project reference do Supabase

5. **Clique em "Create Web Service"**

---

### Passo 2.1: Obter Credenciais do Supabase (se ainda não tiver)

Se você já tem Supabase configurado para o projeto principal, use as mesmas credenciais. Caso contrário:

1. **Acesse:** https://supabase.com
2. **Crie conta grátis** (se não tiver)
3. **Crie um novo projeto** (ou use o existente)
4. **Obtenha as credenciais:**
   - Vá em **Settings** → **Database**
   - Copie a **Connection String** (formato URI)
   - Ou monte manualmente:
     - **Host:** Encontre em "Host" (ex: `db.abcdefghijk.supabase.co`)
     - **Password:** A senha que você definiu na criação do projeto
     - **Database:** `postgres` (padrão)
     - **Port:** `5432`

**Formato da URI:**
```
postgresql://postgres:SUA_SENHA@db.SEU_PROJECT_REF.supabase.co:5432/postgres
```

**Exemplo real:**
```
postgresql://postgres:MinhaSenhaForte123@db.xyzabc123.supabase.co:5432/postgres
```

5. **Cole essa URI** no campo `DATABASE_CONNECTION_URI` do Render

---

## ⏱️ Passo 3: Aguardar Deploy

1. Você verá os logs de build em tempo real
2. Primeiro deploy demora **5-10 minutos**
3. Quando aparecer **"Your service is live 🎉"**, está pronto!
4. Copie a URL: `https://evolution-api-demo.onrender.com`

---

## 🔧 Passo 4: Testar API

Teste se está funcionando:

```bash
curl https://evolution-api-demo.onrender.com/
```

Deve retornar algo como:
```json
{
  "status": "ok",
  "version": "2.x.x"
}
```

---

## 📱 Passo 5: Conectar WhatsApp

### 5.1. Criar Instância

```bash
curl -X POST https://evolution-api-demo.onrender.com/instance/create \
  -H "Content-Type: application/json" \
  -H "apikey: MinhaChaveSecreta123456" \
  -d '{
    "instanceName": "satisfaction-survey",
    "qrcode": true,
    "integration": "WHATSAPP-BAILEYS"
  }'
```

**Substitua:** `MinhaChaveSecreta123456` pela sua chave!

### 5.2. Obter QR Code

```bash
curl -X GET https://evolution-api-demo.onrender.com/instance/connect/satisfaction-survey \
  -H "apikey: MinhaChaveSecreta123456"
```

**Resposta:**
```json
{
  "base64": "data:image/png;base64,iVBORw0KG..."
}
```

### 5.3. Escanear QR Code

1. **Copie o conteúdo do campo `base64`**
2. **Cole em:** https://base64.guru/converter/decode/image
3. **O QR Code aparecerá**
4. **No WhatsApp do celular:**
   - Abra WhatsApp
   - Configurações → Aparelhos conectados
   - Conectar um aparelho
   - Escaneie o QR Code

✅ **WhatsApp conectado!**

### 5.4. Verificar Status

```bash
curl -X GET https://evolution-api-demo.onrender.com/instance/fetchInstances \
  -H "apikey: MinhaChaveSecreta123456"
```

Procure por:
```json
{
  "instance": {
    "instanceName": "satisfaction-survey",
    "state": "open"  // ← Conectado!
  }
}
```

---

## 🧪 Passo 6: Testar Envio de Mensagem

Envie uma mensagem de teste para você mesmo:

```bash
curl -X POST https://evolution-api-demo.onrender.com/message/sendText/satisfaction-survey \
  -H "Content-Type: application/json" \
  -H "apikey: MinhaChaveSecreta123456" \
  -d '{
    "number": "5511999999999",
    "text": "Teste da Evolution API! 🚀"
  }'
```

**Substitua:**
- `5511999999999` pelo seu número (DDI + DDD + número, sem espaços)
- `MinhaChaveSecreta123456` pela sua chave

✅ **Você deve receber a mensagem no WhatsApp!**

---

## 📋 Credenciais para Guardar

Anote em local seguro:

```
Evolution API URL: https://evolution-api-demo.onrender.com
API Key: MinhaChaveSecreta123456
Instance Name: satisfaction-survey
Status: Conectado ✅
```

---

## ⚠️ Limitações do Free Tier da Render

1. **Hibernação:** Após 15min de inatividade, o serviço hiberna
2. **Cold Start:** Primeira requisição após hibernar demora ~30s
3. **Solução:** Fazer um "wake up" antes de usar:
   ```bash
   curl https://evolution-api-demo.onrender.com/
   ```

4. **Alternativa:** Upgrade para plano pago ($7/mês) - sem hibernação

---

## 🔗 Próximos Passos

Agora que Evolution API está rodando:

1. ✅ Guardar URL e API Key
2. ✅ Testar envio de mensagem
3. 🔜 Configurar webhook para n8n (depois)
4. 🔜 Integrar com workflow n8n

---

## 🆘 Troubleshooting

### Problema: "Database provider sqlite invalid" ou "Database provider [X] invalid"
**Causa:** Evolution API NÃO suporta SQLite e NÃO permite desabilitar o banco de dados.

**Solução:**
1. Use PostgreSQL (Supabase free tier) ou MySQL
2. Configure corretamente as variáveis:
   ```bash
   DATABASE_PROVIDER=postgresql
   DATABASE_CONNECTION_URI=postgresql://postgres:senha@host:5432/postgres
   ```
3. Não use `DATABASE_ENABLED=false` (essa variável não existe)
4. Não use `DATABASE_PROVIDER=sqlite` (não é suportado)

**Valores válidos para DATABASE_PROVIDER:**
- `postgresql` ✅ (recomendado - use Supabase)
- `mysql` ✅
- `psql_bouncer` ✅

### Problema: Build falhou
**Solução:**
- Verifique se fez fork do repositório correto
- Certifique-se que Runtime está como "Docker"
- Verifique se configurou DATABASE_CONNECTION_URI corretamente

### Problema: QR Code expirou
**Solução:**
- Execute novamente o comando de `/instance/connect`
- QR Code expira em ~45 segundos

### Problema: Webhook não funciona
**Solução:**
- Vamos configurar isso depois, quando integrar com n8n

### Problema: WhatsApp desconectou
**Solução:**
```bash
# Verificar status
curl -X GET https://evolution-api-demo.onrender.com/instance/connectionState/satisfaction-survey \
  -H "apikey: SuaChave"

# Se desconectado, gerar novo QR
curl -X GET https://evolution-api-demo.onrender.com/instance/connect/satisfaction-survey \
  -H "apikey: SuaChave"
```

---

**Evolution API pronta para uso!** ✅
