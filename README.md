# Sistema de Pesquisa de Satisfação com IA via WhatsApp

**Projeto Final — MBA em Inteligência Artificial | FMA**

Sistema automatizado end-to-end para medição de satisfação de clientes utilizando 4 agentes de IA especializados, integrando HubSpot CRM, Meta WhatsApp Business API oficial, e orquestração via n8n Cloud.

**Objetivo:** Reduzir de 30 minutos para ~2 minutos o processo de pesquisa por cliente, garantindo personalização, consistência e rastreabilidade completa.

---

## 🎥 Demonstração

**[▶ Assistir Demo no Loom (~9 minutos)](https://www.loom.com/share/2656b06ebba644c98097d4aa0e125ff9)**

Demonstração completa do sistema funcionando end-to-end: execução do workflow, envio de mensagem personalizada, conversa via WhatsApp, monitoramento no Chatwoot e dados registrados no banco.

---

## 📄 Documento Principal

**[ENTREGA-FINAL.md](ENTREGA-FINAL.md)** — Documento master com arquitetura, implementação, evidências, ROI e guia de reprodução (865 linhas).

---

## 📊 Resultados Validados

**5 conversas reais coletadas e analisadas:**

| Métrica | Resultado |
|---------|-----------|
| Taxa de conclusão | **100%** (5/5 surveys) |
| Satisfação média | **3.60/5** |
| NPS | **60%** (4 promotores, 1 detrator) |
| Precisão na extração | **100%** (score + sentiment) |
| Redução de tempo | **93%** (30 min → 2 min) |
| Redução de custo | **61%** (R$ 12.100 → R$ 4.660/mês) |
| ROI Ano 1 | **62%** |
| ROI 3 Anos | **487%** |
| Payback | **8 meses** |

---

## 🏗️ Arquitetura

```
┌─────────────────────────────────────────────────────────┐
│                  FLUXO 1: ENVIO PROATIVO                 │
│  Schedule → Config → HubSpot Search → Loop:             │
│    Elegibilidade? → Agent 1 (Data) →                    │
│    Agent 2 (Contexto) → Agent 3 (Mensagem) →            │
│    Meta WhatsApp → Supabase → Chatwoot                  │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                FLUXO 2: RESPOSTAS (WEBHOOK)              │
│  Meta Webhook → Supabase (busca) → Chatwoot (incoming)  │
│    → Agent 4 V2.0 → Meta WhatsApp → Chatwoot (outgoing) │
│    → IF completed: Nota Privada + Resolver + Supabase   │
└─────────────────────────────────────────────────────────┘
```

---

## 🤖 Agentes de IA (Tess AI / gpt-4o-mini)

| Agente | ID | Função |
|--------|-----|--------|
| Agent 1 | — | Data Fetcher — consolida dados do HubSpot (inline n8n) |
| Agent 2 | [38717](https://tess.im/pt-BR/dashboard/user/ai/chat/ai-chat/context-analyzer-6TBb4l) | Context Analyzer — analisa histórico e gera insights |
| Agent 3 | [38728](https://tess.im/pt-BR/dashboard/user/ai/chat/ai-chat/message-generator-vE6X3l) | Message Generator — cria mensagem personalizada |
| Agent 4 | [38733](https://tess.im/pt-BR/dashboard/user/ai/chat/ai-chat/conversation-handler-GjH0pE) | Conversation Handler V2.0 — conduz conversa e extrai dados |

**Agent 4 V2.0:** 9/9 cenários de teste validados, finalização inteligente em 2-3 turnos.

---

## 🔧 Stack Tecnológica

| Camada | Tecnologia | Função |
|--------|------------|--------|
| Orquestração | n8n Cloud | Workflow automation |
| IA/LLM | Tess AI (gpt-4o-mini) | 3 agentes especializados |
| Mensageria | Meta WhatsApp Business API v21.0 | API oficial |
| CRM | HubSpot API v3 | Fonte de dados de clientes |
| Banco de Dados | Supabase PostgreSQL | Histórico JSONB |
| Monitoramento | Chatwoot Cloud | Interface em tempo real (requisito FMA) |

**Custo operacional:** R$ 4.660/mês (200 pesquisas/mês)

---

## 📁 Estrutura do Repositório

```
fma-case-agent/
├── ENTREGA-FINAL.md                        ← Documento master de entrega
├── README.md                               ← Este arquivo
├── docs/
│   ├── 01-arquitetura-solucao.md           ← Stack e arquitetura técnica
│   ├── 02-agentes-ia-detalhamento.md       ← Detalhes dos 4 agentes
│   ├── 03-processo-as-is-to-be.md          ← Processo manual vs automatizado
│   ├── 04-plano-projeto-roi.md             ← ROI detalhado e cenários
│   ├── 05-guia-deploy-online.md            ← Guia passo a passo de deploy
│   ├── 06-guia-entrega.md                  ← Checklist de entrega
│   ├── exemplos-conversas-completas.md     ← 5 transcrições reais
│   └── metricas-validacao.md              ← Análise de performance
├── workflows/
│   └── satisfaction-survey-workflow.json  ← Workflow n8n v2.0 (Config nodes)
├── prompts/
│   ├── agent-2-context-analyzer-AJUSTADO.txt
│   ├── agent-3-message-generator-AJUSTADO.txt
│   └── agent-4-conversation-handler-V2.txt
├── diagrams/
│   ├── as-is-processo-manual.png           ← Processo manual
│   └── to-be-processo-automatizado.png     ← Processo automatizado com loop
└── screenshots/                            ← 11 evidências visuais do sistema
```

---

## 🚀 Como Reproduzir

**Pré-requisitos:**
- Conta n8n Cloud (trial 14 dias gratuito)
- Conta Supabase (free tier)
- HubSpot com API key
- Tess AI com créditos
- Meta WhatsApp Business API (número aprovado)
- Chatwoot Cloud (plano Hacker gratuito)

**Deploy em 6 passos:**
1. Criar tabela `surveys` no Supabase (SQL em [docs/05-guia-deploy-online.md](docs/05-guia-deploy-online.md))
2. Importar [workflow](workflows/satisfaction-survey-workflow.json) no n8n Cloud
3. Configurar credenciais (Meta, Tess, HubSpot, Supabase, Chatwoot)
4. Ajustar Config nodes com seus IDs
5. Configurar webhook na Meta Developer Console
6. Ativar workflow e testar

**Guia completo:** [docs/05-guia-deploy-online.md](docs/05-guia-deploy-online.md)

---

## 📈 ROI

| Métrica | Valor |
|---------|-------|
| Investimento Inicial | R$ 55.000 |
| Custo Mensal TO-BE | R$ 4.660 |
| Custo Mensal AS-IS | R$ 12.100 |
| Economia Anual | R$ 89.280 |
| **ROI Ano 1** | **62%** |
| **ROI 3 Anos** | **487%** |
| **Payback** | **8 meses** |

---

---

**MBA em Inteligência Artificial | FMA | Fevereiro 2026**
**Status:** ✅ 100% completo — Aprovado para produção imediata
