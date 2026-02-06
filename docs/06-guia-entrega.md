# Guia de Entrega - Demo Completo

**Projeto:** Sistema de Satisfação com IA
**Objetivo:** Coletar evidências, links e métricas para a apresentação final
**Público:** Desenvolvedor responsável pela entrega

---

## 📸 Parte 1: Coletar Evidências

### 1.1. Screenshots Necessários

Tire prints de tela de:

1. **n8n Workflow Overview**
   - Vista geral do workflow com todos os nodes
   - Zoom em cada parte (FLUXO 1: Envio Proativo e FLUXO 2: Respostas)

2. **n8n Execution Logs**
   - Execução bem-sucedida do envio inicial
   - Execução do loop de conversa
   - Detalhes de cada agente (input/output)

3. **Supabase Database**
   - Tabela `surveys` com registros de teste
   - Campos `status`, `satisfaction_score`, `conversation_transcript` preenchidos

4. **WhatsApp Conversa Real**
   - Print da conversa completa cliente ↔ bot
   - Mostrar naturalidade das respostas

5. **Chatwoot Dashboard**
   - Conversa com contexto do cliente
   - Nota privada com score
   - Conversa marcada como resolvida

6. **Tess AI**
   - Dashboard mostrando os 3 agentes configurados
   - Uso de créditos (se disponível)

### 1.2. Exportar Dados para Análise

**Transcrição de Conversa:**

No Supabase SQL Editor:

```sql
SELECT
  customer_name,
  satisfaction_score,
  main_feedback,
  conversation_transcript,
  created_at,
  updated_at
FROM surveys
WHERE status = 'completed'
ORDER BY updated_at DESC
LIMIT 5;
```

**Métricas de Performance:**

```sql
-- Tempo médio de conclusão (updated_at marca o momento da finalização)
SELECT
  AVG(EXTRACT(EPOCH FROM (updated_at - created_at))) / 60 as avg_minutes,
  COUNT(*) as total_surveys,
  AVG(satisfaction_score) as avg_score
FROM surveys
WHERE status = 'completed';
```

### 1.3. Exemplo de Transcrição Formatada

Crie um arquivo `docs/exemplo-conversa-real.md`:

```markdown
# Exemplo de Conversa Real - Pesquisa de Satisfação

**Cliente:** João Silva
**Data:** 2026-01-28 14:30
**Duração:** 3 minutos
**Nota Final:** 5/5
**Status:** Completed

---

## Transcrição

**[14:30] Bot:**
> Oi João! Vi que você fez upgrade para o Plano Pro recentemente. Como está sendo a experiência com as funcionalidades avançadas? De 1 a 5, como você avalia nosso serviço?

**[14:31] João Silva:**
> 5! Estou muito satisfeito, a plataforma é excelente!

**[14:31] Bot:**
> Que ótimo saber que você está satisfeito! 😊 O que você mais gosta na plataforma?

**[14:32] João Silva:**
> A facilidade de usar as automações e o suporte rápido quando precisei

**[14:32] Bot:**
> Perfeito! Ficamos muito felizes em saber que conseguimos te ajudar. Obrigado pelo feedback, João! Vamos continuar melhorando para você 🙂

---

## Análise Automática

**Sentiment Detected:** very_positive
**Satisfaction Score:** 5
**Key Feedback Points:**
- Facilidade de uso das automações
- Qualidade do suporte (rápido)
- Upgrade para Plano Pro foi positivo

**Action Taken:** Survey completed, data saved to Supabase, conversa resolvida no Chatwoot
```

---

## 🔗 Parte 2: Links Públicos

### 2.1. Compartilhar Workflow n8n

No n8n Cloud:

1. Abra o workflow
2. Clique no menu "..." → **"Share"**
3. Escolha **"Create Template"**
4. Preencha nome e descrição
5. Marque **"Public"**
6. Copie o link público gerado

### 2.2. Compartilhar Dados do Supabase (Read-Only)

**IMPORTANTE:** Nunca compartilhe service_role key em público!

Criar uma view pública (somente leitura):

```sql
-- Criar view pública
CREATE VIEW public_surveys AS
SELECT
  id,
  customer_name,
  satisfaction_score,
  main_feedback,
  status,
  created_at,
  updated_at
FROM surveys;

-- Permitir acesso público à view
GRANT SELECT ON public_surveys TO anon;
```

Para compartilhar:
- Project URL: `https://xxxxxxx.supabase.co`
- Anon key: (pode ser pública)
- Tabela: `public_surveys` (view read-only)

### 2.3. Agentes Tess AI (Links Públicos)

Os agentes são acessados pela plataforma Tess AI e também chamados via API pelo workflow n8n:

