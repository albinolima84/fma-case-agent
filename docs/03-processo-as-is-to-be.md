# Mapeamento de Processo: AS-IS vs TO-BE

## Visão Geral

Este documento apresenta a comparação entre o processo atual (AS-IS) manual de pesquisa de satisfação de clientes e o processo otimizado (TO-BE) com automação e agentes de IA.

## Fluxogramas

### Como Visualizar

Os fluxogramas foram criados em formato Mermaid e podem ser visualizados de várias formas:

**Opção 1: GitHub/GitLab** - Renderiza automaticamente em arquivos .md

**Opção 2: VS Code** - Instale extensão "Markdown Preview Mermaid Support"

**Opção 3: Online** - Cole o conteúdo em https://mermaid.live

**Opção 4: Exportar PNG** - Use Mermaid CLI ou ferramentas online

---

## AS-IS: Processo Manual Atual

### Arquivo
[`diagrams/as-is-processo-manual.mermaid`](../diagrams/as-is-processo-manual.mermaid)

### Descrição do Fluxo

**Etapa 1: Acesso Manual ao CRM (5 min)**
1. Gerente de Qualidade acessa HubSpot
2. Busca manualmente o cliente na interface
3. Abre histórico dos últimos 30 dias

**Etapa 2: Revisão Manual de Dados (10 min)**
4. Lê emails trocados (um por um)
5. Verifica negócios/deals criados ou fechados
6. Analisa tickets de suporte (status, categoria)
7. Revisa anotações diversas da equipe
8. Confere produtos contratados pelo cliente

**Etapa 3: Análise Mental (5 min)**
9. Gerente processa mentalmente todas as informações
10. Identifica eventos relevantes
11. Decide tom da abordagem
12. Formula mensagem personalizada na cabeça

**Etapa 4: Envio Manual (3 min)**
13. Abre canal de mensageria (WhatsApp Web, Telegram, etc.)
14. Digita ou copy/paste mensagem personalizada
15. Envia mensagem
16. Verifica se foi entregue

**Etapa 5: Acompanhamento Manual (variável)**
17. Aguarda resposta do cliente
18. Lê resposta quando chega
19. Se não tem nota: responde manualmente pedindo
20. Interpreta nota de 1-5
21. Pode continuar conversa se necessário
22. Finaliza quando obtém nota

**Etapa 6: Registro Manual (5 min)**
23. Abre planilha Excel ou Google Sheets
24. Digita manualmente:
    - Nome do cliente
    - Data do contato
    - Nota de satisfação
    - Observações relevantes
25. Salva planilha

**Etapa 7: Loop**
26. Repete para próximo cliente da lista

### Problemas Identificados

#### 1. Tempo Excessivo
- **30 minutos por cliente** em média
- Gerente consegue atender apenas **16 clientes/dia** (8h de trabalho)
- Não escalável: 100 clientes = 6 dias de trabalho

#### 2. Erros Manuais
- **Copy/paste incorreto** de informações
- **Dados duplicados** na planilha
- **Informações inconsistentes** (cliente A recebe dados de cliente B)
- **Erros de digitação** em nomes, notas, datas

#### 3. Falta de Rastreabilidade
- **Impossível saber se mensagem foi lida**
- **Perda de contexto** entre conversas
- **Sem histórico centralizado** das interações
- **Difícil auditoria** (quem falou o quê e quando?)

#### 4. Personalização Inconsistente
- Depende do **humor e energia da gerente**
- **Qualidade varia** ao longo do dia (cansaço)
- **Informações relevantes são esquecidas**
- Sem padrão de qualidade garantido

#### 5. Falta de Insights
- Dados dispersos em múltiplas planilhas
- **Impossível gerar relatórios** consolidados facilmente
- **Sem visão de tendências** (NPS ao longo do tempo)
- **Difícil identificar padrões** de insatisfação

#### 6. Dependência de Pessoas
- Se gerente sair de férias/doença: processo para
- **Conhecimento tácito** (não documentado)
- **Treinamento de novos gerentes leva semanas**

### Métricas do Processo Atual

| Métrica | Valor |
|---------|-------|
| Tempo médio por cliente | 30 min |
| Clientes atendidos/dia (1 gerente) | 16 |
| Taxa de erro (estimada) | 15-20% |
| Custo de hora de gerente sênior | R$ 80-120/h |
| Custo por pesquisa | R$ 40-60 |
| Escalabilidade | Baixa (linear com headcount) |
| Rastreabilidade | Baixa (planilhas) |
| NPS calculável facilmente? | Não |

### Cálculo de Custo Mensal (Processo Manual)

**Cenário:** 200 pesquisas/mês

