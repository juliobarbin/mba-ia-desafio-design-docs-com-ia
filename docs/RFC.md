# RFC-001: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Autor** | Larissa (Tech Lead), rascunho elaborado com apoio de IA a partir da transcrição da reunião técnica e do código |
| **Status** | Em revisão |
| **Data** | 2026-11-06 |
| **Revisores** | Marcos (PM), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Segurança) |
| **Documentos relacionados** | [PRD](PRD.md) · [FDD](FDD.md) · [TRACKER](TRACKER.md) · ADRs 001–008 (ver seção 7) |

## 1. Resumo executivo (TL;DR)

Propomos notificar clientes B2B por **webhooks HTTP de saída** a cada mudança de status de pedido, usando o **padrão Outbox no MySQL existente**: o evento é gravado na mesma transação que muda o status e um **worker em processo separado**, em polling de 2 s, entrega com **assinatura HMAC-SHA256**, **retry com backoff** (5 tentativas) e **DLQ**. A entrega é **at-least-once**, com `X-Event-Id` para deduplicação. Todo o módulo segue os padrões já existentes no OMS. Estimativa: três sprints, incluindo revisão de segurança. Pedimos revisão especialmente nas sete questões em aberto da seção 5.

## 2. Contexto e problema

Atlas Comercial, MaxDistribuição e Nova Cargo pediram notificação em tempo real de mudança de status de pedidos; hoje consultam `GET /orders` periodicamente, o que torna a integração lenta e cara, e a Atlas sinalizou possível migração para concorrente se não entregarmos até o fim do trimestre ([09:00] Marcos). Para eles, "tempo real" é menos de 10 segundos ([09:02] Marcos). O fluxo é só de saída (nós → cliente) ([09:02] Marcos).

O OMS (Node.js + TypeScript + Prisma/MySQL) não tem filas, eventos nem webhooks. A mudança de status ocorre em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de uma transação que já é pesada (pedido, histórico, estoque) ([09:04] Bruno).

## 3. Proposta técnica (visão geral)

1. **Registro atômico.** Em `changeStatus`, dentro da mesma transação, uma função `publishWebhookEvent(tx, ...)` insere um snapshot do evento na tabela `webhook_outbox`, apenas para webhooks do cliente que assinam aquele status. Falha na outbox desfaz a mudança de status ([ADR-001](adrs/ADR-001-outbox-no-mysql.md), [ADR-007](adrs/ADR-007-publish-webhook-event-na-transacao.md)).
2. **Entrega assíncrona.** Um worker (`src/worker.ts`, `npm run worker`), processo separado, consulta a outbox a cada 2 s e envia POST assinado ao endpoint do cliente, com timeout de 10 s ([ADR-005](adrs/ADR-005-worker-separado-em-polling.md), [ADR-008](adrs/ADR-008-payload-headers-e-limites-de-entrega.md)).
3. **Resiliência.** Falhas viram retry com backoff exponencial 1m/5m/30m/2h/12h; esgotado o teto, o evento vai para a tabela `webhook_dead_letter`, reprocessável manualmente por um ADMIN ([ADR-002](adrs/ADR-002-retry-backoff-e-dlq.md)).
4. **Segurança.** HMAC-SHA256 do corpo, secret exclusiva por endpoint, rotação com grace period de 24 h, URL obrigatoriamente https ([ADR-003](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md)).
5. **Semântica de entrega.** At-least-once; o cliente deduplica por `X-Event-Id` ([ADR-004](adrs/ADR-004-at-least-once-com-x-event-id.md)).
6. **Superfície de API.** CRUD de configuração de webhooks, rotação de secret, histórico de entregas e replay de DLQ, como módulo `src/modules/webhooks` nos moldes dos demais ([ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)).

```
changeStatus ──tx──> orders + order_status_history + webhook_outbox
                                        │ (poll 2s)
                                   worker (processo separado)
                                        │ POST assinado (HMAC, timeout 10s)
                              cliente ◄─┘     falha → retry/backoff → webhook_dead_letter
```

O desenho de tabelas, contratos HTTP, erros e fluxos detalhados está no [FDD](FDD.md).

## 4. Alternativas consideradas

