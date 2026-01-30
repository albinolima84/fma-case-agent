# Detalhamento dos Agentes de IA

## Visão Geral do Sistema Multi-Agent

O sistema utiliza um padrão de **pipeline sequencial** com 4 agentes especializados, cada um responsável por uma etapa específica do processo de pesquisa de satisfação.

### Arquitetura de Agentes

```
┌──────────────────────┐
│  AGENTE 1            │
│  Data Fetcher        │  → Coleta dados brutos do HubSpot
└──────────┬───────────┘
           │
           ↓ JSON estruturado
┌──────────────────────┐
│  AGENTE 2            │
│  Context Analyzer    │  → Analisa e gera insights
└──────────┬───────────┘
           │
           ↓ Summary + sentiment
┌──────────────────────┐
│  AGENTE 3            │
│  Message Generator   │  → Cria mensagem personalizada
└──────────┬───────────┘
           │
           ↓ Mensagem de abertura
┌──────────────────────┐
│  AGENTE 4            │
│  Conversation        │  → Gerencia conversa bidirecional
│  Handler             │
└──────────────────────┘
```

### Justificativa da Abordagem Multi-Agent

**Por que não um único agente?**
- **Separação de responsabilidades:** Cada agente tem uma tarefa clara e testável
- **Reutilização:** Agente 1 e 2 podem ser reutilizados para outros casos de uso
- **Manutenibilidade:** Ajustar um prompt não afeta os outros
- **Escalabilidade:** Agentes podem ser executados em paralelo (futuro)
- **Qualidade:** Especialização resulta em outputs mais consistentes
- **Debugging:** Fácil identificar qual agente gerou um resultado ruim

---

## AGENTE 1: Data Fetcher (Coletor de Dados)

### Responsabilidade
Buscar e consolidar todo o histórico relevante do cliente no HubSpot CRM dos últimos 30 dias.

### Tipo de Agente
**Não é LLM** - É um workflow n8n com chamadas REST para HubSpot API v3.

### Input
```json
{
  "hubspot_contact_id": "12345",
  "period_days": 30
}
```

### Processamento (n8n nodes)

**Node 1: Get Contact Details**
```http
GET https://api.hubapi.com/crm/v3/objects/contacts/{contact_id}
Headers:
  Authorization: Bearer {HUBSPOT_API_KEY}
Query params:
  properties=firstname,lastname,email,phone,company,createdate
```

**Node 2: Get Associated Deals**
```http
GET https://api.hubapi.com/crm/v3/objects/contacts/{contact_id}/associations/deals
```

**Node 3: Get Recent Emails**
```http
GET https://api.hubapi.com/crm/v3/objects/emails/search
Body:
{
  "filterGroups": [{
    "filters": [
      { "propertyName": "associations.contact", "operator": "EQ", "value": "{contact_id}" },
      { "propertyName": "hs_timestamp", "operator": "GTE", "value": "{30_days_ago_timestamp}" }
    ]
  }],
  "sorts": [{ "propertyName": "hs_timestamp", "direction": "DESCENDING" }],
  "limit": 20
}
```

**Node 4: Get Tickets (Support)**
```http
GET https://api.hubapi.com/crm/v3/objects/tickets/search
Body:
{
  "filterGroups": [{
    "filters": [
      { "propertyName": "associations.contact", "operator": "EQ", "value": "{contact_id}" }
    ]
  }]
}
```

**Node 5: Get Notes/Activities**
```http
GET https://api.hubapi.com/crm/v3/objects/notes/search
Body:
{
  "filterGroups": [{
    "filters": [
      { "propertyName": "associations.contact", "operator": "EQ", "value": "{contact_id}" },
      { "propertyName": "hs_timestamp", "operator": "GTE", "value": "{30_days_ago_timestamp}" }
    ]
  }]
}
```

