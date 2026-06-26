# 05 — Testes e Troubleshooting

## Cenarios de Teste Ponta a Ponta

### Teste 1: Fluxo Completo Feliz

**Pre-condicoes:**
- Ticket original existe com `impulsis_ativo` e `Retorno Ativo = true`
- Workflows 1 e 2 ativos no N8N
- Triggers 1, 2 e 3 ativos no Zendesk
- Macro criado

**Passos:**
1. Agente usa Impulsis para disparar template para o cliente
2. Verificar: ticket original recebeu tag `impulsis_ativo`
3. Verificar: Trigger 1 setou campo `Retorno Ativo = true`
4. Cliente responde ao template
5. Verificar: novo ticket criado pelo Sunshine
6. Verificar: IA-Follow adicionou tag `impulsis_novo_contato` no novo ticket
7. Verificar: Trigger 2 disparou webhook (checar execucoes no N8N)
8. Verificar: novo ticket com `assignee` correto + campo `ID Ticket Pai` preenchido
9. Agente conversa com cliente no novo ticket
10. Agente aplica macro `[Impulsis] Encerrar Sessao`
11. Verificar: novo ticket com status `solved`
12. Verificar: Trigger 3 disparou webhook (checar execucoes N8N Workflow 2)
13. Verificar: ticket original com status `solved`
14. Verificar: campo `Retorno Ativo = false` no ticket original
15. Verificar: tag `impulsis_ativo` removida do ticket original

**Resultado esperado:** Ambos os tickets fechados, campo limpo, tags de historico preservadas.

---

### Teste 2: Cliente Responde com Texto Negativo

**Cenario:** Cliente recebe template e responde "nao tenho interesse" ou similar.

**Resultado esperado:** O IA-Follow ainda verifica `Retorno Ativo`. Se `true`, adiciona `impulsis_novo_contato` e o fluxo segue normalmente. O agente recebe o ticket e pode tratar a negativa diretamente.

**Validacao:** Mesmo uma resposta negativa deve ser roteada ao agente, pois e uma resposta valida que merece atendimento.

---

### Teste 3: Cliente Nao Responde

**Cenario:** Template enviado, cliente nao responde.

**Resultado esperado:** Nenhum ticket novo criado. Campo `Retorno Ativo` permanece `true` no ticket original. O agente pode encerrar manualmente o ticket original quando quiser.

**Validacao:** Confirmar que nenhum workflow foi acionado sem resposta do cliente.

---

### Teste 4: Protecao Anti-Duplicidade

**Cenario:** Cliente envia duas mensagens rapidamente (ex: "oi" e "oi estou aqui").

**Resultado esperado:**
- Dois tickets novos criados pelo Sunshine
- IA-Follow adiciona `impulsis_novo_contato` em ambos
- Workflow 1 processa o primeiro, adiciona `roteado_agente_automatico`
- Quando o segundo chega, o Workflow 1 detecta que ja existe ticket com `roteado_agente_automatico` para o mesmo requester e pula o roteamento

**Validacao:** Apenas um ticket novo deve ter o `assignee` correto e o campo `ID Ticket Pai` preenchido.

---

### Teste 5: Novo Ciclo Apos Encerramento

**Cenario:** Ciclo completo encerrado. Agente usa Impulsis novamente para o mesmo cliente.

**Pre-condicoes:** Workflow 2 tiver limpado `Retorno Ativo = false` e removido `impulsis_ativo`.

**Resultado esperado:** Novo ciclo funciona de forma completamente independente, sem colisao com o ciclo anterior.

**Validacao:** Campo `Retorno Ativo` no novo ticket original setado para `true` pelo Trigger 1.

---

## Troubleshooting: Problemas Comuns

### Problema: Workflow 1 nao foi acionado

**Verificar:**
1. Tag `impulsis_novo_contato` esta no ticket novo?
2. Trigger 2 esta ativo no Zendesk?
3. As condicoes do Trigger 2 estao corretas (canal = WhatsApp, tag contem, tags nao contem)?
4. O webhook esta respondendo? Testar manualmente a URL do N8N
5. O workflow N8N esta ativo (toggle on)?
6. Checar execucoes do workflow em `N8N > Executions`

**Acao rapida:** Adicionar manualmente a tag `impulsis_novo_contato` no ticket para reacionar o trigger.

---

### Problema: Ticket nao foi roteado para o agente correto

**Verificar:**
1. O ticket original tem a tag `impulsis_ativo`?
2. O ticket original tem `assignee_id` preenchido (agente que fez o disparo)?
3. O campo `Retorno Ativo` esta `true` no ticket original?
4. Na execucao do Workflow 1, o node `Valida e Extrai Ticket de Origem` retornou `found: true`?
5. O agente existe e esta ativo no Zendesk?

---

### Problema: Campo ID Ticket Pai vazio no ticket novo

**Causa provavel:** Workflow 1 falhou no node de atualizacao do ticket.

**Verificar:** Execucoes do N8N, verificar se o campo ID `41306351713940` esta correto na configuracao do workflow.

**Acao manual:** Preencher o campo `ID Ticket Pai` manualmente no ticket novo e aplicar o macro normalmente.

---

### Problema: Ticket original nao foi fechado apos macro

**Verificar:**
1. A tag `impulsis_encerrar_origem` foi adicionada ao ticket novo?
2. O campo `ID Ticket Pai` esta preenchido no ticket novo?
3. O Trigger 3 esta ativo?
4. O Workflow 2 esta ativo no N8N?
5. Checar execucoes do Workflow 2

**Acao manual:** Fechar o ticket original manualmente via interface.

---

### Problema: Campo Retorno Ativo nao foi limpo

**Verificar:** Execucao do Workflow 2 - node `Limpa Retorno Ativo`. Verificar se o ID do campo `50639967936148` esta correto no JSON do workflow.

**Impacto:** Pode causar roteamento indevido no proximo ciclo. Limpar manualmente o campo no ticket original.

---

## Como Monitorar Execucoes no N8N

1. Acesse `N8N > Executions` no menu lateral
2. Filtre pelo nome do workflow: `Impulsis - Roteamento` ou `Impulsis - Encerramento`
3. Clique em uma execucao para ver o detalhe node a node
4. Nodes vermelhos = erro; nodes verdes = sucesso; nodes cinza = nao executados
5. Clique em cada node para ver input e output exatos

**Dica:** Ative `Save successful executions` nas configuracoes do workflow para ter historico completo.


## Erros recentes corrigidos

### `users/[undefined]/group_memberships.json`

Causa: depois do no `Buscar Dados do Agente`, o item atual passa a ser a resposta do Zendesk (`user.id`). O valor `assignee_id_final` nao esta mais diretamente em `$json`.

Correcao aplicada no Workflow 1:

```js
={{ 'https://webposto.zendesk.com/api/v2/users/' + (($json.user && $json.user.id) || $('Validar Duplicidade de Filho').first().json.assignee_id_final) + '/group_memberships.json' }}
```

### Tags sobrescritas no update do ticket

Causa: usar `tags` em `PUT /api/v2/tickets/{id}.json` pode substituir tags existentes.

Correcao aplicada:

```js
additional_tags: ['impulsis_retorno', 'roteado_agente_automatico']
```

### Workflow 2 fechando origem sem validar filho

Causa: versoes antigas enviavam `PUT`/`DELETE` sem body JSON util e nao confirmavam o status do ticket filho.

Correcao aplicada:

- valida payload do webhook;
- busca ticket filho;
- so fecha origem se filho estiver `solved` ou `closed`;
- envia body JSON no fechamento e na remocao da tag `impulsis_ativo`.
