# 01 — Arquitetura e Logica do Fluxo

## Visao Geral

O fluxo Impulsis resolve um problema especifico do Zendesk com WhatsApp via Sunshine Conversations: quando a sessao de 24h expira, nao e possivel continuar a conversa no mesmo ticket. O Impulsis envia um template ativo (HSM) para reabrir a comunicacao, mas a resposta do cliente cria um NOVO ticket sem vinculo algum com o ticket original.

Este fluxo automatiza o roteamento desse novo ticket diretamente para o agente que realizou o disparo, preservando o contexto e eliminando a necessidade de configuracao manual no construtor de dialogo.

---

## Os Dois Tickets

| # | Nome | Papel | O que acontece |
|---|---|---|---|
| **Ticket Original** | Ticket do disparo | Contexto historico, contem `impulsis_ativo` | Preservado intacto; fechado pelo agente ao fim da sessao via Workflow 2 |
| **Ticket Novo** | Ticket de retorno | Canal ativo de conversa com o cliente | Roteado automaticamente pelo Workflow 1 para o agente correto |

**Por que nao fazer merge?**
O merge nativo do Zendesk redirecionaria o canal do ticket filho para o pai. O ticket original esta com sessao encerrada — o merge jogaria a conversa de volta para um canal morto. Inviavel.

**Por que nao usar pai/filho nativo?**
O `problem_id` do Zendesk so funciona quando o ticket pai e do tipo `Problem`. Nao se aplica ao caso de uso. O vinculo e feito via campo customizado `ID Ticket Pai`.

---

## Campos Customizados de Vinculo

| Campo | ID | Tipo | Preenchido em |
|---|---|---|---|
| Retorno Ativo | `50639967936148` | Checkbox | Ticket original, pela Trigger `SET Retorno Ativo` |
| ID Ticket Pai | `41306351713940` | Texto | Ticket novo, pelo Workflow 1 |
| Ticket Pai | `41270881369364` | Checkbox | Ticket original, pelo Workflow 1 |
| Ticket Filho | `41270897374356` | Checkbox | Ticket novo, pelo Workflow 1 |

---

## Cadeia Completa de Eventos

```
1. AGENTE usa o Impulsis para disparar template WhatsApp ao cliente
   Resultado: ticket original recebe tags impulsis_ativo + sessao_encerrada_agente

2. TRIGGER ZENDESK [SET Retorno Ativo] detecta:
   - Canal = WhatsApp
   - Tag: impulsis_ativo
   - Tag NAO contem: roteado_agente_automatico
   Acao: seta campo Retorno Ativo (50639967936148) = true no ticket original

3. CLIENTE responde ao template (sim / nao / qualquer texto)
   Resultado: Sunshine Conversations cria um NOVO ticket automaticamente
   Este ticket nao tem tags, nao tem vinculo com o original

4. BOT IA-Follow recebe o novo ticket no Welcome Reply
   Verifica: existe ticket do mesmo requester com Retorno Ativo = true?
   - SIM: adiciona tag impulsis_novo_contato no novo ticket
   - NAO: segue fluxo normal do bot

5. TRIGGER ZENDESK [Webhook Roteamento Impulsis] detecta:
   - Ticket criado
   - Canal = WhatsApp
   - Tag contem: impulsis_novo_contato
   - Tag NAO contem: impulsis_ativo
   - Tag NAO contem: roteado_agente_automatico
   Acao: dispara webhook para N8N Workflow 1

6. N8N WORKFLOW 1 executa:
   a. Busca tickets do mesmo requester com tag impulsis_ativo (excluindo o novo)
   b. Verifica se ja existe ticket filho ativo (protecao anti-duplicidade)
   c. Recupera dados do agente assignee do ticket original
   d. Atualiza o novo ticket:
      - assignee_id = agente do ticket original
      - status = open
      - tags adiciona: impulsis_retorno, roteado_agente_automatico
      - campo ID Ticket Pai = ID do ticket original
      - campo Ticket Filho = true
      - nota interna com contexto completo
   e. Atualiza o ticket original:
      - campo Ticket Pai = true
      - nota interna confirmando roteamento

7. AGENTE recebe o novo ticket ja roteado para ele
   Conversa com o cliente sobre o assunto do ticket original
   Tem visibilidade do ticket de origem via campo ID Ticket Pai

8. AGENTE aplica o Macro [Encerrar Sessao Impulsis] ao concluir
   Acao do macro:
   - status = solved
   - adiciona tag: impulsis_encerrar_origem
   - nota interna com referencia ao ticket original

9. TRIGGER ZENDESK [Webhook Encerramento Impulsis] detecta:
   - Status = solved
   - Tag contem: impulsis_encerrar_origem
   - Campo ID Ticket Pai nao vazio
   Acao: dispara webhook para N8N Workflow 2

10. N8N WORKFLOW 2 executa:
    a. Valida ID do ticket original
    b. Fecha ticket original (status = solved)
    c. Limpa campo Retorno Ativo (false) no ticket original
    d. Remove tag impulsis_ativo do ticket original
    e. Adiciona nota de encerramento em ambos os tickets
```

---

## Decisoes de Design

### Por que usar campo Retorno Ativo como semaforo?
Usar um campo checkbox em vez de apenas tags e mais confiavel porque o campo tem estado binario explicito (true/false). Tags podem acumular de ciclos anteriores e causar falso positivo. O campo e limpo no Workflow 2, garantindo que um novo ciclo de disparo para o mesmo cliente funcione de forma independente.

### Por que o IA-Follow adiciona a tag e nao o Trigger diretamente?
O Trigger do Zendesk nao tem acesso ao estado de custom fields de outros tickets do mesmo requester. O IA-Follow (construtor de dialogo) pode fazer essa consulta e adicionar a tag semaforo `impulsis_novo_contato` somente quando ha de fato um ticket original valido. Isso evita que qualquer mensagem de WhatsApp de um cliente que ja usou o Impulsis no passado acione o roteamento indevidamente.

### Por que manter o ticket original intacto?
O ticket original e o registro historico do disparo. Alterar seu status ou conteudo durante o processo de retorno poderia confundir metricas, SLAs e auditorias. Ele e encerrado apenas no final, pelo Workflow 2, com nota explicativa.

### Protecao contra duplicidade
Se o cliente enviar duas mensagens rapidamente antes do roteamento ser concluido, dois tickets podem receber `impulsis_novo_contato`. O Workflow 1 verifica se ja existe um ticket recente do mesmo requester com a tag `roteado_agente_automatico` antes de prosseguir, evitando roteamento duplo.

---

## Limpeza ao Encerrar (Workflow 2)

| Campo/Tag | Operacao | Motivo |
|---|---|---|
| `Retorno Ativo` (50639967936148) | `false` | Evita reentrada indevida em novo disparo futuro |
| `impulsis_ativo` | Removida | Evita falso positivo no IA-Follow em novo ciclo |
| `roteado_agente_automatico` | Mantida | Auditoria e trava historica |
| `ID Ticket Pai` (41306351713940) | Mantido | Rastreabilidade historica |
| `impulsis_retorno` no ticket filho | Mantida | Dado analitico do ciclo |