**Node 6: Merge and Structure Data** (Function node - JavaScript)
```javascript
// Consolida todos os dados em JSON estruturado
const contact = $node["Get Contact"].json;
const deals = $node["Get Deals"].json.results;
const emails = $node["Get Emails"].json.results;
const tickets = $node["Get Tickets"].json.results;
const notes = $node["Get Notes"].json.results;

return {
  contact: {
    id: contact.id,
    name: `${contact.properties.firstname} ${contact.properties.lastname}`,
    email: contact.properties.email,
    phone: contact.properties.phone,
    company: contact.properties.company,
    created_date: contact.properties.createdate
  },
  summary: {
    total_emails: emails.length,
    total_deals: deals.length,
    total_tickets: tickets.length,
    total_notes: notes.length
  },
  deals: deals.map(d => ({
    id: d.id,
    name: d.properties.dealname,
    stage: d.properties.dealstage,
    amount: d.properties.amount,
    close_date: d.properties.closedate,
    created_date: d.properties.createdate
  })),
  emails: emails.map(e => ({
    subject: e.properties.hs_email_subject,
    direction: e.properties.hs_email_direction, // INCOMING_EMAIL or EMAIL
    timestamp: e.properties.hs_timestamp,
    snippet: e.properties.hs_email_text?.substring(0, 200) // primeiros 200 chars
  })),
  tickets: tickets.map(t => ({
    id: t.id,
    subject: t.properties.subject,
    status: t.properties.hs_ticket_priority,
    category: t.properties.hs_ticket_category, // churn, downgrade, support, etc
    created_date: t.properties.createdate,
    closed_date: t.properties.closed_date
  })),
  notes: notes.map(n => ({
    body: n.properties.hs_note_body,
    timestamp: n.properties.hs_timestamp
  })),
  products: [] // TODO: se HubSpot tiver produtos associados
};
```

### Output
```json
{
  "contact": {
    "id": "12345",
    "name": "João Silva",
    "email": "joao@empresa.com",
    "phone": "11999999999",
    "company": "Empresa XYZ Ltda",
    "created_date": "2024-11-15T10:00:00Z"
  },
  "summary": {
    "total_emails": 12,
    "total_deals": 2,
    "total_tickets": 1,
    "total_notes": 3
  },
  "deals": [
    {
      "id": "67890",
      "name": "Upgrade para Plano Pro",
      "stage": "closedwon",
      "amount": "997.00",
      "close_date": "2025-01-10",
      "created_date": "2025-01-05"
    }
  ],
  "emails": [
    {
      "subject": "Dúvida sobre integração",
      "direction": "INCOMING_EMAIL",
      "timestamp": "2025-01-20T14:30:00Z",
      "snippet": "Olá, estou com dificuldade para integrar..."
    }
  ],
  "tickets": [
    {
      "id": "11111",
      "subject": "Erro ao exportar relatório",
      "status": "HIGH",
      "category": "support",
      "created_date": "2025-01-18T09:00:00Z",
      "closed_date": "2025-01-18T15:00:00Z"
    }
  ],
  "notes": [
    {
      "body": "Cliente demonstrou interesse em features avançadas",
      "timestamp": "2025-01-12T16:00:00Z"
    }
  ],
  "products": []
}
```

### Tratamento de Erros
- Se HubSpot API retornar 429 (rate limit): retry com backoff exponencial
- Se contato não existir (404): registrar erro e pular
- Se token expirado (401): alertar administrador
- Timeout de 30s por requisição

### Performance
- **Tempo médio de execução:** 5-8 segundos
- **Paralelização:** Chamadas de API podem ser feitas em paralelo (n8n suporta)
- **Cache:** Considerar cache de 1h para contatos que não mudaram (otimização futura)

---

## AGENTE 2: Context Analyzer (Analisador de Contexto)

### Responsabilidade
Analisar dados brutos do HubSpot e gerar insights focados em satisfação do cliente, identificando eventos-chave e determinando o tom adequado para a abordagem.

### Tipo de Agente
**LLM** - Claude 3.5 Sonnet via Anthropic API

### Input
Output completo do Agente 1 (JSON com ~5-20kb)

### Prompt System

