# Prompts dos Agentes de IA

Esta pasta contém os prompts completos dos agentes de IA utilizados no sistema de pesquisa de satisfação automatizada.

## 📁 Arquivos

### Agente 2: Context Analyzer
**Arquivo:** `agent-2-context-analyzer.txt`

**Responsabilidade:** Analisar dados brutos do HubSpot e gerar insights focados em satisfação.

**Input:** JSON com histórico do cliente (emails, deals, tickets, notas)

**Output:** JSON com summary, sentiment, key_events, suggested_tone, red_flags, personalization_points

**Modelo:** Claude 3.5 Sonnet
**Temperatura:** 0.3 (baixa para consistência)
**Tokens médios:** ~1900
**Custo:** ~$0.006 por análise

---

### Agente 3: Message Generator
**Arquivo:** `agent-3-message-generator.txt`

**Responsabilidade:** Criar mensagem personalizada de abertura para WhatsApp.

**Input:** Nome do cliente + análise do Agente 2

**Output:** Texto da mensagem pronta para envio

**Modelo:** Claude 3.5 Sonnet
**Temperatura:** 0.7 (moderada para naturalidade)
**Tokens médios:** ~700
**Custo:** ~$0.003 por mensagem

---

### Agente 4: Conversation Handler
**Arquivo:** `agent-4-conversation-handler.txt`

**Responsabilidade:** Conduzir conversa bidirecional e extrair nota de satisfação.

**Input:** Histórico completo da conversa + nova mensagem do cliente

**Output:** JSON com response, status (continue/completed/escalate), extracted_data

**Modelo:** Claude 3.5 Sonnet
**Temperatura:** 0.8 (alta para conversação natural)
**Tokens médios:** ~800 por turno, ~2500 por conversa
**Custo:** ~$0.010 por conversa completa

---

## 🚀 Como Usar

### 1. Integração via Anthropic API

Todos os prompts são projetados para uso com a API da Anthropic (Claude).

**Exemplo de chamada (Node.js):**

```javascript
const Anthropic = require('@anthropic-ai/sdk');

const anthropic = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
});

const message = await anthropic.messages.create({
  model: "claude-3-5-sonnet-20241022",
  max_tokens: 1024,
  temperature: 0.3,
  system: SYSTEM_PROMPT_DO_ARQUIVO,
  messages: [
    {
      role: "user",
      content: JSON.stringify(inputData)
    }
  ]
});

const response = message.content[0].text;
```

### 2. Integração via n8n

No n8n, use o node **HTTP Request** para chamar a API da Anthropic:

**Configuração do node:**
- Method: POST
- URL: `https://api.anthropic.com/v1/messages`
- Headers:
  - `x-api-key`: `{{$env.ANTHROPIC_API_KEY}}`
  - `anthropic-version`: `2023-06-01`
  - `content-type`: `application/json`
- Body (JSON):
```json
{
  "model": "claude-3-5-sonnet-20241022",
  "max_tokens": 1024,
  "temperature": 0.3,
  "system": "{{$node.ReadPrompt.json.system_prompt}}",
  "messages": [
    {
      "role": "user",
      "content": "{{$json.input_data}}"
    }
  ]
}
```

### 3. Leitura dos Prompts

Os arquivos `.txt` contêm:
- System prompt completo
- Few-shot examples
- Configurações recomendadas
- Validações e fallbacks
- Notas de implementação

**Para extrair apenas o system prompt:**

```javascript
const fs = require('fs');
const promptFile = fs.readFileSync('agent-2-context-analyzer.txt', 'utf8');

// Extrair seção SYSTEM PROMPT
const systemPromptMatch = promptFile.match(/===+\nSYSTEM PROMPT\n===+\n([\s\S]*?)\n===+/);
const systemPrompt = systemPromptMatch ? systemPromptMatch[1].trim() : null;
```

---

## 📊 Custos Estimados

### Por Pesquisa Completa (1 cliente)

| Agente | Tokens | Custo |
|--------|--------|-------|
| Agente 2 (Context Analyzer) | ~1900 | $0.006 |
| Agente 3 (Message Generator) | ~700 | $0.003 |
| Agente 4 (Conversation Handler) | ~2500 (3 turnos) | $0.010 |
| **TOTAL** | **~5100** | **$0.019** |

### Em Escala

| Volume | Custo/dia | Custo/mês |
|--------|-----------|-----------|
| 50 pesquisas/dia | $0.95 | $28 |
| 200 pesquisas/dia | $3.80 | $114 |
| 1000 pesquisas/dia | $19.00 | $570 |

**Nota:** Custos baseados em pricing da Anthropic (janeiro 2025):
- Input: $3 por 1M tokens
- Output: $15 por 1M tokens

---

## 🔧 Configurações Recomendadas

### Agente 2: Context Analyzer
```json
{
  "model": "claude-3-5-sonnet-20241022",
  "max_tokens": 1024,
  "temperature": 0.3,
  "top_p": 1.0
}
```
**Justificativa:** Temperatura baixa garante consistência e objetividade na análise.

