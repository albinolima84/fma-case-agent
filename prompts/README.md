# Prompts dos Agentes de IA

Esta pasta contém os prompts completos dos agentes de IA utilizados no sistema de pesquisa de satisfação automatizada.

## 📁 Arquivos

### Agente 2: Context Analyzer
**Arquivo:** `agent-2-context-analyzer.txt`

**Responsabilidade:** Analisar dados brutos do HubSpot e gerar insights focados em satisfação.

**Input:** JSON com histórico do cliente (emails, deals, tickets, notas)

**Output:** JSON com summary, sentiment, key_events, suggested_tone, red_flags, personalization_points

**Modelo:** gpt-4o-mini (via Tess AI)
**Temperatura:** 0.3 (baixa para consistência)
**Tokens médios:** ~1900
**Custo:** ~$0.006 por análise

---

### Agente 3: Message Generator
**Arquivo:** `agent-3-message-generator.txt`

**Responsabilidade:** Criar mensagem personalizada de abertura para WhatsApp.

**Input:** Nome do cliente + análise do Agente 2

**Output:** Texto da mensagem pronta para envio

**Modelo:** gpt-4o-mini (via Tess AI)
**Temperatura:** 0.7 (moderada para naturalidade)
**Tokens médios:** ~700
**Custo:** ~$0.003 por mensagem

---

### Agente 4: Conversation Handler
**Arquivo:** `agent-4-conversation-handler.txt`

**Responsabilidade:** Conduzir conversa bidirecional e extrair nota de satisfação.

**Input:** Histórico completo da conversa + nova mensagem do cliente

**Output:** JSON com response, status (continue/completed/escalate), extracted_data

**Modelo:** gpt-4o-mini (via Tess AI)
**Temperatura:** 0.8 (alta para conversação natural)
**Tokens médios:** ~800 por turno, ~2500 por conversa
**Custo:** ~$0.010 por conversa completa

---

## 🚀 Como Usar

### 1. Integração via Tess AI (implementação atual)

Os prompts estão configurados nos agentes Tess AI (IDs 38717, 38728, 38733) usando gpt-4o-mini.

**Exemplo de chamada (HTTP Request via curl):**

```bash
curl -X POST https://api.tess.im/agents/38717/execute \
  -H "Authorization: Bearer YOUR_TESS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "input": "<INPUT_JSON>"
  }'
```

### 2. Integração via n8n

No n8n, os agentes são chamados via node **HTTP Request** para a API da Tess AI:

**Configuração do node (Agente 2 — adaptar ID para cada agente):**
- Method: POST
- URL: `https://api.tess.im/agents/38717/execute`
- Authentication: Header Auth (`Authorization: Bearer YOUR_TESS_API_KEY`)
- Body (JSON):
```json
{
  "input": "{{$json.input_data}}"
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

**Nota:** Custos baseados em pricing do gpt-4o-mini (via Tess AI):
- Input: $0.15 por 1M tokens
- Output: $0.60 por 1M tokens

---

## 🔧 Configurações Recomendadas

### Agente 2: Context Analyzer
```json
{
  "model": "gpt-4o-mini",
  "max_tokens": 1024,
  "temperature": 0.3,
  "top_p": 1.0
}
```
**Justificativa:** Temperatura baixa garante consistência e objetividade na análise.

### Agente 3: Message Generator
```json
{
  "model": "gpt-4o-mini",
  "max_tokens": 256,
  "temperature": 0.7,
  "top_p": 1.0
}
```
**Justificativa:** Temperatura moderada permite criatividade natural mantendo coerência.

### Agente 4: Conversation Handler
```json
{
  "model": "gpt-4o-mini",
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

- [Tess AI Documentation](https://tess.im/pt-BR/docs)
- [OpenAI gpt-4o-mini Overview](https://platform.openai.com/docs/models/gpt-4o-mini)
- [Prompt Engineering Guide](https://platform.openai.com/docs/guides/prompt-engineering)
- [n8n HTTP Request Node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/)

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
