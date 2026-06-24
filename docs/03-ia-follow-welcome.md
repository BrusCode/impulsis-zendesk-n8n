# 03 — IA-Follow: Logica do Welcome Reply

## Papel do IA-Follow neste Fluxo

O bot IA-Follow e o ponto central de validacao que determina se um novo ticket de WhatsApp deve ou nao ser roteado via fluxo Impulsis. Ele atua no **Welcome Reply** (primeira resposta automatica ao cliente) do novo ticket criado pelo Sunshine Conversations.

Sem essa validacao no bot, o Trigger de roteamento poderia disparar para qualquer mensagem de WhatsApp de clientes que tiveram algum disparo Impulsis no passado, causando roteamentos indevidos.

---

## Logica no Construtor de Dialogo

### Fluxo no Welcome Reply

```
[Novo ticket criado - Welcome Reply ativado]
         |
         v
[Passo 1: API Call - Busca tickets do requester]
GET /api/v2/search.json
?query=type:ticket requester:{ticket.requester.id}
       custom_field_50639967936148:true
       -id:{ticket.id}
         |
         v
[Passo 2: Condicional]
results.count > 0?
    /         \
  SIM          NAO
   |             |
   v             v
[Adiciona tag:  [Segue fluxo
 impulsis_novo_contato  normal do bot]
 no ticket atual]
   |
   v
[Encerra nó sem resposta ao cliente]
[Trigger Zendesk assume a partir daqui]
```

---

## Implementacao: Opcao A (HTTP Step nativo do construtor)

Se o construtor de dialogo suportar chamadas HTTP com condicionais:

**Step de API:**
- Metodo: GET
- URL: `https://webposto.zendesk.com/api/v2/search.json`
- Params: `query=type:ticket requester:{{ticket.requester_id}} custom_field_50639967936148:true -id:{{ticket.id}}`
- Auth: Basic (email/token)

**Condicional:**
- Se `{{response.count}} > 0` -> adiciona tag `impulsis_novo_contato`
- Caso contrario -> segue fluxo normal

**Adicionar tag via API:**
- Metodo: PUT
- URL: `https://webposto.zendesk.com/api/v2/tickets/{{ticket.id}}.json`
- Body: `{"ticket": {"tags": ["impulsis_novo_contato"]}}`

> Atencao: o PUT de tags substitui todas as tags existentes. Use o endpoint de tags especifico para adicionar sem substituir:
> POST `/api/v2/tickets/{{ticket.id}}/tags.json` com body `{"tags": ["impulsis_novo_contato"]}`

---

## Implementacao: Opcao B (N8N como intermediario)

Caso o construtor de dialogo nao suporte HTTP Steps com condicionais, use um webhook N8N intermediario:

**Fluxo alternativo:**
```
[Trigger Zendesk - Todo ticket WhatsApp criado]
         |
         v
[N8N Webhook Intermediario]
  - Busca tickets do requester com Retorno Ativo = true
  - Se encontrar: adiciona tag impulsis_novo_contato via API
  - Se nao encontrar: nao faz nada
         |
         v
[Trigger Zendesk - Roteamento detecta impulsis_novo_contato]
```

Neste caso, o IA-Follow nao precisa ser alterado. O N8N intermediario age antes do bot responder ao cliente.

**Vantagem:** Mais robusto, nao depende de capacidades do construtor de dialogo.
**Desvantagem:** Adiciona um webhook a mais na cadeia.

---

## Query de Busca Recomendada

```
type:ticket
requester:{REQUESTER_ID}
custom_field_50639967936148:true
status:pending OR status:open OR status:hold
-id:{NOVO_TICKET_ID}
```

Esta query retorna apenas tickets:
- Do mesmo cliente
- Com campo `Retorno Ativo` marcado como true
- Com status ativo (nao novo, nao resolvido)
- Excluindo o proprio ticket novo

---

## Consideracoes Importantes

### Nao responder ao cliente neste passo
Quando a tag `impulsis_novo_contato` e adicionada, o Welcome Reply NAO deve enviar nenhuma mensagem ao cliente. O roteamento automatico do Workflow 1 vai transferir o ticket para o agente, que iniciara a conversa de forma humanizada.

### Timeout e falha de API
Se a chamada de API do IA-Follow falhar por timeout, o ticket fica sem a tag semaforo e cai no fluxo normal do bot. Isso e aceitavel — e melhor do que um roteamento indevido. O agente pode identificar o caso manualmente pelo campo `Retorno Ativo` visivel no ticket original.

### Multiplos tickets com Retorno Ativo
Se o cliente tiver mais de um ticket com `Retorno Ativo = true` (cenario raro mas possivel), o Workflow 1 vai pegar o mais recentemente atualizado. O IA-Follow nao precisa tratar esse caso — a logica de selecao fica no N8N.
