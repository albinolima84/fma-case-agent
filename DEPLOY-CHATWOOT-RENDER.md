# Deploy Chatwoot no Render.com (Free Tier)

**Data de decisão:** 2026-02-02
**Motivo:** Projeto piloto (1 mês), custo zero adequado para validação

---

## 📋 Decisão de Hospedagem

### Por que Render Free Tier?

✅ **Vantagens:**
- Custo: $0 para projeto piloto
- Deploy rápido via GitHub
- Já possui conta Render ativa
- PostgreSQL e Redis incluídos no free tier

⚠️ **Limitação Conhecida:**
- **Sleep após 15min de inatividade**
- **Warm-up:** 30-60 segundos na primeira abertura
- **Aceitável porque:**
  - Projeto piloto (1 mês)
  - Gerentes acessam poucas vezes por dia
  - Interface administrativa (não crítico)
  - **Dados nunca são perdidos** (salvos no Supabase)

### Arquitetura de Dados (Importante!)

```
┌─────────────────────────────────────────────────────────┐
│  Cliente WhatsApp → Meta API → n8n Cloud               │
│                                    ↓                     │
│                              Supabase (PRIMÁRIO)        │
│                              - 100% dos dados           │
│                              - Sempre ativo             │
│                              - Fonte de verdade         │
│                                    ↓                     │
│                              Chatwoot (SECUNDÁRIO)      │
│                              - Interface visualização   │
│                              - Pode ter sleep           │
│                              - Sincronização best effort│
└─────────────────────────────────────────────────────────┘
```

**Garantia:** Mesmo se Chatwoot estiver offline, nenhum dado é perdido (Supabase salva tudo).

---

## 🚀 Guia Rápido de Deploy

### Passo-a-Passo Resumido (5 minutos)

1. **Render Dashboard** → New → PostgreSQL
   - Nome: `chatwoot-postgres`
   - Database: `chatwoot`
   - **Copiar Internal Database URL**

2. **Render Dashboard** → New → Redis
   - Nome: `chatwoot-redis`
   - **Copiar Internal Redis URL**

3. **Render Dashboard** → New → Web Service → **Existing Image**
   - Image: `chatwoot/chatwoot:latest`
   - Nome: `chatwoot-web`
   - Tier: Free

4. **Environment Variables** (no Web Service)
   - Colar URLs do PostgreSQL e Redis
   - Adicionar SECRET_KEY_BASE (gerar novo)
   - Adicionar FRONTEND_URL (URL do Render)

5. **Deploy** → Aguardar ~5-10 min

6. **Acessar URL** → Criar conta admin

---

## 📝 Passos Detalhados

### Pré-requisitos
- [x] Conta Render.com ativa
- [ ] Variáveis de ambiente preparadas

### Etapa 1: Criar Serviços no Render

**Importante: Ordem de criação**
1. PostgreSQL (primeiro)
2. Redis (segundo)
3. Web Service - Chatwoot (terceiro - precisa dos anteriores)

**1.1 PostgreSQL:**
- Nome: `chatwoot-postgres`
- Tier: Free
- Versão: 15

**1.2 Redis:**
- Nome: `chatwoot-redis`
- Tier: Free

**1.3 Web Service (Chatwoot):**

**Opção de Deploy (escolha uma):**

Render oferece 3 opções ao criar Web Service:

---

**OPÇÃO A: Existing Image** ⭐ **RECOMENDADO PARA PILOTO**

✅ **Mais rápido** (imagem já compilada)
✅ **Sem necessidade de fork** no GitHub
✅ **Oficialmente mantida** pela equipe Chatwoot
✅ **Ideal para piloto** (deploy em ~5 min)

**Como configurar:**
```
Render Dashboard > New > Web Service > Existing Image

Image URL: chatwoot/chatwoot:latest
Registry: Docker Hub (público)

Nome: chatwoot-web
Region: Oregon (US West) ou Frankfurt (EU)
Instance Type: Free (512MB RAM)
```

---

**OPÇÃO B: Public Git Repository**

✅ **Sem necessidade de fork**
⚠️ Mais lento (precisa buildar ~15-20 min)

**Como configurar:**
```
Render Dashboard > New > Web Service > Public Git Repository

Repository URL: https://github.com/chatwoot/chatwoot
Branch: main

Build Command: (deixar vazio - usa Dockerfile)
Start Command: (deixar vazio - usa Dockerfile)
```

---

**OPÇÃO C: Git Provider (GitHub/GitLab)**

✅ **Permite customização** futura
⚠️ Requer fork do repositório
⚠️ Mais trabalho inicial

**Quando usar:** Se planeja customizar Chatwoot no futuro

---

**Configurações comuns (qualquer opção):**
- Nome: `chatwoot-web`
- Runtime: Docker
- Tier: **Free** (512MB RAM - suficiente para piloto)
- Auto-Deploy: Yes

### Etapa 2: Conectar Banco de Dados e Redis

**Importante:** Após criar PostgreSQL e Redis, você receberá as URLs de conexão.

**PostgreSQL:**
- Internal Database URL: `postgresql://user:pass@host:5432/chatwoot`
- Copiar e guardar (usar nas variáveis de ambiente)

