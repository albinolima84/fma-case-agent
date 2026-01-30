# Case Agent Dev - Sistema de Satisfação de Clientes com IA

Projeto desenvolvido para o desafio técnico FMA/Pareto/IA Leader - MBA em Inteligência Artificial.

## 📋 Sobre o Projeto

Sistema automatizado end-to-end para medição de satisfação de clientes utilizando agentes de IA, integrando dados do HubSpot CRM e canais de mensageria (WhatsApp).

**Objetivo:** Reduzir de 30 minutos para ~2 minutos o processo de análise e contato com cada cliente, garantindo personalização, consistência e rastreabilidade completa.

## 🏗️ Estrutura do Repositório

```
case-agent-dev/
├── README.md                    # Este arquivo
├── FMA.pdf                     # Documento do desafio
├── docs/                       # Documentação técnica
│   ├── 01-arquitetura-solucao.md
│   ├── 02-agentes-ia-detalhamento.md
│   ├── 03-processo-as-is-to-be.md
│   └── 04-plano-projeto-roi.md
├── diagrams/                   # Fluxogramas e diagramas
│   ├── as-is.mermaid
│   ├── to-be.mermaid
│   └── arquitetura.mermaid
├── prompts/                    # Prompts dos agentes de IA
│   ├── agent-1-data-fetcher.md
│   ├── agent-2-context-analyzer.md
│   ├── agent-3-message-generator.md
│   └── agent-4-conversation-handler.md
├── workflows/                  # Workflows n8n
│   ├── workflow-complete.json
│   ├── README.md
│   └── screenshots/
├── scripts/                    # Scripts auxiliares
│   └── setup/
└── interface/                  # Documentação Chatwoot
    └── chatwoot-config.md
```

## 🔧 Stack Tecnológica

- **Orquestração:** n8n (open-source)
- **IA/LLM:** Claude 3.5 Sonnet (Anthropic)
- **CRM:** HubSpot API v3
- **Mensageria:** Evolution API + WhatsApp Business
- **Interface:** Chatwoot (open-source)
- **Banco de Dados:** Supabase (PostgreSQL)

## 🤖 Agentes de IA

O sistema utiliza 4 agentes especializados:

1. **Data Fetcher:** Coleta histórico do cliente no HubSpot (30 dias)
2. **Context Analyzer:** Analisa dados e gera insights de satisfação
3. **Message Generator:** Cria mensagem personalizada de abertura
4. **Conversation Handler:** Conduz conversa bidirecional e extrai nota

## 📊 Documentação

### Principais Documentos

1. **[Arquitetura da Solução](docs/01-arquitetura-solucao.md)**
   - Stack técnica completa e justificativas
   - Fluxo de dados end-to-end
   - Infraestrutura e deploy
   - Estimativa de custos

2. **[Detalhamento dos Agentes](docs/02-agentes-ia-detalhamento.md)**
   - Responsabilidades de cada agente
   - Prompts completos
   - Inputs/Outputs esperados
   - Performance e custos

3. **[Processo AS-IS vs TO-BE](docs/03-processo-as-is-to-be.md)**
   - Fluxograma do processo atual (manual)
   - Fluxograma do processo otimizado (IA)
   - Análise comparativa

4. **[Plano de Projeto e ROI](docs/04-plano-projeto-roi.md)**
   - Fases de implementação
   - Atividades-chave e entregáveis
   - Estimativa de esforço
   - Cálculo de ROI

## 🎯 Métricas de Sucesso

### Processo Atual (Manual)
- ⏱️ Tempo: 30 min/cliente
- ❌ Erros: Copy/paste, informações inconsistentes
- 📊 Rastreabilidade: Baixa (planilhas)
- 👥 Escalabilidade: Limitada (1 gerente = ~16 clientes/dia)

### Processo Otimizado (IA)
- ⏱️ Tempo: ~2 min/cliente (93% redução)
- ✅ Erros: Mínimos (validação automática)
- 📊 Rastreabilidade: Completa (BD + Chatwoot)
- 👥 Escalabilidade: 200+ clientes/dia/gerente

### ROI Estimado
- **Economia de tempo:** 28 min/cliente × 200 clientes/mês = 93 horas/mês
- **Custo operacional:** ~$50-157/mês (infraestrutura + APIs)
- **Payback:** < 1 mês
- **ROI anual:** ~400-600%

