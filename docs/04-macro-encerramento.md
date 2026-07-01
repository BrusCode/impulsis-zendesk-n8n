# 04 — Macro de Encerramento Duplo

## Objetivo

O macro permite que o agente encerre os dois tickets do ciclo Impulsis com uma unica acao:
- **Ticket novo (retorno):** fechado diretamente pelo macro
- **Ticket original (disparo):** fechado automaticamente pelo Workflow 2, acionado pelo Trigger que detecta a tag adicionada pelo macro

---

## Configuracao do Macro

Caminho: `Admin > Workspaces > Macros > Add Macro`

**Nome:** `[Impulsis] Encerrar Sessao - Fechar Retorno + Origem`

**Descricao:** `Encerra a sessao de retorno Impulsis. Fecha este ticket e aciona o encerramento automatico do ticket de origem via N8N.`

### Acoes do Macro

| Acao | Campo | Valor |
|---|---|---|
| Ticket: Status | — | Solved |
| Ticket: Add tags | — | `impulsis_encerrar_origem` |
| Ticket: Remove tags | — | `impulsis_falha_fechamento_origem` |
| Ticket: Add comment | Interno | Ver template abaixo |

### Template do Comentario Interno

```
Sessao Impulsis encerrada pelo agente.

Ticket de origem: #{{ticket.ticket_field_41306351713940}}
O ticket de origem sera fechado automaticamente.

Resumo da sessao: [agente preenche manualmente se necessario]
```

> O placeholder `{{ticket.ticket_field_41306351713940}}` exibe o ID do ticket original, confirmando visualmente ao agente qual ticket sera encerrado.

---

## Fluxo Apos Aplicar o Macro

```
[Agente aplica macro no ticket novo]
         |
         v
[Ticket novo: status = solved]
[Ticket novo: tag impulsis_encerrar_origem adicionada]
[Ticket novo: nota interna criada]
         |
         v
[Trigger: Webhook Encerramento Impulsis detecta]
  - status = solved
  - tag: impulsis_encerrar_origem
  - tag NAO contem: impulsis_falha_fechamento_origem
  - campo ID Ticket Pai preenchido
         |
         v
[N8N Workflow 2 executa:]
  1. Valida ID do ticket original
  2. Fecha ticket original (status = solved)
  3. Limpa campo Retorno Ativo (false)
  4. Adiciona tag impulsis_origem_fechada via Tags API
  5. Remove tag impulsis_ativo
  6. Adiciona nota de encerramento
```

---

## Visibilidade do Macro para os Agentes

Para facilitar o acesso, adicione o macro como acao rapida na view de tickets WhatsApp:

1. `Admin > Workspaces > Agent Workspaces`
2. Selecione o workspace de WhatsApp
3. Em `Macros`, adicione `[Impulsis] Encerrar Sessao`

O agente vera o macro disponivel no painel lateral do ticket sem precisar buscar.

---

## Comportamento de Limpeza (Workflow 2)

Apos o macro ser aplicado e o Workflow 2 executar:

| Ticket | Campo/Tag | Estado Final |
|---|---|---|
| Ticket novo | Status | solved |
| Ticket novo | Tag `impulsis_encerrar_origem` | mantida (historico) |
| Ticket novo | Tag `roteado_agente_automatico` | mantida (historico) |
| Ticket original | Status | solved |
| Ticket original | Campo Retorno Ativo | false (limpo) |
| Ticket original | Tag `impulsis_ativo` | removida |
| Ticket original | Campo Ticket Pai | true (mantido para rastreabilidade) |

---

## Aviso: Macro Nao Funciona Sem o Campo ID Ticket Pai

Se o Workflow 1 nao tiver preenchido o campo `ID Ticket Pai` no ticket novo (por alguma falha no roteamento), o Trigger 3 nao vai disparar porque sua condicao verifica que o campo nao esteja vazio.

Nesse caso, o agente pode:
1. Preencher manualmente o campo `ID Ticket Pai` com o ID correto
2. Aplicar o macro novamente

Ou encerrar o ticket original manualmente.


## Reprocessamento apos falha de fechamento

Se o Workflow 2 não conseguir fechar o ticket original por campos obrigatórios pendentes, ele adiciona `impulsis_falha_fechamento_origem` no ticket filho. Essa tag bloqueia o gatilho para evitar loop.

Depois que os campos obrigatórios forem preenchidos no ticket original, o agente deve aplicar novamente este macro no ticket filho. O macro remove `impulsis_falha_fechamento_origem`, mantém/adiciona `impulsis_encerrar_origem` e permite uma nova tentativa de fechamento.

Importante: a tag `impulsis_falha_fechamento_origem` deve estar como condição negativa no Trigger 3. Sem essa condição, comentários internos gerados pelo próprio Workflow 2 podem reacionar o webhook em loop.