```
Você é um Analista de Relacionamento Sênior especializado em Customer Success e Satisfação de Clientes.

Sua missão é analisar o histórico recente de interação de um cliente com nossa empresa e gerar insights focados em medir satisfação.

## Contexto
Estamos conduzindo uma pesquisa proativa de satisfação para entender como o cliente está se sentindo em relação aos nossos serviços. Você receberá dados estruturados do nosso CRM (HubSpot) contendo:
- Informações básicas do contato
- Emails trocados nos últimos 30 dias
- Negócios/deals (criados, ganhos ou perdidos)
- Tickets de suporte abertos/resolvidos
- Notas e atividades registradas
- Produtos contratados

## Sua Tarefa
Analise os dados e produza um JSON estruturado com os seguintes campos:

1. **summary** (string): Resumo conciso (100-150 palavras) dos eventos mais relevantes do período. Foque em:
   - Marcos importantes (upgrade, novos produtos, renovação)
   - Interações de suporte (tickets resolvidos ou não)
   - Nível de engajamento (quantidade e qualidade de interações)
   - Sinais de satisfação ou insatisfação

2. **sentiment** (string): Sentimento geral baseado no histórico. Valores possíveis:
   - "very_positive": Cliente claramente satisfeito (upgrade, elogios, alta atividade positiva)
   - "positive": Cliente engajado e sem problemas aparentes
   - "neutral": Pouca interação ou mix de positivo/negativo
   - "negative": Sinais de insatisfação (tickets sem resolução, downgrade, reclamações)
   - "very_negative": Cliente em churn ou com múltiplos problemas graves

3. **key_events** (array de strings): Lista de 3-5 eventos-chave mais relevantes para a conversa de satisfação. Exemplos:
   - "upgrade_to_premium_plan"
   - "support_ticket_resolved_quickly"
   - "support_ticket_unresolved"
   - "contract_renewal"
   - "trial_ending_soon"
   - "low_engagement_last_30_days"
   - "churn_risk_detected"

4. **suggested_tone** (string): Tom recomendado para a mensagem inicial. Valores possíveis:
   - "celebratory": Para clientes com eventos muito positivos (upgrade, milestone)
   - "appreciative": Para clientes engajados e satisfeitos
   - "curious": Para clientes neutros ou com pouca interação
   - "empathetic": Para clientes com problemas recentes
   - "proactive": Para clientes com sinais de risco

5. **red_flags** (array de strings): Lista de alertas importantes que a gerente deve saber. Exemplos:
   - "Ticket de churn aberto há 5 dias"
   - "Cliente não respondeu últimos 3 emails"
   - "Downgrade de plano Pro para Basic"
   - "Múltiplos tickets de suporte no período"

6. **personalization_points** (array de strings): 2-3 pontos específicos para personalizar a mensagem. Exemplos:
   - "Mencionar o upgrade para plano Pro"
   - "Referenciar a resolução rápida do ticket de integração"
   - "Perguntar sobre a experiência com o novo dashboard"

## Regras Importantes
- Seja objetivo e baseado em dados
- Evite suposições não fundamentadas nos dados
- Se houver poucos dados, sinalize isso no summary
- Priorize eventos recentes (últimos 7-15 dias têm mais peso)
- Red flags são críticos: sempre liste se existirem

## Output Format
Retorne APENAS um JSON válido, sem markdown nem explicações adicionais:

{
  "summary": "...",
  "sentiment": "...",
  "key_events": [...],
  "suggested_tone": "...",
  "red_flags": [...],
  "personalization_points": [...]
}
```

### Prompt User (exemplo)
```json
{
  "contact": {
    "id": "12345",
    "name": "João Silva",
    "email": "joao@empresa.com",
    "phone": "11999999999",
    "company": "Empresa XYZ Ltda",
    "created_date": "2024-11-15T10:00:00Z"
  },
  "summary": {
    "total_emails": 12,
    "total_deals": 2,
    "total_tickets": 1,
    "total_notes": 3
  },
  "deals": [
    {
      "id": "67890",
      "name": "Upgrade para Plano Pro",
      "stage": "closedwon",
      "amount": "997.00",
      "close_date": "2025-01-10",
      "created_date": "2025-01-05"
    }
  ],
  "emails": [
    {
      "subject": "Dúvida sobre integração",
      "direction": "INCOMING_EMAIL",
      "timestamp": "2025-01-20T14:30:00Z",
      "snippet": "Olá, estou com dificuldade para integrar..."
    },
    {
      "subject": "Re: Dúvida sobre integração",
      "direction": "EMAIL",
      "timestamp": "2025-01-20T15:00:00Z",
      "snippet": "Oi João, vou te ajudar com isso..."
    }
  ],
  "tickets": [
    {
      "id": "11111",
      "subject": "Erro ao exportar relatório",
      "status": "CLOSED",
      "category": "support",
      "created_date": "2025-01-18T09:00:00Z",
      "closed_date": "2025-01-18T15:00:00Z"
    }
  ],
  "notes": [
    {
      "body": "Cliente demonstrou interesse em features avançadas durante call de onboarding",
      "timestamp": "2025-01-12T16:00:00Z"
    }
  ]
}
```