| # | Alternativa | Por que foi descartada (trade-off) | Origem |
|---|---|---|---|
| RFC-ALT-01 | HTTP síncrono dentro do `OrderService` | Cliente lento/offline trava a transação de status de outros pedidos; não se faz rollback da mudança de status por falha do cliente | [09:04] Bruno, [09:06] Diego |
| RFC-ALT-02 | Redis Streams ou fila equivalente | Exige infraestrutura nova; para um time pequeno é overengineering, versus reaproveitar o MySQL | [09:07] Larissa, [09:07] Diego |
| RFC-ALT-03 | Trigger de banco para acordar o worker | MySQL não tem LISTEN/NOTIFY; a trigger só executa SQL; polling de 2 s atende o requisito de <10 s | [09:09] Diego |
| RFC-ALT-04 | 3 tentativas de retry, ou retry indefinido | 3 é pouco para indisponibilidades de horas; indefinido deixa evento pendurado para sempre | [09:15] Diego, [09:16] Diego |
| RFC-ALT-05 | Exactly-once | Exige coordenação dos dois lados e é bem mais complexo que at-least-once com `X-Event-Id` | [09:25] Diego |
| RFC-ALT-06 | Secret global da plataforma | Vazamento de uma secret expõe todos os clientes | [09:21] Sofia |

## 5. Questões em aberto

| # | Questão | Situação na reunião | Origem |
|---|---|---|---|
| RFC-OQ-01 | **Rate limiting de saída** por cliente (ex.: 50 pedidos mudando em 1 min geram 50 chamadas) | Adiado: "observar e decidir depois" | [09:38] Diego, [09:39] Larissa |
| RFC-OQ-02 | **Proteção da secret em repouso** (a plataforma precisa recuperá-la para assinar) | Não discutido; sugerimos levar à revisão de segurança de Sofia | [09:46] Sofia (revisão do código de segurança), CODIGO `prisma/schema.prisma` |
| RFC-OQ-03 | **Contagem das tentativas**: "5 tentativas" e 5 intervalos; o total de ~15 h só fecha com tentativa inicial + 5 retries (6 envios) | Ambíguo na fala; confirmar | [09:15] Diego, [09:17] Diego |
| RFC-OQ-04 | **Assinatura durante o grace period de 24 h**: assinar com a secret nova, a antiga ou enviar as duas? | Decidida a existência do grace; o mecanismo não | [09:21] Sofia, [09:22] Sofia |
| RFC-OQ-05 | **Evento na criação do pedido** (status inicial `PENDING`): `create` emite evento? | Reunião tratou só `changeStatus` | [09:40] Bruno, CODIGO `src/modules/orders/order.service.ts` |
| RFC-OQ-06 | **Escalar para múltiplos workers** (particionar por `order_id` ou lock pessimista) | Adiado: "problema do futuro" | [09:13] Diego |
| RFC-OQ-07 | **Endurecer a autorização do CRUD** (hoje qualquer role autenticada) | "Por enquanto sim; mais pra frente pode endurecer" | [09:37] Sofia |

Também adiados e fora do escopo: e-mail de alerta de falha ([09:37] Larissa), painel visual ([09:40] Larissa) e arquivamento de linhas entregues após ~30 dias ([09:08] Diego).

## 6. Impacto e riscos

**Impacto no sistema existente.** Uma alteração em `OrderService.changeStatus`; novos arquivos no módulo `src/modules/webhooks`; nova entry point `src/worker.ts`; novas tabelas Prisma (`webhook_outbox`, `webhook_dead_letter` e de configuração) via migration; registro do módulo em `src/app.ts` e `src/routes/index.ts`. Sem alteração de contrato das APIs atuais.

**Riscos principais** (detalhe e probabilidade/impacto no [PRD](PRD.md)):
- Falha na outbox bloqueia mudança de status (aceito por design, [ADR-007](adrs/ADR-007-publish-webhook-event-na-transacao.md)).
- Crescimento da outbox sem arquivamento nesta fase.
- Head-of-line blocking com worker único e cliente lento (ADR-005/ADR-008).
- Vazamento de secret, inclusive via log (o logger atual não redige `secret`).
- Prazo: Atlas quer até o fim de novembro e a estimativa é de três sprints ([09:45] Marcos, [09:46] Larissa).
- Duplicatas e fora-de-ordem exigem cliente preparado ([ADR-004](adrs/ADR-004-at-least-once-com-x-event-id.md)).

## 7. Decisões relacionadas

- [ADR-001 — Outbox no MySQL](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Retry, backoff e DLQ](adrs/ADR-002-retry-backoff-e-dlq.md)
- [ADR-003 — HMAC-SHA256 e secret por endpoint](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md)
- [ADR-004 — At-least-once e X-Event-Id](adrs/ADR-004-at-least-once-com-x-event-id.md)
- [ADR-005 — Worker separado em polling](adrs/ADR-005-worker-separado-em-polling.md)
- [ADR-006 — Reuso dos padrões do projeto](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)
- [ADR-007 — `publishWebhookEvent` na transação](adrs/ADR-007-publish-webhook-event-na-transacao.md)
- [ADR-008 — Payload, headers e limites](adrs/ADR-008-payload-headers-e-limites-de-entrega.md)