### Agente 3: Message Generator
```json
{
  "model": "claude-3-5-sonnet-20241022",
  "max_tokens": 256,
  "temperature": 0.7,
  "top_p": 1.0
}
```
**Justificativa:** Temperatura moderada permite criatividade natural mantendo coerência.

### Agente 4: Conversation Handler
```json
{
  "model": "claude-3-5-sonnet-20241022",
  "max_tokens": 256,
  "temperature": 0.8,
  "top_p": 1.0
}
```
**Justificativa:** Temperatura alta gera respostas mais naturais e conversacionais.

---

## ✅ Validações Importantes

### Agente 2 Output
- ✅ JSON válido
- ✅ Campo `sentiment` é um dos valores permitidos
- ✅ Campo `suggested_tone` é um dos valores permitidos
- ✅ Arrays não estão vazios (exceto `red_flags`)
- ✅ Summary tem 50-200 palavras

### Agente 3 Output
- ✅ Mensagem não está vazia
- ✅ Contém "1 a 5" ou "1-5"
- ✅ Comprimento <= 400 caracteres
- ✅ Sem formatação markdown
- ⚠️ Warning se > 300 caracteres

### Agente 4 Output
- ✅ JSON válido
- ✅ Campo `status` é: continue, completed ou escalate
- ✅ Campo `response` não está vazio
- ✅ `satisfaction_score` é 1-5 ou null
- ✅ Response <= 500 caracteres

---

## 🎯 Few-Shot Examples

Todos os prompts incluem **few-shot examples** (exemplos de input/output) que ajudam o modelo a entender exatamente o formato e qualidade esperados.

### Como adicionar novos examples:

1. Colete conversas reais bem-sucedidas
2. Anonimize dados sensíveis
3. Adicione ao arquivo de prompt na seção `FEW-SHOT EXAMPLES`
4. Mantenha 5-7 examples variados (casos positivos, negativos, neutros, edge cases)

---

## 🔄 Versionamento de Prompts

**Versão atual:** 1.0 (Janeiro 2026)

### Changelog

**v1.0 (2026-01-27)**
- Criação inicial dos 3 prompts
- Few-shot examples adicionados
- Validações e fallbacks documentados

### Boas práticas:

- **Sempre versione** mudanças nos prompts
- **Teste A/B** antes de substituir prompts em produção
- **Mantenha histórico** de versões anteriores
- **Documente mudanças** e razões

---

## 📈 Otimização e Melhoria Contínua

### Métricas a monitorar:

**Agente 2:**
- Acurácia do sentiment (validação manual de amostra)
- Qualidade dos personalization_points (úteis para Agente 3?)
- Taxa de red_flags detectados vs missed

**Agente 3:**
- Taxa de resposta dos clientes (mensagem engaja?)
- Tempo médio até primeira resposta
- Feedback de gerentes sobre qualidade

**Agente 4:**
- Taxa de extração de nota (% conversas com score)
- Número médio de turnos até completar
- Taxa de escalação (% pedidos de humano)
- NPS final do processo

### Como otimizar:

1. **Coletar dados:** Registre todas as execuções
2. **Analisar padrões:** Identifique onde IA falha ou não performa bem
3. **Ajustar prompts:** Adicione instruções específicas ou examples
4. **A/B test:** Compare versões diferentes
5. **Iterar:** Ciclo contínuo de melhoria

---

## 🚨 Fallbacks e Tratamento de Erros

Cada prompt inclui fallbacks seguros caso a API falhe ou retorne formato incorreto.

**Estratégia geral:**
1. Retry com backoff exponencial (max 3 tentativas)
2. Se parsing JSON falhar: usar fallback estruturado
3. Se API retornar erro crítico: escalar para humano
4. Registrar todos os erros para análise

**Exemplo de fallback (Agente 2):**
```json
{
  "summary": "Análise automática não disponível. Revisão manual necessária.",
  "sentiment": "neutral",
  "key_events": [],
  "suggested_tone": "curious",
  "red_flags": ["Erro na análise automática"],
  "personalization_points": []
}
```

Isso garante que o sistema nunca trava completamente.

---

## 🔗 Links Úteis

- [Anthropic API Documentation](https://docs.anthropic.com/claude/reference/getting-started-with-the-api)
- [Claude Model Comparison](https://docs.anthropic.com/claude/docs/models-overview)
- [Prompt Engineering Guide](https://docs.anthropic.com/claude/docs/prompt-engineering)
- [n8n Anthropic Integration](https://docs.n8n.io/integrations/builtin/credentials/anthropic/)

---

## 📞 Suporte

Para dúvidas sobre os prompts:
1. Consulte a documentação completa em `/docs/02-agentes-ia-detalhamento.md`
2. Revise os few-shot examples incluídos em cada arquivo
3. Teste com dados de exemplo antes de usar em produção
4. Monitore métricas e ajuste conforme necessário

---

**Última atualização:** 2026-01-27
**Versão:** 1.0
**Projeto:** Case Agent Dev - FMA/Pareto/IA Leader
