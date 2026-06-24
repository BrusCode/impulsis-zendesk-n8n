# 06 — Checklist de Implantacao

Use este checklist para garantir que todos os componentes estao configurados antes de ativar o fluxo em producao.

---

## Fase 1: Zendesk — Campos e Webhooks

- [ ] Campo **Retorno Ativo** (ID: `50639967936148`) existe e e do tipo Checkbox
- [ ] Campo **ID Ticket Pai** (ID: `41306351713940`) existe e e do tipo Texto
- [ ] Campo **Ticket Pai** (ID: `41270881369364`) existe e e do tipo Checkbox
- [ ] Campo **Ticket Filho** (ID: `41270897374356`) existe e e do tipo Checkbox
- [ ] Webhook **N8N - Roteamento Impulsis** criado e testado (status 200)
- [ ] Webhook **N8N - Encerramento Impulsis** criado e testado (status 200)

---

## Fase 2: Zendesk — Triggers

- [ ] **Trigger 1** `[Impulsis] SET Campo Retorno Ativo` criado e ATIVO
  - [ ] Condicao: canal = WhatsApp
  - [ ] Condicao: tag contem `impulsis_ativo`
  - [ ] Condicao: tag nao contem `roteado_agente_automatico`
  - [ ] Acao: seta campo Retorno Ativo = true

- [ ] **Trigger 2** `[Impulsis] Webhook Roteamento - Novo Contato` criado e ATIVO
  - [ ] Condicao: ticket criado = true
  - [ ] Condicao: canal = WhatsApp
  - [ ] Condicao: tag contem `impulsis_novo_contato`
  - [ ] Condicao: tag nao contem `impulsis_ativo`
  - [ ] Condicao: tag nao contem `roteado_agente_automatico`
  - [ ] Acao: notifica webhook Roteamento com payload correto

- [ ] **Trigger 3** `[Impulsis] Webhook Encerramento - Fechar Origem` criado e ATIVO
  - [ ] Condicao: status = solved
  - [ ] Condicao: tag contem `impulsis_encerrar_origem`
  - [ ] Condicao: campo ID Ticket Pai nao vazio
  - [ ] Acao: notifica webhook Encerramento com payload correto

---

## Fase 3: Zendesk — Macro

- [ ] Macro `[Impulsis] Encerrar Sessao - Fechar Retorno + Origem` criado
  - [ ] Acao: status = solved
  - [ ] Acao: adiciona tag `impulsis_encerrar_origem`
  - [ ] Acao: nota interna com placeholder `{{ticket.ticket_field_41306351713940}}`
- [ ] Macro adicionado ao workspace de agentes WhatsApp (opcional mas recomendado)

---

## Fase 4: N8N — Credenciais

- [ ] Credencial `ZENDESK_BASIC_AUTH` cadastrada no N8N
  - [ ] Username no formato `email@dominio.com/token`
  - [ ] Password = API Token gerado no Zendesk
- [ ] Credencial testada com uma chamada manual de GET /api/v2/users/me.json

---

## Fase 5: N8N — Workflow 1 (Roteamento)

- [ ] Workflow importado do arquivo `workflows/workflow1-roteamento.json`
- [ ] Credencial `ZENDESK_BASIC_AUTH` substituida em todos os nodes HTTP
- [ ] Path do webhook anotado e configurado no Webhook do Zendesk
- [ ] Workflow ATIVO (toggle on)
- [ ] Teste manual: enviar POST ao webhook com payload simulado
- [ ] Verificar execucao em N8N > Executions

---

## Fase 6: N8N — Workflow 2 (Encerramento)

- [ ] Workflow importado do arquivo `workflows/workflow2-encerramento.json`
- [ ] Credencial `ZENDESK_BASIC_AUTH` substituida em todos os nodes HTTP
- [ ] Path do webhook anotado e configurado no Webhook do Zendesk
- [ ] Workflow ATIVO (toggle on)
- [ ] Teste manual: enviar POST ao webhook com payload simulado
- [ ] Verificar execucao em N8N > Executions

---

## Fase 7: IA-Follow — Welcome Reply

- [ ] Logica de verificacao de `Retorno Ativo` implementada no Welcome Reply
  - [ ] Opcao A: HTTP Step nativo configurado com query correta
  - [ ] Opcao B: Webhook N8N intermediario configurado
- [ ] Comportamento testado: ticket com `Retorno Ativo = true` recebe tag `impulsis_novo_contato`
- [ ] Comportamento testado: ticket sem `Retorno Ativo` nao recebe a tag
- [ ] Welcome Reply nao envia mensagem ao cliente quando `impulsis_novo_contato` e adicionada

---

## Fase 8: Testes Ponta a Ponta

- [ ] Teste 1: Fluxo completo feliz executado com sucesso
- [ ] Teste 2: Cliente responde com texto negativo — roteamento correto
- [ ] Teste 3: Cliente nao responde — nenhum workflow acionado
- [ ] Teste 4: Protecao anti-duplicidade validada
- [ ] Teste 5: Novo ciclo apos encerramento funciona sem colisao

---

## Fase 9: Go-Live

- [ ] Todos os itens acima marcados
- [ ] Equipe de agentes treinada no uso do macro
- [ ] Agentes sabem identificar o campo `ID Ticket Pai` no ticket novo
- [ ] Agentes sabem encerrar manualmente em caso de falha do workflow
- [ ] Monitoramento de execucoes N8N ativado para os primeiros dias
- [ ] Data de go-live registrada no versionamento deste repositorio

---

## Valores a Substituir nos Workflows

| Variavel | Valor Real | Arquivo |
|---|---|---|
| `SEU_N8N` | URL do seu N8N (ex: n8n.webposto.com.br) | Webhooks Zendesk |
| `ZENDESK_BASIC_AUTH` | Nome da credencial cadastrada no N8N | Ambos workflows |
| `webposto.zendesk.com` | Seu subdominio Zendesk | Ambos workflows |
| `50639967936148` | ID campo Retorno Ativo | Workflow 1 e 2 |
| `41306351713940` | ID campo ID Ticket Pai | Workflow 1 |
| `41270881369364` | ID campo Ticket Pai | Workflow 1 |
| `41270897374356` | ID campo Ticket Filho | Workflow 1 |