**Redis:**
- Internal Redis URL: `redis://host:6379`
- Copiar e guardar (usar nas variáveis de ambiente)

---

### Etapa 3: Variáveis de Ambiente

```bash
# Database (Render fornece automaticamente)
POSTGRES_HOST=<render-postgres-hostname>
POSTGRES_DATABASE=chatwoot
POSTGRES_USERNAME=chatwoot
POSTGRES_PASSWORD=<gerado-pelo-render>

# Redis (Render fornece automaticamente)
REDIS_URL=<render-redis-url>

# Chatwoot Config
SECRET_KEY_BASE=<gerar: openssl rand -hex 64>
FRONTEND_URL=https://chatwoot-web.onrender.com
INSTALLATION_NAME=Pareto

# Email (opcional para piloto)
MAILER_SENDER_EMAIL=noreply@seudominio.com

# Timezone
TZ=America/Sao_Paulo
```

### Etapa 4: Deploy

**Se escolheu Opção A (Existing Image):**
1. Configurar variáveis de ambiente (ver Etapa 3)
2. Clicar "Create Web Service"
3. Aguardar ~5-10 minutos (download da imagem + start)
4. Chatwoot estará disponível na URL fornecida pelo Render

**Se escolheu Opção B ou C (Git):**
1. Conectar repositório (público ou privado)
2. Configurar variáveis de ambiente (ver Etapa 3)
3. Clicar "Create Web Service"
4. Aguardar ~15-20 minutos (build + deploy)

---

### Etapa 5: Setup Inicial

1. Acessar `https://chatwoot-web.onrender.com`
2. Criar conta de administrador
3. Criar inbox "Satisfação WhatsApp"
4. Gerar API Access Token
5. Salvar token no n8n

---

## 🔧 Integração n8n ↔ Chatwoot

### Nodes a criar no workflow n8n:

**FLUXO 1 - Envio Proativo:**
1. **Create Chatwoot Contact** (após Agent 3)
2. **Create Chatwoot Conversation** (após enviar WhatsApp)
3. **Send Message to Chatwoot** (registrar mensagem inicial)

**FLUXO 2 - Respostas:**
1. **Send Message to Chatwoot** (cada mensagem enviada/recebida)
2. **Update Chatwoot Conversation** (ao finalizar: status "resolved", nota privada com score)

### Configuração de erro handling:

```javascript
// Supabase sempre tem prioridade
try {
  // CRÍTICO: Salvar no Supabase
  await supabase.insert(survey_data);
} catch (error) {
  throw error; // Falha se Supabase falhar
}

// BEST EFFORT: Tentar Chatwoot
try {
  await chatwoot.createMessage(data);
} catch (error) {
  console.log('Chatwoot indisponível (possivelmente em sleep), dados salvos no Supabase');
  // Não falha o workflow
}
```

---

## ⚠️ Limitações do Free Tier

### Sleep/Warm-up
- **Quando:** Após 15 minutos de inatividade
- **Tempo de wake-up:** 30-60 segundos
- **Impacto:**
  - ✅ **Zero** na captura de dados (Supabase)
  - ⚠️ Delay ao abrir interface (gerentes)
  - ⚠️ Possível falha ao enviar mensagens via n8n (não crítico)

### Uptime
- **Limite:** 750 horas/mês grátis
- **Para piloto 1 mês:** Suficiente (720h = 30 dias)
- **Se exceder:** Considerar upgrade ($7/mês) ou migrar para Chatwoot Cloud ($19/mês)

### Performance
- **512MB RAM:** Adequado para 1-2 gerentes, <50 conversas ativas
- **Latência:** ~200-500ms (aceitável para interface admin)

---

## 📊 Monitoramento

### Como saber se está funcionando:

1. **Supabase:** Verificar tabela `surveys` - sempre deve ter todos os dados
2. **Chatwoot:** Verificar inbox - pode ter delay se estava em sleep
3. **n8n Logs:** Verificar se há erros ao chamar Chatwoot API (não crítico)

### Métricas a observar:

- Taxa de sucesso ao enviar para Chatwoot (meta: >80%)
- Tempo de warm-up (informar gerentes)
- Satisfação dos gerentes com delay (se incomodar, considerar upgrade)

---

## 🔄 Plano de Migração (Se Necessário)

### Cenário 1: Sleep incomoda muito
**Solução:** Adicionar UptimeRobot (ping a cada 5min)
**Custo:** $0
**Tempo:** 5 minutos

### Cenário 2: Performance insuficiente
**Solução:** Render Starter ($7/mês - 512MB sem sleep)
**Quando:** Se >5 gerentes ou >100 conversas/dia

### Cenário 3: Produção após piloto
**Solução:** Chatwoot Cloud ($19/mês) ou VPS Hetzner ($5/mês)
**Quando:** Aprovar piloto, escalar para produção

---

## 📝 Documentação de Referência

- **Chatwoot Docs:** https://www.chatwoot.com/docs/self-hosted/deployment/render
- **Render Docs:** https://render.com/docs
- **Arquitetura completa:** `docs/01-arquitetura-solucao.md` seção 5

---

**Status:** Documentado, pronto para implementação
**Próximo passo:** Criar serviços no Render e fazer deploy
**Estimativa:** 30-45 minutos de setup inicial
