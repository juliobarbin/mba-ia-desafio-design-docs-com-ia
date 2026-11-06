# ADR-004: Garantia de entrega at-least-once com X-Event-Id para deduplicação no cliente

- **Status:** Aceito
- **Data:** 2026-11-06
- **Decisores:** Diego, Larissa (Marcos e Sofia presentes)
- **Fonte:** TRANSCRICAO [09:24]–[09:26], [09:48]

## Contexto

Com outbox, retry e worker em polling, é possível que o mesmo evento seja entregue mais de uma vez (por exemplo, o cliente processa mas a resposta expira dentro do timeout de 10 s e o evento é retentado). É preciso definir a semântica de entrega prometida aos clientes.

## Decisão

- Garantir **at-least-once**: o cliente pode receber o mesmo evento mais de uma vez e deve estar preparado ([09:24] Diego).
- Cada evento recebe um **UUID gerado quando entra na outbox**, enviado no header **`X-Event-Id`**; o cliente deduplica por ele ([09:25] Diego).
- Marcos documentará isso de forma destacada no portal de desenvolvedor ([09:26] Marcos).
- Complementam a identificação os headers `X-Webhook-Id` (id do endpoint cadastrado) ([09:44] Sofia, [09:45] Diego) e `X-Timestamp`.

## Alternativas Consideradas

1. **Exactly-once** — descartada: exige coordenação dos dois lados e é muito mais complexa ([09:25] Diego).
2. **At-most-once (sem retry)** — descartada implicitamente: contradiz a política de retry/DLQ do [ADR-002](ADR-002-retry-backoff-e-dlq.md).

## Consequências

**Positivas**
- Modelo simples e comprovado no mercado ("Stripe faz assim, GitHub faz assim") ([09:25] Diego).
- "Resolve 99% dos casos" sem protocolo de confirmação bilateral.

**Negativas / trade-offs**
- **Transfere responsabilidade de idempotência ao cliente** ([09:25] Sofia); integradores mal preparados podem processar duplicado.
- Exige comunicação clara no portal de desenvolvedor, a cargo de Marcos.
- Duplicatas, junto com retries tardios, podem chegar fora de ordem em relação a eventos posteriores do mesmo pedido.
