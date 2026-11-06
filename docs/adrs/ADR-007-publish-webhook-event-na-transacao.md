# ADR-007: Publicação do evento via `publishWebhookEvent(tx, ...)` dentro da transação de `changeStatus`

- **Status:** Aceito
- **Data:** 2026-11-06
- **Decisores:** Bruno, Diego, Larissa
- **Fonte:** TRANSCRICAO [09:40]–[09:41]; CODIGO `src/modules/orders/order.service.ts`

## Contexto

O ponto crítico de integração é `OrderService.changeStatus` em `src/modules/orders/order.service.ts`. Hoje, dentro de `this.prisma.$transaction(async (tx) => ...)`, o método valida a transição (`canTransition`), debita/repõe estoque, faz `tx.order.update` e `tx.orderStatusHistory.create`. Para manter a garantia do [ADR-001](ADR-001-outbox-no-mysql.md), o insert na outbox precisa estar **na mesma transação** ([09:41] Diego: "se ficar fora da transação, perde a garantia toda").

## Decisão

- Criar a função **`publishWebhookEvent(tx, order, fromStatus, toStatus)`**, que recebe o **`tx` client** da transação corrente ([09:41] Bruno), e chamá-la em `changeStatus` logo após o `tx.orderStatusHistory.create`.
- A função é "pura, recebendo o tx": **não** se injeta o repository do webhook inteiro no `OrderService` ([09:41] Diego).
- Ela consulta os webhooks ativos do customer que assinam o `toStatus`, renderiza o snapshot do payload e insere uma linha na `webhook_outbox` por webhook interessado ([09:34] Bruno, [09:52] Larissa).
- **Se o insert na outbox falhar, a transação inteira faz rollback** e a mudança de status não é persistida ([09:40] Bruno).

## Alternativas Consideradas

1. **Injetar `WebhookRepository` no construtor de `OrderService`** — descartada: acoplamento maior sem ganho ([09:41] Diego).
2. **Publicar o evento após o commit (fora da transação)** — descartada: abre a janela em que o status muda e o evento não é registrado ([09:41] Diego).
3. **Ler `order_status_history` no worker e derivar eventos** — plausível (a tabela já registra toda mudança de status), não discutida na reunião; descartada por não oferecer o snapshot nem o filtro por webhook na inserção decididos no ADR-001.

## Consequências

**Positivas**
- Mudança mínima e localizada no `OrderService`; a lógica de webhooks fica no módulo `src/modules/webhooks`.
- Atomicidade total entre status, histórico, estoque e evento.

**Negativas / trade-offs**
- `OrderService` passa a depender de uma função do módulo de webhooks (acoplamento de import).
- Falha de bug em `publishWebhookEvent` passa a bloquear mudanças de status; exige testes ponta a ponta (`tests/orders.test.ts` serve de base).
- A reunião cobriu apenas `changeStatus`; a criação do pedido (`create`, status inicial `PENDING`) **não** foi mencionada e fica como questão em aberto no [RFC](../RFC.md) (RFC-OQ-05).
