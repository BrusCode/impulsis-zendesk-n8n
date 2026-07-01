# Impulsis + Zendesk + N8N — Documentacao Operacional

> Documentacao completa do fluxo de disparo ativo WhatsApp via Impulsis, roteamento automatico de tickets via N8N e encerramento sincronizado no Zendesk (WebPosto).

## Estrutura do Repositorio

```
impulsis-zendesk-n8n/
├── README.md
├── docs/
│   ├── 01-arquitetura-e-logica.md
│   ├── 02-configuracao-zendesk.md
│   ├── 03-ia-follow-welcome.md
│   ├── 04-macro-encerramento.md
│   ├── 05-testes-e-troubleshooting.md
│   └── 06-checklist-implantacao.md
└── workflows/
    ├── workflow1-roteamento.json
    └── workflow2-encerramento.json
```

## Campos Zendesk Utilizados

| Campo | ID | Tipo | Uso |
|---|---|---|---|
| Retorno Ativo | `50639967936148` | Checkbox | Sinaliza disparo ativo em andamento |
| ID Ticket Pai | `41306351713940` | Texto | ID do ticket original gravado no ticket novo |
| Ticket Pai | `41270881369364` | Checkbox | Referencia visual no ticket original |
| Ticket Filho | `41270897374356` | Checkbox | Referencia visual no ticket novo |

## Tags Utilizadas

| Tag | Onde | Significado |
|---|---|---|
| `impulsis_ativo` | Ticket original | Disparo realizado, sessao encerrada |
| `sessao_encerrada_agente` | Ticket original | Sessao WhatsApp encerrada |
| `impulsis_novo_contato` | Ticket novo | Semaforo: cliente respondeu |
| `roteado_agente_automatico` | Ticket novo | Trava anti-reprocessamento |
| `impulsis_encerrar_origem` | Ticket novo | Gatilho para fechar ticket original |
| `impulsis_falha_fechamento_origem` | Ticket novo | Trava anti-loop quando a origem não fecha por pendências |
| `impulsis_pendencia_fechamento` | Ticket original | Indica que há campos obrigatórios pendentes para fechamento |
| `impulsis_origem_fechada` | Ticket original | Auditoria: origem fechada pelo Workflow 2 |
| `erro_roteamento_agente` | Ticket novo | Falha controlada: agente/grupo válido não encontrado |

## Workflows N8N

| Arquivo | Funcao |
|---|---|
| workflow1-roteamento.json | Roteia novo ticket ao agente do disparo original |
| workflow2-encerramento.json | Fecha ticket original, limpa Retorno Ativo e tags |

**Como importar:** Copie o JSON > N8N > Workflows > Import from clipboard. Substitua `ZENDESK_BASIC_AUTH` pela sua credencial.

## Ambiente

- Zendesk: webposto.zendesk.com
- Canal: WhatsApp via Sunshine Conversations
- Automacao: N8N self-hosted
- App de Disparo: Impulsis
- Bot: IA-Follow

## Versionamento

| Versao | Data | Alteracao |
|---|---|---|
| v1.0 | 2026-06-24 | Criacao inicial — fluxo completo validado |

> Mantido por BrusCode / WebPosto
| v2.0 | 2026-06-24 | Workflow 2 de encerramento + macro + limpeza de campos |
| v3.0 | 2026-06-25 | **Assignee via comentário Impulsis** - Identifica agente que enviou mensagem ativa via author_id do comentário privado (fallback: assignee original) |

> **Novidade v3.0:** O Workflow 1 agora busca comentários privados do Impulsis no ticket original para identificar o agente que realmente enviou a mensagem ativa (`author_id`), resolvendo casos onde o agente que dispara é diferente do assignee do ticket original.

## Tutorial Completo

Para instruções detalhadas de implementação, acesse o [Tutorial HTML](tutorial.html) que contém:
- Guia passo a passo de instalação
- Explicação detalhada dos workflows v3.0
- Configuração completa do Zendesk e N8N
- Checklist de validação e troubleshooting


## Atualizacao 2026-06-26

Os workflows atuais substituem a primeira versao operacional por uma logica mais segura:

- `workflow1-roteamento.json`: roteamento seguro do novo ticket para o agente que iniciou o contato Impulsis.
  - Identifica o agente pelo comentario privado mais recente do Impulsis.
  - Busca os grupos ativos do agente no Zendesk.
  - Atualiza o novo ticket com `group_id` + `assignee_id`, evitando redistribuicao automatica para outro agente.
  - Aplica tags por nós dedicados da Tags API (`PUT /api/v2/tickets/{id}/tags.json`), sem depender de `additional_tags` no update do ticket.
  - Define assunto padrão `Retorno ticket #<origem>: preencher assunto` para forçar revisão humana antes do encerramento.
  - Se nao encontrar agente/grupo valido, marca erro controlado no ticket novo e nao usa o grupo do ticket original como fallback.

- `workflow2-encerramento.json`: encerramento seguro do ticket original.
  - Valida payload recebido pelo webhook.
  - Busca o ticket filho antes de fechar a origem.
  - Fecha a origem apenas se o filho estiver `solved` ou `closed`.
  - Limpa o campo de controle e remove `impulsis_ativo` com body JSON valido.
  - Aplica tags de auditoria/falha por Tags API dedicada.
  - Responde ao webhook somente ao final do fluxo.

> O arquivo `tutorial.html` ainda descreve a documentacao inicial e sera revisado em uma etapa posterior.