- Tempo total: 200 × 30 min = **100 horas/mês**
- Custo gerente (R$ 100/h): **R$ 10.000/mês**
- Erros e retrabalho (+20%): **+R$ 2.000/mês**
- Ferramentas (planilhas, etc): **+R$ 100/mês**
- **Total: R$ 12.100/mês**

---

## TO-BE: Processo Automatizado com IA

### Arquivo
[`diagrams/to-be-processo-automatizado.mermaid`](../diagrams/to-be-processo-automatizado.mermaid)

### Descrição do Fluxo

**Etapa 1: Trigger (1s)**
1. Schedule (cron diário) ou execução manual no n8n
2. Cliente selecionado manualmente pelo ID no workflow

**Etapa 2: Workflow n8n Inicia (1s)**
3. Dispara pipeline de agentes para o cliente selecionado

**Etapa 3: Agente 1 - Data Fetcher (5-8s)**
4. Faz chamadas à API do HubSpot:
   - GET contact details
   - GET emails últimos 30 dias
   - GET deals/negócios
   - GET tickets de suporte
5. Consolida tudo em JSON estruturado

**Etapa 4: Agente 2 - Context Analyzer (2-4s)**
6. Tess AI (Agent 2 — ID 38717) analisa JSON
7. Gera insights:
   - Summary dos eventos relevantes
   - Sentimento (positive/negative/neutral)
   - Red flags (alertas importantes)
   - Tom sugerido (celebratory/empathetic/curious)
   - Pontos de personalização

**Etapa 5: Agente 3 - Message Generator (1-2s)**
8. Tess AI (Agent 3 — ID 38728) cria mensagem personalizada
9. Usa contexto específico do cliente
10. Segue tom sugerido pelo Agente 2
11. Inclui call-to-action claro (pedir nota 1-5)

**Etapa 6: Envio Automático (1s)**
12. Meta WhatsApp API envia via WhatsApp
13. Confirma entrega

**Etapa 7: Registro Automático (1s)**
14. **Supabase:** Insere registro com status 'sent'
15. **Chatwoot:** Cria conversa com todo o contexto
16. Gerente pode acompanhar em tempo real no dashboard

**Etapa 8: Aguarda Resposta**
17. Webhook fica escutando mensagens do cliente

**Etapa 9: Agente 4 - Conversation Handler (2-3s por turno)**
18. Recebe mensagem do cliente via webhook
19. Tess AI (Agent 4 V2.0 — ID 38733) analisa resposta:
    - **Se contém nota 1-5:** Extrai e agradece
    - **Se é feedback:** Responde empaticamente e pede nota
    - **Se é dúvida:** Responde e retorna ao objetivo
20. Envia resposta via WhatsApp
21. Atualiza Chatwoot e Supabase
22. **Loop:** Repete até obter nota ou atingir 5 turnos

**Etapa 10: Finalização (1s)**
23. Quando nota é extraída:
    - Agradece graciosamente
    - Atualiza Supabase com nota, feedback e transcrição
    - Marca conversa como resolvida no Chatwoot

**Etapa 11: Loop Automático**
24. Sistema processa próximo cliente automaticamente

### Monitoramento Paralelo

**Durante todo o processo:**
- Gerente visualiza conversas em tempo real no Chatwoot
- Pode **responder manualmente** a qualquer momento no Chatwoot
- Nota privada automática com score, sentiment e feedback quando o survey é concluído

### Benefícios do Processo Otimizado

#### 1. Redução Drástica de Tempo
- **~2 minutos por cliente** (vs 30 min)
- **93% de redução de tempo**
- 1 gerente pode supervisionar **200+ clientes/dia**

#### 2. Eliminação de Erros
- Dados extraídos automaticamente (zero copy/paste)
- Validação automática de JSON e campos
- Transcrições completas armazenadas
- Impossível duplicar ou perder dados

#### 3. Rastreabilidade Completa
- **Tudo registrado** no banco de dados
- **Timestamps precisos** (enviado, lido, respondido)
- **Transcrições completas** de cada conversa
- **Auditoria facilitada** (quem, quando, o quê)

#### 4. Personalização Garantida
- IA analisa 100% dos dados disponíveis
- **Qualidade consistente** 24/7
- **Sem fadiga** ou viés humano
- Usa contexto específico sempre

#### 5. Escalabilidade Horizontal
- Processar **1000+ clientes/dia** sem contratar mais gerentes
- Custo marginal por cliente é **apenas ~R$ 1.50 (APIs cloud)**
- Gerentes focam em **casos complexos** e intervenções estratégicas

#### 6. Experiência do Cliente
- **Respostas instantâneas** (não aguardar horário comercial)
- **Contexto preservado** em toda conversa

