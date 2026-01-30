# Evolution API - Configuração Correta para Render

**Data:** 2026-01-29
**Status:** Configuração validada contra `.env.example` oficial

---

## ❌ O QUE NÃO FUNCIONA (Erro comum)

```bash
# ERRADO - Evolution API NÃO suporta isso
DATABASE_ENABLED=false
DATABASE_PROVIDER=sqlite
```

**Erro gerado:**
```
Error: Database provider sqlite invalid.
==> Exited with status 1
```

---

## ✅ CONFIGURAÇÃO CORRETA

### Variáveis Obrigatórias para Render

```bash
# ==========================================
# SERVER CONFIGURATION
# ==========================================
SERVER_URL=https://evolution-api-demo.onrender.com
AUTHENTICATION_API_KEY=SuaChaveSecretaForte123

# ==========================================
# DATABASE (OBRIGATÓRIO - NÃO PODE SER DESABILITADO)
# ==========================================
DATABASE_PROVIDER=postgresql
DATABASE_CONNECTION_URI=postgresql://postgres:SUA_SENHA@db.SEU_PROJECT_REF.supabase.co:5432/postgres
DATABASE_CONNECTION_CLIENT_NAME=evolution_api

# ==========================================
# DATABASE PERSISTENCE (Opcional - default true)
# ==========================================
DATABASE_SAVE_DATA_INSTANCE=true
DATABASE_SAVE_DATA_NEW_MESSAGE=true
DATABASE_SAVE_MESSAGE_UPDATE=true
DATABASE_SAVE_DATA_CONTACTS=true
DATABASE_SAVE_DATA_CHATS=true
DATABASE_SAVE_DATA_LABELS=true
DATABASE_SAVE_DATA_HISTORIC=true

# ==========================================
# STORAGE (Desabilitar S3 para usar local)
# ==========================================
S3_ENABLED=false

# ==========================================
# WEBHOOK CONFIGURATION
# ==========================================
WEBHOOK_GLOBAL_ENABLED=true
WEBHOOK_EVENTS_QRCODE_UPDATED=true
WEBHOOK_EVENTS_MESSAGES_UPSERT=true
WEBHOOK_EVENTS_SEND_MESSAGE=true
WEBHOOK_EVENTS_STATUS_INSTANCE=true
WEBHOOK_EVENTS_CONNECTION_UPDATE=true

# ==========================================
# LOGS
# ==========================================
LOG_LEVEL=ERROR
LOG_COLOR=false
LOG_BAILEYS=error
```

---

## 🔑 Como Obter DATABASE_CONNECTION_URI

### Opção 1: Usar Supabase (Recomendado - Free Tier)

1. **Acesse:** https://supabase.com
2. **Login/Criar conta**
3. **Vá em Settings → Database**
4. **Copie a Connection String** ou monte manualmente:

**Formato:**
```
postgresql://postgres:[YOUR-PASSWORD]@db.[PROJECT-REF].supabase.co:5432/postgres
```

**Exemplo real:**
```
postgresql://postgres:MinhaSenha123@db.xyzabc456.supabase.co:5432/postgres
```

5. **Cole no Render** como valor de `DATABASE_CONNECTION_URI`

### Opção 2: Outros Provedores PostgreSQL Gratuitos

- **Neon.tech:** https://neon.tech (500MB grátis)
- **ElephantSQL:** https://elephantsql.com (20MB grátis)
- **Railway.app:** https://railway.app (limite de horas grátis)

---

## 📋 Passo a Passo no Render

1. **Abra seu Web Service** no Render
2. **Vá em Environment**
3. **Delete as variáveis antigas** (se tiver `DATABASE_ENABLED` ou `DATABASE_PROVIDER=sqlite`)
4. **Adicione as variáveis corretas** (copie do bloco acima)
5. **Substitua os valores:**
   - `SuaChaveSecretaForte123` → Sua API key escolhida
   - `SUA_SENHA` → Senha do PostgreSQL
   - `SEU_PROJECT_REF` → Project reference do Supabase
6. **Clique em "Save Changes"**
7. **Aguarde o redeploy automático**

---

## ✅ Validação

Após deploy bem-sucedido, teste:

```bash
curl https://evolution-api-demo.onrender.com/
```

**Resposta esperada:**
```json
{
  "status": 200,
  "message": "Welcome to the Evolution API",
  "version": "2.x.x"
}
```

---

## 🎯 Próximo Passo

Após Evolution API rodando:

1. ✅ Conectar WhatsApp via QR Code
2. ✅ Testar envio de mensagem
3. ✅ Configurar webhook para n8n
4. ✅ Integrar com workflow completo

---

## 📚 Referência

- **Repositório oficial:** https://github.com/EvolutionAPI/evolution-api
- **Arquivo de configuração:** https://github.com/EvolutionAPI/evolution-api/blob/main/.env.example
- **Documentação:** https://doc.evolution-api.com

---

**Última atualização:** 2026-01-29
**Verificado contra:** evolution-api `.env.example` oficial
