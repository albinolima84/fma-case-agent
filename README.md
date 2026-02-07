# Sistema de Pesquisa de Satisfação com IA via WhatsApp

Sistema automatizado end-to-end para medição de satisfação de clientes utilizando agentes de IA, integrando dados do HubSpot CRM e WhatsApp Business via Meta API oficial.

**Objetivo:** Reduzir de 30 minutos para ~2 minutos o processo de análise e contato com cada cliente, garantindo personalização, consistência e rastreabilidade completa.

---

## 🏗️ Estrutura do Repositório

```
case-agent-dev/
├── README.md                              # Este arquivo
├── PROMPT-RETOMADA.md                     # Para retomar desenvolvimento
├── docs/                                  # Documentação estruturada
│   ├── 01-arquitetura-solucao.md
│   ├── 02-agentes-ia-detalhamento.md
│   ├── 03-processo-as-is-to-be.md
│   ├── 04-plano-projeto-roi.md
│   ├── 05-guia-deploy-online.md
│   └── 06-guia-entrega.md
├── workflows/
│   ├── satisfaction-survey-workflow.json   # Workflow n8n exportado
│   └── README.md
├── prompts/                               # Prompts dos agentes
│   ├── agent-2-context-analyzer.txt
│   ├── agent-3-message-generator.txt
│   └── agent-4-conversation-handler.txt
└── diagrams/                              # Diagramas do processo
    ├── as-is-processo-manual.mermaid + .png
    └── to-be-processo-automatizado.mermaid + .png
```

---

## 🔧 Stack Tecnológica

- **Orquestração:** n8n Cloud (workflow automation)
- **IA/LLM:** Tess AI (gpt-4o-mini)
- **CRM:** HubSpot API v3
- **Mensageria:** **Meta WhatsApp API (Oficial)** 🆕
- **Banco de Dados:** Supabase PostgreSQL (Cloud)
- **Interface:** Chatwoot Cloud (Hacker gratuito no piloto; Pro R$105/mês em produção)

---

## 🤖 Agentes de IA (Tess AI)

O sistema utiliza 4 agentes especializados:

| Agente | ID | Função | Status |
|--------|-----|--------|--------|
| **Agent 1** | - | Data Fetcher (inline n8n) | ✅ Implementado |
| **Agent 2** | 38717 | Context Analyzer | ✅ Funcionando |
| **Agent 3** | 38728 | Message Generator | ✅ Funcionando |
| **Agent 4** | 38733 | Conversation Handler V2.0 | ✅ Otimizado |

### 🔗 Links Públicos dos Agentes