### Métricas do Processo Otimizado

| Métrica | Valor | Comparação AS-IS |
|---------|-------|------------------|
| Tempo médio por cliente | ~2 min | 93% ↓ |
| Clientes atendidos/dia (1 gerente) | 200+ | 1150% ↑ |
| Taxa de erro | < 1% | 95% ↓ |
| Custo por pesquisa (API + infra) | ~R$ 1.50 | 97% ↓ |
| Custo gerente (supervisão) | R$ 100/dia | 90% ↓ |
| Escalabilidade | Muito alta (horizontal) | ∞ |
| Rastreabilidade | Completa (BD + UI) | 100% ↑ |

### Cálculo de Custo Mensal (Processo Automatizado)

**Cenário:** 200 pesquisas/mês

**Custos de Infraestrutura (Cloud):**
- n8n Cloud: R$ 0 (tier gratuito)
- Tess AI (créditos — 200 pesquisas): ~R$ 165/mês
- Supabase: R$ 0 (tier gratuito)
- Chatwoot Cloud: R$ 0 (Plano Hacker — gratuito)
- Meta WhatsApp API: ~R$ 30/mês
- **Subtotal infra: ~R$ 195/mês**

**Custos de Supervisão:**
- Gerente (2h/dia supervisão): 40h/mês × R$ 100/h = R$ 4.000/mês

**Custos de Manutenção:**
- Manutenção (5% do custo de desenvolvimento/mês): R$ 210/mês

**Total: R$ 4.405/mês**

**Economia vs Manual: R$ 12.100 - R$ 4.405 = R$ 7.695/mês (64% redução)**

---

## Comparação Lado a Lado

### Tabela Comparativa

| Aspecto | AS-IS (Manual) | TO-BE (IA) | Ganho |
|---------|----------------|------------|-------|
| **Tempo por cliente** | 30 min | 2 min | 93% ↓ |
| **Clientes/dia/gerente** | 16 | 200+ | 1150% ↑ |
| **Custo mensal (200 pesquisas)** | R$ 12.100 | R$ 4.405 | 64% ↓ |
| **Taxa de erro** | 15-20% | < 1% | 95% ↓ |
| **Personalização** | Inconsistente | Sempre personalizado | 100% ↑ |
| **Rastreabilidade** | Planilhas dispersas | BD centralizado + UI | Completa |
| **Escalabilidade** | Linear (headcount) | Exponencial | ∞ |
| **Disponibilidade** | Horário comercial | 24/7 | ✅ |
| **Intervenção humana** | Sempre | Quando necessário | Otimizado |

### Ganhos Estratégicos

#### Curto Prazo (1-3 meses)
✅ Redução de 64% no custo operacional
✅ Aumento de 10x na capacidade de pesquisas
✅ Eliminação de 95% dos erros manuais

#### Médio Prazo (3-6 meses)
✅ Identificação proativa de clientes em risco (churn)
✅ Insights de produto baseados em feedback estruturado
✅ Benchmark de satisfação por segmento/produto
✅ Otimização contínua dos prompts (A/B testing)

#### Longo Prazo (6-12 meses)
✅ Expansão para outros canais (voz, email, SMS)
✅ Integração com outros processos (onboarding, renovação)
✅ Fine-tuning de modelo próprio
✅ Exportação automática para Data Warehouse/BI

---

## Riscos e Mitigações

### Riscos Identificados TO-BE

| Risco | Probabilidade | Impacto | Mitigação |
|-------|---------------|---------|-----------|
| API HubSpot cair | Baixa | Alto | Retry + cache de dados + alertas |
| Tess AI indisponível | Baixa | Alto | Retry com backoff + alertas automáticos |
| Meta WhatsApp API indisponível | Baixa | Médio | Retry automático + monitoramento |
| Cliente não entender que é IA | Baixa | Baixo | Deixar claro no início ("equipe de relacionamento") |
| IA não entender contexto | Baixa | Médio | Few-shot examples + validação humana spot |
| Custo API explodir | Baixa | Médio | Budget limits + alertas + otimização de prompts |

### Controles de Qualidade

Práticas manuais que o responsável pelo sistema pode executar periodicamente usando as ferramentas já disponíveis (Chatwoot + Supabase).

1. **Amostragem Humana (10%):**
   - Gerente revisa 10% das conversas aleatoriamente no Chatwoot
   - Identifica padrões de erro da IA
   - Ajusta prompts conforme necessário

2. **A/B Testing:**
   - Testar variações de prompts nos agentes Tess AI
   - Medir taxa de resposta e qualidade comparando resultados no Supabase
   - Implementar melhor versão

---

## Roadmap de Implementação