| Agente | Link Público | Endpoint API |
|--------|-------------|--------------|
| Agent 2 – Context Analyzer | [tess.im/...context-analyzer](https://tess.im/pt-BR/dashboard/user/ai/chat/ai-chat/context-analyzer-6TBb4l) | `POST https://api.tess.im/agents/38717/execute` |
| Agent 3 – Message Generator | [tess.im/...message-generator](https://tess.im/pt-BR/dashboard/user/ai/chat/ai-chat/message-generator-vE6X3l) | `POST https://api.tess.im/agents/38728/execute` |
| Agent 4 – Conversation Handler | [tess.im/...conversation-handler](https://tess.im/pt-BR/dashboard/user/ai/chat/ai-chat/conversation-handler-GjH0pE) | `POST https://api.tess.im/agents/38733/execute` |

Os prompts completos estão no diretório `prompts/` do repositório.

### 2.4. Resumo de Links

```markdown
## 🔗 Links e Recursos

### Agentes Tess AI (Públicos)
- **Agent 2 – Context Analyzer:** https://tess.im/pt-BR/dashboard/user/ai/chat/ai-chat/context-analyzer-6TBb4l
- **Agent 3 – Message Generator:** https://tess.im/pt-BR/dashboard/user/ai/chat/ai-chat/message-generator-vE6X3l
- **Agent 4 – Conversation Handler:** https://tess.im/pt-BR/dashboard/user/ai/chat/ai-chat/conversation-handler-GjH0pE

### Workflow n8n
- **Template n8n (público):** https://n8n.io/workflows/xxxxx
- **Export (.json):** `workflows/satisfaction-survey-workflow.json` (no repositório)

### Banco de Dados (Read-Only Demo)
- **Supabase Project:** https://xxxxxxx.supabase.co
- **Tabela Pública:** `public_surveys`

### APIs Utilizadas
- **n8n Cloud:** https://n8n.io
- **Supabase:** https://supabase.com
- **Tess AI:** https://tess.im
- **Meta WhatsApp Business API:** https://developers.facebook.com
- **Chatwoot Cloud:** https://app.chatwoot.com
- **HubSpot:** https://developers.hubspot.com
```

---

## 📊 Parte 3: Métricas e Validação

### 3.1. Testes de Validação

Execute os seguintes testes e documente resultados:

| Teste | Objetivo | Resultado Esperado |
|-------|----------|-------------------|
| 1. Envio inicial | Verificar integração completa | Mensagem enviada via WhatsApp, registro no Supabase, conversa no Chatwoot |
| 2. Conversa 1 turno | Testar detecção de nota | Score extraído corretamente, status = completed |
| 3. Conversa 3 turnos | Testar loop conversacional | Conversa natural, encerramento gracioso |
| 4. Nota ambígua | Testar parsing variado | "muito bom" → 4 ou 5 |
| 5. Limite de turnos | Testar máximo 5 turnos | Conversa encerrada após 5 turnos |

### 3.2. Métricas de Performance

Documente:

**Tempo de Execução:**
- Envio inicial (todos os agentes): X segundos
- Resposta conversacional (Agente 4): Y segundos
- Total por pesquisa completa: Z segundos

**Custos Reais:**
- Créditos Tess AI usados por pesquisa: ~X
- Custo Meta WhatsApp API por mensagem: ~$X
- Custo total por pesquisa: ~R$ X

**Taxa de Sucesso:**
- Pesquisas completadas: X%
- Notas extraídas corretamente: Y%

### 3.3. Análise de Qualidade

**Naturalidade das Respostas:**
- Peça para 2-3 pessoas testarem
- Elas perceberam que era bot?
- Feedback sobre tom e empatia

**Precisão da Análise:**
- Agente 2 detectou sentiment corretamente?
- Personalization points foram relevantes?
- Tone sugerido foi apropriado?

---

## ✅ Parte 4: Checklist Final

### Infraestrutura
- [ ] n8n Cloud ativo e workflow funcionando
- [ ] Supabase com tabela criada e dados de teste
- [ ] Meta WhatsApp Business API conectada e webhook verificado pela Meta
- [ ] Tess AI com os 3 agentes configurados e créditos disponíveis
- [ ] Chatwoot Cloud com inbox criado e credenciais configuradas
- [ ] HubSpot conectado (ou mock de dados ativo)

### Testes Realizados
- [ ] Envio inicial testado com sucesso (mensagem recebida no WhatsApp)
- [ ] Conversa completa testada (mínimo 2-3 turnos)
- [ ] Nota de satisfação extraída corretamente
- [ ] Dados salvos no Supabase corretamente (`status = 'completed'`)
- [ ] Conversa registrada e resolvida no Chatwoot
- [ ] Logs de execução capturados

### Evidências Coletadas
- [ ] Screenshots do workflow n8n (overview + detalhes)
- [ ] Screenshots de execuções bem-sucedidas
- [ ] Print da conversa real no WhatsApp
- [ ] Print do Supabase com dados salvos
- [ ] Print do Chatwoot com conversa e nota privada
- [ ] Transcrição formatada da conversa
- [ ] Métricas de performance documentadas
- [ ] Fluxogramas AS-IS e TO-BE (`diagrams/*.png`)
- [ ] Vídeo demo ou demo ao vivo com tutorial de uso

### Links Públicos
- [ ] Template n8n publicado (link público)
- [ ] Export do workflow n8n com link público (`workflows/satisfaction-survey-workflow.json`)
- [ ] Links dos agentes Tess AI publicados
- [ ] Supabase view read-only criada (opcional)
- [ ] Links testados e funcionando

### Documentação
- [ ] Guia de deploy completo (`docs/05-guia-deploy-online.md`)
- [ ] README atualizado com links
- [ ] Exemplo de conversa documentado

---

## 🎁 Parte 5: Extras para Impressionar Avaliadores

### 1. Vídeo Demo (Obrigatório: vídeo ou demo ao vivo)

Grave um vídeo de 3-5 minutos mostrando:
1. Acesso ao n8n Cloud (workflow overview)
2. Executar manualmente um envio
3. Receber mensagem no WhatsApp
4. Responder e ver bot conversando
5. Mostrar dados salvos no Supabase + conversa no Chatwoot
6. Mostrar logs de execução

**Ferramentas:**
- Loom (gratuito, 5min suficiente)
- OBS Studio (gratuito, sem limites)

### 2. Monitoramento via Chatwoot

O Chatwoot já oferece visualização de todas as conversas em tempo real:
- Lista de conversas ativas/pendentes/concluídas
- Filtros por status e data
- Gerente pode assumir conversa manualmente a qualquer momento

---

**Última atualização:** 2026-02-05
**Versão:** 1.1