## 🚀 Como Executar

Instruções detalhadas em [workflows/README.md](workflows/README.md)

### Pré-requisitos
- Docker e Docker Compose
- Conta HubSpot (API key)
- API key Anthropic (Claude)
- Conta Supabase (tier gratuito OK)
- Número WhatsApp Business (Evolution API)

### Setup Rápido
```bash
# 1. Clone o repositório
git clone <repo-url>
cd case-agent-dev

# 2. Configure variáveis de ambiente
cp .env.example .env
# Edite .env com suas API keys

# 3. Inicie os serviços
docker-compose up -d

# 4. Importe workflow no n8n
# Acesse http://localhost:5678
# Import: workflows/workflow-complete.json

# 5. Configure Evolution API
# Conecte número WhatsApp

# 6. Configure Chatwoot
# Acesse http://localhost:3000
# Crie inbox e integração
```

## 📱 Demo

- **Vídeo demonstrativo:** [Link para vídeo]
- **Workflow n8n:** [workflows/workflow-complete.json](workflows/workflow-complete.json)
- **Screenshots:** [workflows/screenshots/](workflows/screenshots/)
- **Exemplo de conversa:** [docs/exemplo-conversa.md](docs/exemplo-conversa.md)

## 📈 Próximos Passos (Pós-Piloto)

1. Adicionar canal de voz (Vapi.ai / Bland.ai)
2. Integração com Telegram e SMS
3. Dashboard analytics customizado
4. A/B testing de prompts
5. Fine-tuning de modelo próprio
6. Integração com NPS automático
7. Exportação automática para Data Warehouse

## 📄 Licença

Este projeto foi desenvolvido para fins educacionais como parte do MBA em Inteligência Artificial.

## 👥 Autor

Desenvolvido para o Case Agent Dev - FMA/Pareto/IA Leader

---

## 🔄 STATUS ATUAL DO PROJETO

**Última atualização:** 2026-01-29 23:30
**Progresso:** 92% completo
**Próximo passo:** Conectar WhatsApp à Evolution API

### ✅ Completo
- [x] Documentação completa (8 arquivos, ~150 páginas)
- [x] 3 Agentes IA testados e aprovados na Tess
- [x] Workflow n8n pronto (35 nodes)
- [x] **Evolution API deployada na Render** ✨ (hoje!)
- [x] PostgreSQL configurado (Supabase)

### ⏳ Pendente
- [ ] Conectar WhatsApp (próximo passo)
- [ ] Deploy n8n Cloud
- [ ] Testes end-to-end
- [ ] PDF final

---

## 🚀 PARA CONTINUAR AMANHÃ

Use este prompt:

```
Olá! Vamos continuar o projeto de onde paramos.

Ontem fizemos o deploy da Evolution API na Render com sucesso.
Agora preciso conectar o WhatsApp à API.

Por favor:
1. Me ajude a criar a instância "satisfaction-survey"
2. Gerar o QR Code para conectar o WhatsApp
3. Testar o envio de mensagem

Leia o arquivo CHECKPOINT-2026-01-29.md para ver todo o contexto.
```

---

## 📁 ARQUIVOS IMPORTANTES

### Status e Checkpoints
- **`STATUS-PROJETO.md`** - Status geral atualizado
- **`CHECKPOINT-2026-01-29.md`** - Checkpoint de hoje (MAIS RECENTE)
- **`CREDENCIAIS-E-CONFIGS.md`** - Todas as credenciais e configs

### Guias de Deploy
- **`DEPLOY-EVOLUTION-API-RENDER.md`** - Guia corrigido e validado
- **`EVOLUTION-API-ENV-CORRETO.md`** - Configurações corretas

### Documentação
- **`/docs/`** - Toda a documentação técnica
- **`/prompts/`** - Prompts ajustados dos agentes
- **`/workflows/`** - Workflow n8n pronto

---

## 🔑 ACESSO RÁPIDO

**Evolution API:** https://evolution-api-demo.onrender.com
**Status:** ✅ Online e funcionando

---

**Última atualização:** 2026-01-29 23:30
**Versão:** 1.2