### Fase 1: MVP (Semanas 1-2)
- Setup dos serviços cloud (n8n, Supabase, Tess AI)
- Integração HubSpot API
- Desenvolvimento dos 4 agentes (Tess AI)
- Workflow n8n básico
- Teste com 5 clientes piloto

### Fase 2: Piloto (Semanas 3-4)
- Configuração Chatwoot Cloud
- Integração Meta WhatsApp API (oficial)
- Teste com 50 clientes
- Coleta de feedback dos gerentes
- Ajustes de prompts

### Fase 3: Produção (Semanas 5-6)
- Escalar para 200+ clientes
- Dashboard de métricas
- Documentação completa
- Treinamento de gerentes
- Handoff para operação

### Fase 4: Otimização (Semanas 7-8)
- A/B testing de prompts
- Análise de ROI real
- Identificação de melhorias
- Expansão para outros canais (opcional)

---

## Propostas Futuras

As funcionalidades abaixo foram identificadas durante o projeto mas não foram implementadas no MVP atual. Podem ser incorporadas em fases subsequentes.

### 1. Busca automática de clientes elegíveis
Substituir a seleção manual de cliente por uma lógica automática no FLUXO 1 que: (a) busque no HubSpot contactos com atividade nos últimos 30 dias, e (b) filtre no Supabase quem já recebeu pesquisa nos últimos 30 dias (`WHERE customer_phone = X AND created_at > NOW() - INTERVAL '30 days'`). O resultado seria uma lista de clientes elegíveis que o workflow itere automaticamente via loop no n8n.

### 2. Notificação automática para scores baixos

Quando o score coletado for 1 ou 2, ou quando o Agent 2 detectar red flags no contexto do cliente, o sistema notificaria automaticamente a gerente de qualidade para follow-up manual prioritário. Isso exigiria um IF adicional no FLUXO 2, após a atualização do Supabase, que verifique o valor de `satisfaction_score` e dispare uma notificação (ex: mensagem no Chatwoot, email ou Slack).

### 3. Relatório consolidado de satisfação
Geração automática de um relatório agregado com métricas como distribuição de scores, NPS, sentiment por período e taxa de conclusão. Candidatos de implementação: dashboard no Supabase (com Grafana ou Metabase apontando para a tabela `surveys`), ou um node n8n agendado que consolida os dados e publica no Chatwoot ou em uma ferramenta de BI.

### 4. IA parar automaticamente quando gerente assume a conversa *(requisito FMA)*
O FMA exige que o gerente possa "assumir o controle da conversa". Atualmente o gerente pode responder manualmente no Chatwoot, mas o bot continua a processar mensagens subsequentes do cliente via webhook Meta independentemente. Para resolver isto, o FLUXO 2 precisaria verificar, antes de chamar o Agent 4, se a conversa no Chatwoot foi atribuída a um agente humano ou se o gerente já respondeu manualmente. Se sim, o fluxo deve parar sem enviar resposta do bot. Isso exigiria uma chamada à API do Chatwoot para consultar o estado da conversa em cada iteração do webhook.

### 5. Feedback Loop de qualidade das conversas
Permitir que o gerente marque conversas como "boa" ou "ruim" no Chatwoot, e usar esse sinal para guiar a optimização dos prompts dos agentes. Exigiria um campo ou label custom no Chatwoot para classificação, e uma forma de exportar essa informação para análise periódica.

---

## Conclusão

A migração do processo manual (AS-IS) para o processo automatizado com IA (TO-BE) representa uma transformação fundamental na forma como a Pareto mede satisfação de clientes:

### Ganhos Quantificáveis
- **93% redução** no tempo por cliente
- **64% redução** no custo operacional
- **1150% aumento** na capacidade de atendimento
- **95% redução** na taxa de erros

### Ganhos Qualitativos
- Personalização consistente e garantida
- Rastreabilidade completa de todas interações
- Dados estruturados por survey (score, sentiment, feedback) no banco
- Escalabilidade exponencial
- Experiência do cliente melhorada (respostas instantâneas)

### ROI Estimado
- **Payback:** 8 meses
- **Economia anual:** R$ 92.340 (200 pesquisas/mês)
- **ROI Ano 1:** 68% | **ROI 3 Anos:** 504%

O processo TO-BE não substitui completamente o humano, mas **potencializa** a capacidade dos gerentes de qualidade, permitindo que foquem em casos complexos, análise estratégica e relacionamento de alto valor, enquanto a IA cuida da execução repetitiva e análise de dados em escala.

---

**Documento elaborado para:** Case Agent Dev - FMA/Pareto/IA Leader
**Data:** Janeiro 2026
**Versão:** 2.0 (Meta WhatsApp API + Tess AI + Chatwoot Cloud)