### Output Esperado
```json
{
  "summary": "João é cliente há cerca de 2 meses e recentemente fez upgrade para o Plano Pro (10/01). Nos últimos 30 dias, teve 12 interações por email e abriu 1 ticket de suporte que foi resolvido no mesmo dia (erro de exportação de relatório). Demonstrou interesse em features avançadas durante onboarding. Seu último contato foi há 7 dias com dúvida sobre integração que foi prontamente respondida. Cliente apresenta bom nível de engajamento e utilização ativa da plataforma.",
  "sentiment": "positive",
  "key_events": [
    "upgrade_to_pro_plan",
    "support_ticket_resolved_quickly",
    "active_email_engagement",
    "interest_in_advanced_features"
  ],
  "suggested_tone": "celebratory",
  "red_flags": [],
  "personalization_points": [
    "Mencionar o upgrade recente para Plano Pro",
    "Referenciar a resolução rápida do ticket de exportação",
    "Perguntar sobre experiência com features avançadas"
  ]
}
```

### Parâmetros de Configuração (Anthropic API)
```javascript
{
  model: "claude-3-5-sonnet-20241022",
  max_tokens: 1024,
  temperature: 0.3, // Baixa para consistência e objetividade
  system: "{PROMPT_SYSTEM_ACIMA}",
  messages: [
    {
      role: "user",
      content: JSON.stringify(dadosDoAgente1)
    }
  ]
}
```

### Tratamento de Resposta
```javascript
// n8n Function node para validar output
const response = $node["Claude_Analyzer"].json.content[0].text;

try {
  const analysis = JSON.parse(response);

  // Validações
  const validSentiments = ["very_positive", "positive", "neutral", "negative", "very_negative"];
  const validTones = ["celebratory", "appreciative", "curious", "empathetic", "proactive"];

  if (!validSentiments.includes(analysis.sentiment)) {
    throw new Error(`Invalid sentiment: ${analysis.sentiment}`);
  }

  if (!validTones.includes(analysis.suggested_tone)) {
    throw new Error(`Invalid tone: ${analysis.suggested_tone}`);
  }

  return analysis;

} catch (error) {
  // Fallback: se parsing falhar, usar valores default
  return {
    summary: "Erro ao analisar contexto. Prosseguindo com abordagem neutra.",
    sentiment: "neutral",
    key_events: [],
    suggested_tone: "curious",
    red_flags: ["Erro na análise automática"],
    personalization_points: []
  };
}
```

### Performance
- **Tempo médio:** 2-4 segundos
- **Tokens consumidos:** ~1500 input + 400 output = 1900 tokens/análise
- **Custo:** ~$0.006 por análise ($3 por milhão de tokens input, $15 por milhão output)

---

## AGENTE 3: Message Generator (Gerador de Mensagens)

### Responsabilidade
Criar mensagem personalizada de abertura para iniciar a conversa de satisfação via WhatsApp.

### Tipo de Agente
**LLM** - Claude 3.5 Sonnet via Anthropic API

### Input
```json
{
  "contact_name": "João Silva",
  "analysis": {
    "summary": "...",
    "sentiment": "positive",
    "suggested_tone": "celebratory",
    "personalization_points": [
      "Mencionar o upgrade recente para Plano Pro",
      "Perguntar sobre experiência com features avançadas"
    ]
  }
}
```

### Prompt System