- **[Agent 2 - Context Analyzer](https://tess.im/pt-BR/dashboard/user/ai/chat/ai-chat/context-analyzer-6TBb4l)** (ID: 38717)
- **[Agent 3 - Message Generator](https://tess.im/pt-BR/dashboard/user/ai/chat/ai-chat/message-generator-vE6X3l)** (ID: 38728)
- **[Agent 4 - Conversation Handler V2](https://tess.im/pt-BR/dashboard/user/ai/chat/ai-chat/conversation-handler-GjH0pE)** (ID: 38733)

> 💡 **Nota:** Os prompts completos de cada agente estão disponíveis no diretório [`prompts/`](prompts/).

### Agent 4 V2.0 - Melhorias Implementadas 🆕

O Agent 4 foi completamente otimizado para garantir finalização correta de conversas:

**Problema anterior:** Entrava em loop, perguntando a nota repetidamente
**Solução V2.0:**
- ✅ 5 regras claras de finalização
- ✅ Checklist mental para decidir status
- ✅ Finaliza após coletar nota + feedback (2-3 turnos ideais)
- ✅ **9 de 9 cenários de teste validados com sucesso**
- ✅ Fluxo otimizado: antes indefinido, agora 2-3 interações

**Condições de finalização:**
1. Nota + Feedback coletados ✅
2. Cliente sinalizou encerramento
3. Turno 3+ com nota coletada
4. Turno 5 atingido (limite absoluto)
5. Cliente não quer participar

### Fluxo de Processamento

**FLUXO 1 - Envio Proativo (100% FUNCIONAL):** ✅
```
HubSpot CRM → Agent 1 (Data) → Agent 2 (Análise) →
Agent 3 (Mensagem) → Meta WhatsApp → Supabase →
Search/Create Contact (Chatwoot) → Create Conversation → Send Message
```

**FLUXO 2 - Respostas (100% FUNCIONAL):** ✅
```
WhatsApp → Meta Webhook → n8n → Supabase (busca survey) →
Send User Message (Chatwoot) → Agent 4 V2.0 (Conversa) →
Meta API (resposta) → Send Bot Response (Chatwoot) →
IF completed: Add Private Note + Mark Resolved (Chatwoot) →
Supabase (update)
```

---

## 📊 Status Atual

**Última atualização:** 2026-02-03
**Progresso:** 100% completo ████████████████████████

### ✅ Completo e Funcionando

- ✅ **Meta WhatsApp API** configurada e verificada
- ✅ **FLUXO 1** (Envio Proativo) 100% operacional com integração Chatwoot
- ✅ **FLUXO 2** (Recebimento/Resposta) 100% operacional com integração Chatwoot
- ✅ **Webhook** Meta → n8n funcionando
- ✅ **Agent 4 V2.0** otimizado com finalização inteligente
- ✅ **Supabase** salvando conversações em JSONB com histórico completo
- ✅ **Chatwoot Cloud** — interface de monitoramento integrada (requisito FMA)
  - Supervisão em tempo real de conversas
  - Histórico completo (mensagens do cliente e do bot)
  - Nota privada automática com score/sentiment/feedback quando survey é concluído
  - Conversa marcada como resolvida automaticamente
  - Intervenção manual disponível pelo gerente
- ✅ **n8n Cloud** workflow ativo 24/7
- ✅ **Testes end-to-end** completos e validados (com Chatwoot)

---

## 🎯 Métricas e Resultados

### Processo Atual (Manual)
- ⏱️ Tempo: 30 min/cliente
- ❌ Erros: Copy/paste, inconsistências
- 📊 Rastreabilidade: Baixa (planilhas)
- 👥 Escalabilidade: ~16 clientes/dia

### Processo Otimizado (IA)
- ⏱️ Tempo: ~2 min/cliente (93% redução)
- ✅ Erros: Mínimos (<1%)
- 📊 Rastreabilidade: Completa (PostgreSQL + histórico)
- 👥 Escalabilidade: 200+ clientes/dia

### ROI Estimado
- **Economia de tempo:** 28 min/cliente × 200 clientes/mês = 93 horas/mês
- **Economia mensal:** R$7.590 (63%) | Anual R$91.080
- **Custo operacional (produção):** R$300/mês (OPEX Total)
- **Payback:** 8 meses (conservador)
- **ROI:** Ano 1: 66% | Ano 2: 166% | 3 anos: 497%

---

## 🚀 Como Funciona

### 1. Configuração Inicial (One-time)

**Pré-requisitos:**
- Conta n8n Cloud (free tier)
- Conta Supabase (free tier)
- HubSpot API key
- Tess AI account
- **Meta WhatsApp Business API** (número aprovado)

### 2. Deploy

Siga o guia completo: **[05-guia-deploy-online.md](docs/05-guia-deploy-online.md)**

**Resumo:**
1. Importar workflow no n8n Cloud
2. Configurar credenciais (Meta, Tess, HubSpot, Supabase)
3. Configurar webhook na Meta Developer Console
4. Ativar workflow
5. Testar!

### 3. Uso Diário

**Envio de Pesquisas:**
- Manual: Executar workflow no n8n
- Automático: Schedule trigger (diário às 10h)

**Recebimento de Respostas:**
- Automático via webhook Meta → n8n
- Agent 4 processa e responde em tempo real
- Histórico salvo no Supabase

---

## 📱 Configuração Meta WhatsApp API

### Credenciais Necessárias

```
Phone Number ID: 674094992450703
WhatsApp Business Account ID: 1255054259608433
Número do Bot: +55 11 5286-8259
API Version: v21.0
```

### Webhook Configurado

```
URL: https://albino.app.n8n.cloud/webhook/whatsapp-meta
Verify Token: satisfaction-survey-2026
Events: messages
Status: ✅ Verificado e ativo
```

---

## 💾 Banco de Dados (Supabase)

### Tabela `surveys`

```sql
CREATE TABLE surveys (
  id SERIAL PRIMARY KEY,
  customer_phone VARCHAR(20) NOT NULL,
  customer_name VARCHAR(255),
  context_summary TEXT,
  conversation_transcript JSONB DEFAULT '[]',
  satisfaction_score INTEGER,
  main_feedback TEXT,
  sentiment VARCHAR(20),
  status VARCHAR(20) DEFAULT 'active',
  chatwoot_conversation_id INTEGER,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

**Formato do histórico (JSONB):**
```json
[
  {"role": "user", "content": "mensagem do cliente"},
  {"role": "assistant", "content": "resposta do bot"}
]
```

---

## 🧪 Teste End-to-End Completo - APROVADO ✅

**Cliente:** Carlos Mendes (+55 21 98144-4992)
**Bot:** +55 11 5286-8259
**Data:** 2026-02-02 15:47

**Conversação Completa (5 mensagens):**
1. **Bot (FLUXO 1):** "Olá Carlos! Notei que você teve uma interação recente com nosso suporte sobre configuração de API..."
2. **Cliente:** "Oi! Sim, foi resolvido rapidamente"
3. **Bot:** "Que ótimo ouvir isso! De 1 a 5, como avalia o atendimento?"
4. **Cliente:** "4 - Gostei do tempo de resposta do suporte"
5. **Bot:** "Muito obrigado pelo feedback, Carlos! Estamos sempre à disposição."

**Resultado Final:**
- ✅ **FLUXO 1:** Mensagem inicial enviada com sucesso (texto livre, sem templates)
- ✅ **FLUXO 2:** Conversa completa processada (5 mensagens)
- ✅ **Nota coletada:** 4 (Muito bom)
- ✅ **Feedback:** "Gostei do tempo de resposta do suporte"
- ✅ **Sentiment:** positive
- ✅ **Status:** completed (finalizado corretamente)
- ✅ **Histórico completo:** 5 mensagens preservadas no banco (JSONB)
- ✅ **Agent 4 V2.0:** Finalizou conversa após coletar nota + feedback
- ✅ **Chatwoot:** Mensagens do cliente e do bot registradas na conversa
- ✅ **Chatwoot:** Nota privada com resumo do survey adicionada
- ✅ **Chatwoot:** Conversa marcada como resolvida

---

## 📖 Documentação Completa

### Documentação Estruturada
- **[01 - Arquitetura da Solução](docs/01-arquitetura-solucao.md)**
- **[02 - Agentes IA (Detalhamento)](docs/02-agentes-ia-detalhamento.md)**
- **[03 - Processo AS-IS / TO-BE](docs/03-processo-as-is-to-be.md)**
- **[04 - Plano do Projeto e ROI](docs/04-plano-projeto-roi.md)**
- **[05 - Guia de Deploy Online](docs/05-guia-deploy-online.md)**
- **[06 - Guia de Entrega](docs/06-guia-entrega.md)**

### Workflows e Prompts
- **[workflows/satisfaction-survey-workflow.json](workflows/satisfaction-survey-workflow.json)** - Workflow n8n exportado
- **[prompts/](prompts/)** - Prompts dos agentes (Agent 2, 3, 4)

### Sessão e Contexto
- **[PROMPT-RETOMADA.md](PROMPT-RETOMADA.md)** - Para retomar desenvolvimento
- **[CHECKPOINT-2026-02-04.md](CHECKPOINT-2026-02-04.md)** - Checkpoint mais recente

---

## 🔧 Troubleshooting

### Webhook não dispara
**Verificar:**
- Workflow ativo no n8n ✅
- Subscription "messages" na Meta ✅
- Survey ativo no banco ✅

### Erro no Supabase
**Verificar:**
- Sintaxe SQL correta
- Interpolação de variáveis
- Formato JSONB válido

### Agent não responde
**Verificar:**
- Credencial Tess configurada no n8n
- Formato correto: temperature "1", messages array
- API key válida

### Tess retorna erro de parse
**Verificar:**
- Output do Tess pode vir com ou sem marcações ````json``` ` — não-determinístico
- Sempre fazer `.replace(/^```json\n?/, '').replace(/\n?```$/, '').trim()` antes de `JSON.parse()`

### Erro SQL "Syntax error" com aspas
**Verificar:**
- Valores do usuário interpolados em queries precisam de escape: `.replace(/'/g, "''")`
- Campos afetados: `main_feedback`, `conversation_transcript`

### Merge após IF trava
**Verificar:**
- Modo "Combine" espera todos os inputs — trava quando só um branch executa
- Usar modo **Append** para IFs onde apenas um branch fica ativo

---

## 🎓 Lições Aprendidas

### 1. Meta API >> Outras Soluções
- Muito mais estável que alternativas
- Webhooks 100% confiáveis
- Infraestrutura oficial da Meta
- Vale o esforço de configuração inicial

### 2. n8n Modern Syntax
- Usar `$('Node Name').first().json`
- Code nodes úteis para transformações complexas
- responseMode correto para webhooks

### 3. Supabase JSONB
- Formato: `'JSON_STRING'::jsonb`
- JSON.stringify() para converter objetos
- Validar interpolação em SQL

### 4. Array Reference vs Copy em JavaScript
- **Problema crítico:** `history = survey.conversation_transcript` cria referência, não cópia
- **Solução:** Usar spread operator `history = [...survey.conversation_transcript]`
- **Impacto:** Histórico sendo sobrescrito causava loops infinitos
- **Lição:** Sempre fazer cópia de arrays que serão modificados

### 5. Agent Prompt Engineering
- Prompts claros e específicos são essenciais para finalização correta
- Checklist mental ajuda o modelo a tomar decisões consistentes
- Validar com múltiplos cenários de teste (9+ casos)
- Agent 4 V2.0: Problema de loop resolvido via otimização de prompt

### 6. LLM Output é Não-Determinístico
- Tess (gpt-4o-mini) às vezes retorna JSON puro, às vezes envolve em ````json``` `
- Sempre sanitizar output antes de parsear — não assumir formato fixo

### 7. SQL com Dados do Usuário
- Valores interpolados diretamente em queries precisam de escape de aspas simples
- `.replace(/'/g, "''")` é essencial para campos como feedback e transcrição

### 8. n8n Merge após IF
- Modo "Combine" trava quando apenas um branch do IF executa
- Usar **Append** — processa cada input independentemente sem esperar todos

---

## 📈 Próximos Passos

### ✅ Fase 1 - Backend (Completo)
1. ✅ Completar FLUXO 1 (envio proativo com texto livre)
2. ✅ Completar FLUXO 2 (recebimento e respostas)
3. ✅ Otimizar Agent 4 V2.0 (finalização inteligente)
4. ✅ Testes end-to-end completos e validados
5. ✅ Documentação completa e atualizada

### ✅ Fase 2 - Interface de Monitoramento (Completo)
**Solução:** Chatwoot Cloud (Hacker gratuito no piloto; Pro R$105/mês em produção)

**Implementado:**
- ✅ Conta Chatwoot Cloud configurada
- ✅ Inbox "Satisfação WhatsApp" criado
- ✅ FLUXO 1: Search/Create Contact → Create Conversation → Save ID → Send Message
- ✅ FLUXO 2: Send User Message → Send Bot Response → Add Private Note → Mark Resolved
- ✅ Teste end-to-end validado com todas as mensagens registradas

### Fase 3 - Melhorias Futuras (Opcional)
1. Dashboard analytics (Supabase + Grafana)
2. A/B testing de prompts
3. Integração com outros canais (Telegram, Email)
4. Exportação automática de métricas
5. NPS automático calculado

### Longo Prazo
1. Fine-tuning de modelo próprio
2. Integração com canal de voz
3. Multi-idioma
4. Escalabilidade para milhares de clientes

---

## 📄 Licença

Este projeto foi desenvolvido para fins educacionais como parte de um case técnico.

---

## 🔗 Links Úteis

- **n8n Cloud:** https://albino.app.n8n.cloud
- **Meta Developer Console:** https://developers.facebook.com/apps
- **Supabase Dashboard:** https://supabase.com/dashboard
- **Tess AI:** https://tess.im

---

**Última atualização:** 2026-02-05
**Versão:** 3.1 (Meta WhatsApp API + Chatwoot Cloud)
**Status:** 100% completo
