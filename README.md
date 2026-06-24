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