```
Você é uma Gerente de Relacionamento da Pareto, especialista em comunicação empática e personalizada com clientes.

Sua missão é criar mensagens de abertura para pesquisas de satisfação via WhatsApp.

## Contexto
Estamos conduzindo uma pesquisa proativa para medir a satisfação dos nossos clientes. Você receberá:
- Nome do cliente
- Análise de contexto do relacionamento recente
- Tom sugerido para a abordagem

## Sua Tarefa
Crie uma mensagem de WhatsApp personalizada que:

1. **Seja conversacional e humana** - Como se uma pessoa real estivesse escrevendo
2. **Personalize com contexto específico** - Use os pontos de personalização fornecidos
3. **Seja concisa** - Máximo 4 linhas (cerca de 200 caracteres)
4. **Tenha objetivo claro** - Solicitar feedback/nota de satisfação
5. **Use tom apropriado** - Seguir o tom sugerido na análise

## Estrutura Recomendada
- Saudação + nome
- 1-2 linhas de contexto personalizado (referência a evento/interação recente)
- Pergunta sobre satisfação (de forma natural, não robótica)
- Call to action (pedir nota 1-5)

## Tons e Como Aplicar

**celebratory** (eventos muito positivos):
- Parabenizar/celebrar o marco
- Energia positiva e entusiasmo
- Exemplo: "Parabéns pelo upgrade! 🎉"

**appreciative** (cliente engajado):
- Agradecer pelo relacionamento
- Tom acolhedor e grato
- Exemplo: "Que bom ter você conosco!"

**curious** (neutro ou pouca interação):
- Tom investigativo gentil
- Interesse genuíno
- Exemplo: "Gostaria de saber como está sendo sua experiência..."

**empathetic** (problemas recentes):
- Empatia e cuidado
- Reconhecer dificuldades
- Exemplo: "Vi que você teve alguns desafios recentemente..."

**proactive** (sinais de risco):
- Tom atencioso mas não alarmista
- Foco em ajudar
- Exemplo: "Quero garantir que está tudo certo com você..."

## Regras Importantes
- NÃO use linguagem corporativa formal demais
- NÃO seja genérico (evite "Esperamos que esteja bem" sem contexto)
- NÃO use múltiplos emojis (máximo 1-2 se apropriado)
- NÃO faça a mensagem parecer automatizada
- SIM seja específico e use dados reais do contexto
- SIM seja breve e direto

## Exemplos de Mensagens Boas

**Exemplo 1 (celebratory):**
"Oi Maria! Vi que você fez upgrade para o Pro semana passada. Como está sendo a experiência com as novas funcionalidades? De 1 a 5, como você avaliaria nosso serviço?"

**Exemplo 2 (appreciative):**
"Olá Carlos! Notei que você tem usado bastante a plataforma esse mês. Que legal! Como você avalia nossa solução até agora? Nota de 1 a 5?"

**Exemplo 3 (empathetic):**
"Oi Ana, vi que você teve aquele problema com exportação de dados, mas conseguimos resolver rapidamente. Como está tudo agora? Avalia nosso suporte de 1 a 5?"

**Exemplo 4 (curious):**
"Olá Pedro! Você está conosco há 3 meses. Gostaria muito de saber: como tem sido sua experiência? Pode dar uma nota de 1 a 5?"

## Output Format
Retorne APENAS o texto da mensagem, sem aspas, sem formatação markdown, sem explicações adicionais.
A mensagem deve estar pronta para ser enviada via WhatsApp.
```

### Prompt User
```json
{
  "contact_name": "João Silva",
  "analysis": {
    "summary": "João é cliente há cerca de 2 meses e recentemente fez upgrade para o Plano Pro (10/01). Teve 1 ticket de suporte resolvido rapidamente. Cliente engajado e ativo.",
    "sentiment": "positive",
    "suggested_tone": "celebratory",
    "personalization_points": [
      "Mencionar o upgrade recente para Plano Pro",
      "Referenciar a resolução rápida do ticket de exportação",
      "Perguntar sobre experiência com features avançadas"
    ]
  }
}
```

### Output Esperado
```
Oi João! Vi que você fez upgrade para o Plano Pro recentemente. Como está sendo a experiência com as funcionalidades avançadas? De 1 a 5, como você avalia nosso serviço?
```

### Parâmetros de Configuração
```javascript
{
  model: "claude-3-5-sonnet-20241022",
  max_tokens: 256,
  temperature: 0.7, // Moderada para criatividade natural
  system: "{PROMPT_SYSTEM_ACIMA}",
  messages: [
    {
      role: "user",
      content: JSON.stringify({
        contact_name: contactName,
        analysis: analysisFromAgent2
      })
    }
  ]
}
```

### Validação de Output
```javascript
// n8n Function node
const message = $node["Claude_MessageGen"].json.content[0].text.trim();

// Validações
if (message.length > 400) {
  // Mensagem muito longa, truncar ou regenerar
  console.warn("Mensagem muito longa:", message.length);
}

if (!message.includes("1 a 5") && !message.includes("1-5")) {
  // Não solicitou nota explicitamente
  console.warn("Mensagem não solicita nota de 1-5");
}

return { message };
```

### Performance
- **Tempo médio:** 1-2 segundos
- **Tokens:** ~600 input + 100 output = 700 tokens
- **Custo:** ~$0.003 por mensagem

---

## AGENTE 4: Conversation Handler (Gerenciador de Conversa)

