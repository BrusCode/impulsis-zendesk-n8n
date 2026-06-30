# 02 — Configuracao Zendesk

## Webhooks Ativos

Crie em `Admin > Apps and Integrations > Webhooks > Add Webhook`.

### Webhook 1: Roteamento Impulsis
- **Nome:** `N8N - Roteamento Impulsis`
- **URL:** `https://SEU_N8N/webhook/impulsis-retorno-routing`
- **Metodo:** POST
- **Autenticacao:** Bearer Token (token gerado no N8N)

### Webhook 2: Encerramento Impulsis
- **Nome:** `N8N - Encerramento Impulsis`
- **URL:** `https://SEU_N8N/webhook/impulsis-close-origin`
- **Metodo:** POST
- **Autenticacao:** Bearer Token

---

## Triggers

### Trigger 1: SET Retorno Ativo

Caminho: `Admin > Objects and Rules > Triggers > Add Trigger`

**Nome:** `[Impulsis] SET Campo Retorno Ativo`

**Condicoes (ALL):**
| Campo | Operador | Valor |
|---|---|---|
| Ticket: Updated via | Is | Any channel |
| Ticket: Tags | Contains at least one of | `impulsis_ativo` |
| Ticket: Canal categoria | Is | WhatsApp |
| Ticket: Tags | Does not contain | `roteado_agente_automatico` |

**Acoes:**
| Acao | Campo | Valor |
|---|---|---|
| Ticket: Retorno Ativo (50639967936148) | Checked | true |

> Este trigger seta o campo checkbox que o IA-Follow vai verificar no Welcome Reply do novo ticket.

---

### Trigger 2: Webhook Roteamento Impulsis

**Nome:** `[Impulsis] Webhook Roteamento - Novo Contato`

**Condicoes (ALL):**
| Campo | Operador | Valor |
|---|---|---|
| Ticket: Created | Is | True |
| Ticket: Canal categoria | Is | WhatsApp |
| Ticket: Tags | Contains at least one of | `impulsis_novo_contato` |
| Ticket: Tags | Does not contain | `impulsis_ativo` |
| Ticket: Tags | Does not contain | `roteado_agente_automatico` |

**Acao:** `Notify by active webhook > N8N - Roteamento Impulsis`

**Payload:**
```json
{
  "ticket_id": "{{ticket.id}}",
  "requester_id": "{{ticket.requester.id}}",
  "requester_name": "{{ticket.requester.name}}",
  "requester_phone": "{{ticket.requester.phone}}",
  "ticket_status": "{{ticket.status}}",
  "trigger": "impulsis_novo_contato"
}
```

---

### Trigger 3: Webhook Encerramento Impulsis

**Nome:** `[Impulsis] Webhook Encerramento - Fechar Origem`

**Condicoes (ALL):**
| Campo | Operador | Valor |
|---|---|---|
| Ticket: Status | Is | Solved |
| Ticket: Tags | Contains at least one of | `impulsis_encerrar_origem` |
| Ticket: Tags | Contains none of | `impulsis_falha_fechamento_origem` |
| Ticket: Campo ID Ticket Pai (41306351713940) | Is not | (vazio) |

**Acao:** `Notify by active webhook > N8N - Encerramento Impulsis`

> A condição `Contains none of` bloqueia loop quando o Workflow 2 não consegue fechar a origem e adiciona a tag de falha no ticket filho.

**Payload:**
```json
{
  "action": "close_origin",
  "child_ticket_id": "{{ticket.id}}",
  "origin_ticket_id": "{{ticket.ticket_field_41306351713940}}",
  "agent_name": "{{ticket.assignee.name}}",
  "agent_email": "{{ticket.assignee.email}}"
}
```

---

## Campos Customizados

Consulte em `Admin > Objects > Ticket Fields`.

| Campo | ID | Tipo | Onde usar |
|---|---|---|---|
| Retorno Ativo | `50639967936148` | Checkbox | Ticket original — setado pelo Trigger 1 |
| ID Ticket Pai | `41306351713940` | Texto | Ticket novo — preenchido pelo Workflow 1 |
| Ticket Pai | `41270881369364` | Checkbox | Ticket original — marcado pelo Workflow 1 |
| Ticket Filho | `41270897374356` | Checkbox | Ticket novo — marcado pelo Workflow 1 |

**Uso em placeholders (triggers/macros):**
```
{{ticket.ticket_field_50639967936148}}  -> Retorno Ativo
{{ticket.ticket_field_41306351713940}}  -> ID Ticket Pai
{{ticket.ticket_field_41270881369364}}  -> Ticket Pai
{{ticket.ticket_field_41270897374356}}  -> Ticket Filho
```

---

## Credenciais API Zendesk para o N8N

Crie em `Admin > Apps and Integrations > Zendesk API > Add API Token`.

No N8N, configure a credencial `httpBasicAuth`:
- **Username:** `seu@email.com/token`
- **Password:** `SEU_API_TOKEN_ZENDESK`

Formate o username exatamente como `email/token` para autenticacao por token (nao senha).
