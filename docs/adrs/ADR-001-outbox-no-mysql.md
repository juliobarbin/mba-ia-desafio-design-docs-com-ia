# ADR-001: Padrão Outbox no MySQL existente para registrar eventos de webhook

- **Status:** Aceito
- **Data:** 2026-11-06 (decisão tomada na reunião técnica de quinta-feira, ver [09:08] Larissa)
- **Decisores:** Larissa, Diego, Bruno (Marcos e Sofia presentes)
- **Fonte:** TRANSCRICAO [09:03]–[09:08], [09:51], [09:52]

## Contexto

Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) querem ser notificados quando o status de um pedido muda ([09:00] Marcos). Hoje o método `changeStatus` em `src/modules/orders/order.service.ts` já executa, em uma única `$transaction`, o `update` do pedido, o insert em `order_status_history` e o débito/reposição de estoque. Disparar HTTP dentro dessa transação travaria mudanças de status de outros pedidos quando um cliente fosse lento, e não há como "dar rollback" da mudança de status se o cliente estiver fora do ar ([09:04] Bruno). Não existe hoje nenhum mecanismo de fila, evento ou webhook no código.

## Decisão

Adotar o **padrão Outbox no MySQL existente**: dentro da mesma transação SQL que atualiza `orders` e `order_status_history`, inserir uma linha na tabela `webhook_outbox` com o evento. Um worker separado (ver [ADR-005](ADR-005-worker-separado-em-polling.md)) lê essa tabela e faz as chamadas HTTP. Detalhes fechados na reunião:

- A tabela tem índice no campo de status (`pendente`, `processando`, `falhou`, `entregue`) e em `created_at`; o worker lê só os pendentes, em batch pequeno ([09:08] Diego).
- Chave primária em **UUID**, seguindo o padrão do projeto (`@id @default(uuid()) @db.Char(36)` em `prisma/schema.prisma`) ([09:51] Larissa).
- O evento guarda o **payload já renderizado (snapshot) no momento da inserção** ([09:52] Larissa, [09:52] Diego, [09:52] Bruno).
- O filtro de eventos por webhook acontece **na inserção**: se nenhum webhook do customer quer aquele status, nenhuma linha é inserida ([09:34] Bruno, [09:34] Diego).
- Arquivamento de linhas entregues (cerca de 30 dias) fica **fora do escopo** desta feature ([09:08] Diego).

## Alternativas Consideradas

1. **Disparo HTTP síncrono no `OrderService`** — descartada: um cliente lento ou offline trava/derruba a transação de status ([09:04] Bruno, [09:06] Diego "fora de questão").
2. **Redis Streams ou fila equivalente** — descartada: exigiria subir infraestrutura nova para um time pequeno; "Redis Cluster pra isso é overengineering" ([09:07] Larissa, [09:07] Diego).
3. **Trigger de banco para acionar o worker** — descartada: MySQL não tem LISTEN/NOTIFY e a trigger só executa SQL, não notifica processo externo ([09:09] Diego). Esse ponto é tratado em [ADR-005](ADR-005-worker-separado-em-polling.md).
4. **Gravar só `order_id` na outbox e renderizar o payload no envio** — descartada: se o pedido mudar depois, o evento deixaria de refletir o estado do momento da mudança ([09:52] Larissa).

## Consequências

**Positivas**
- Atomicidade: se a transação de `changeStatus` commita, o evento existe; se dá rollback, o evento some junto ([09:06] Diego). Não há caso de status mudar sem evento.
- Zero infraestrutura nova: reaproveita o MySQL e o Prisma já usados.
- Snapshot torna o evento imutável e auditável.

**Negativas / trade-offs**
- A transação de `changeStatus` fica ligeiramente mais pesada (insert extra por webhook interessado) e, se o insert na outbox falhar, a **mudança de status é revertida** ([09:40] Bruno). Aceito: preferimos consistência a disponibilidade da mudança de status.
- A tabela cresce sem política de arquivamento nesta fase; o volume deve ser acompanhado.
- Latência mínima passa a depender do intervalo de polling (ver ADR-005).
- O MySQL passa a ser também a "fila", acoplando carga de entrega e carga transacional do OMS.