### Responsabilidade
Conduzir conversa bidirecional com o cliente, responder perguntas, extrair nota de satisfação e finalizar graciosamente.

### Tipo de Agente
**LLM Stateful** - Claude 3.5 Sonnet com histórico de conversa (mensagens anteriores mantidas)

### Input
```json
{
  "conversation_history": [
    {
      "role": "assistant",
      "content": "Oi João! Vi que você fez upgrade para o Plano Pro recentemente..."
    },
    {
      "role": "user",
      "content": "Muito bom! Estou gostando bastante"
    }
  ],
  "context": {
    "contact_name": "João Silva",
    "analysis_summary": "Cliente fez upgrade recente, ticket resolvido rapidamente",
    "satisfaction_score": null
  }
}
```

### Prompt System

```
Você é Sofia, Gerente de Relacionamento da Pareto. Sua missão é conduzir conversas de satisfação com clientes via WhatsApp de forma natural, empática e eficiente.

## Contexto da Conversa
Você iniciou uma conversa com um cliente para medir sua satisfação com nossos serviços. Você já enviou uma mensagem de abertura personalizada solicitando uma nota de 1 a 5.

## Seu Objetivo Principal
Extrair uma **nota de satisfação de 1 a 5** do cliente e entender o motivo por trás dessa nota.

## Como Conduzir a Conversa

### Se cliente respondeu com nota (1-5):
1. Agradeça pela nota
2. Se nota >= 4: celebre e peça feedback sobre o que está funcionando bem
3. Se nota <= 3: mostre empatia e pergunte o que podemos melhorar
4. Após 1-2 trocas adicionais, finalize graciosamente agradecendo

### Se cliente respondeu sem nota:
1. Se foi feedback positivo genérico ("muito bom", "gostando"): peça a nota de forma natural
2. Se foi feedback negativo ou neutro: mostre empatia e depois peça a nota
3. Se foi uma pergunta: responda de forma útil e depois retorne ao objetivo (pedir nota)

### Se cliente pediu para falar com humano:
1. Confirme que vai transferir
2. Retorne status "escalate" (você não continuará)

### Regras de Condução
- **Máximo 3 mensagens suas** antes de finalizar (evitar loops infinitos)
- Seja **breve** - mensagens de 1-2 linhas no máximo
- Seja **natural** - como um humano real conversaria via WhatsApp
- **NÃO seja insistente** - se cliente não quer dar nota, agradeça e finalize
- **NÃO faça múltiplas perguntas** na mesma mensagem
- **NÃO use linguagem corporativa** formal

### Quando Finalizar
Finalize a conversa quando:
- Cliente deu nota E explicou motivo (ou não quis explicar)
- Cliente claramente não quer continuar
- Você já enviou 3 mensagens
- Cliente pediu para falar com humano

## Output Format
Retorne um JSON com:

```json
{
  "response": "sua mensagem de resposta aqui",
  "status": "continue" | "completed" | "escalate",
  "extracted_data": {
    "satisfaction_score": 1-5 ou null,
    "feedback": "texto do feedback" ou null
  }
}
```

### Campos Explicados
- **response**: Texto da mensagem para enviar ao cliente
- **status**:
  - "continue" = conversa deve continuar, aguardar próxima resposta
  - "completed" = conversa finalizada com sucesso (nota extraída)
  - "escalate" = cliente pediu humano, transferir para gerente
- **extracted_data.satisfaction_score**: Nota de 1 a 5 se foi mencionada, senão null
- **extracted_data.feedback**: Qualquer feedback textual relevante

## Exemplos

### Exemplo 1: Cliente deu nota diretamente
**User:** "5! Estou muito satisfeito"
**Output:**
```json
{
  "response": "Que ótimo ouvir isso! 🎉 Muito obrigada pelo feedback. Estamos à disposição sempre que precisar!",
  "status": "completed",
  "extracted_data": {
    "satisfaction_score": 5,
    "feedback": "Muito satisfeito"
  }
}
```

### Exemplo 2: Cliente respondeu sem nota
**User:** "Está indo bem!"
**Output:**
```json
{
  "response": "Fico feliz! Se pudesse dar uma nota de 1 a 5, qual seria?",
  "status": "continue",
  "extracted_data": {
    "satisfaction_score": null,
    "feedback": "Está indo bem"
  }
}
```

### Exemplo 3: Cliente deu nota baixa
**User:** "2, tive alguns problemas"
**Output:**
```json
{
  "response": "Entendo, sinto muito pelos problemas. Pode me contar um pouco mais sobre o que aconteceu? Quero ajudar a melhorar sua experiência.",
  "status": "continue",
  "extracted_data": {
    "satisfaction_score": 2,
    "feedback": "Tive alguns problemas"
  }
}
```

### Exemplo 4: Cliente quer falar com humano
**User:** "Prefiro falar com alguém da equipe"
**Output:**
```json
{
  "response": "Claro! Vou transferir você para nossa equipe. Já vou conectar você com uma gerente. Aguarde um momento!",
  "status": "escalate",
  "extracted_data": {
    "satisfaction_score": null,
    "feedback": null
  }
}
```

### Exemplo 5: Cliente não quer dar nota
**User:** "Não quero responder agora"
**Output:**
```json
{
  "response": "Sem problemas! Quando quiser conversar, estamos por aqui. Obrigada! 😊",
  "status": "completed",
  "extracted_data": {
    "satisfaction_score": null,
    "feedback": "Não quis responder"
  }
}
```
```

### Parâmetros de Configuração
```javascript
{
  model: "claude-3-5-sonnet-20241022",
  max_tokens: 256,
  temperature: 0.8, // Mais alta para naturalidade conversacional
  system: "{PROMPT_SYSTEM_ACIMA}",
  messages: conversationHistory // Array completo de mensagens anteriores
}
```

### Lógica de Loop (n8n)
```javascript
// n8n Function node - Conversation Loop Logic
const response = JSON.parse($node["Claude_Conversation"].json.content[0].text);

// Tracking
const currentTurn = $json.turn_count || 1;
const maxTurns = 5;

// Decisão de continuar ou parar
if (response.status === "completed" || response.status === "escalate") {
  // Finalizar
  return {
    ...response,
    should_continue: false,
    final_status: response.status
  };
}

if (currentTurn >= maxTurns) {
  // Limite de turnos atingido, finalizar mesmo sem nota
  return {
    response: "Obrigada pelo seu tempo! Qualquer coisa, estamos à disposição. 😊",
    status: "completed",
    extracted_data: response.extracted_data,
    should_continue: false,
    final_status: "timeout"
  };
}

// Continuar conversa
return {
  ...response,
  should_continue: true,
  turn_count: currentTurn + 1
};
```

### Tratamento de Escalação
```javascript
// Se status === "escalate"
if (response.status === "escalate") {
  // 1. Atualizar Chatwoot para marcar conversa como "aguardando humano"
  // 2. Notificar gerente via Chatwoot (assignee)
  // 3. Parar o loop de IA
  // 4. Registrar no Supabase: status = "escalated"
}
```

### Performance
- **Tempo médio por resposta:** 2-3 segundos
- **Tokens por turno:** ~800 input + 100 output
- **Turnos médios por conversa:** 2-3
- **Custo por conversa completa:** ~$0.008-0.012

---

## Resumo: Custos por Pesquisa Completa

| Agente | Tokens | Custo/pesquisa |
|--------|--------|----------------|
| Agente 1 (Data Fetcher) | N/A (API calls) | $0 |
| Agente 2 (Context Analyzer) | ~1900 | $0.006 |
| Agente 3 (Message Generator) | ~700 | $0.003 |
| Agente 4 (Conversation) | ~2500 (média 3 turnos) | $0.010 |
| **TOTAL** | **~5100** | **$0.019** |

### Escala de Custos
- **50 pesquisas/dia:** $0.95/dia = $28/mês
- **200 pesquisas/dia:** $3.80/dia = $114/mês
- **1000 pesquisas/dia:** $19/dia = $570/mês

**Importante:** Custos de API Claude são variáveis. Valores baseados em:
- Input: $3 por 1M tokens
- Output: $15 por 1M tokens
- Modelo: claude-3-5-sonnet-20241022 (Janeiro 2025)

---

## Próximos Passos

1. ✅ Agentes definidos com responsabilidades claras
2. ⏳ Implementar prompts completos (pasta /prompts)
3. ⏳ Criar workflow n8n integrando os 4 agentes
4. ⏳ Testar com dados reais de um cliente
5. ⏳ Ajustar prompts baseado em resultados

---

**Documento elaborado para:** Case Agent Dev - FMA/Pareto/IA Leader
**Data:** Janeiro 2026
**Versão:** 1.0
