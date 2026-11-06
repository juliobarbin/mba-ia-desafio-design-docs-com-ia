# ADR-008: Formato do payload, headers e limites de entrega (timeout 10 s, 64 KB)

- **Status:** Aceito
- **Data:** 2026-11-06
- **Decisores:** Diego, Sofia, Larissa, Marcos
- **Fonte:** TRANSCRICAO [09:23]–[09:24], [09:42]–[09:45]

## Contexto

Precisamos fixar o contrato de saída que os clientes vão integrar. Cobre decisões secundárias da reunião: payload enxuto, conjunto de headers, timeout da chamada e limite de tamanho.

## Decisão

- **Payload JSON** com: `event_id`, `event_type` (ex.: `order.status_changed`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos da order como `total_cents`. **Sem `items`**; quem precisar de detalhes consulta `GET /orders/:id` ([09:43] Diego, [09:44] Bruno).
- **Headers**: `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type: application/json` ([09:44] Diego, [09:44] Sofia, [09:45] Diego).
- **Timeout HTTP de 10 segundos**; excedido, é falha e vai para retry ([09:42] Diego).
- **Limite de payload de 64 KB**: acima disso o evento **não é enviado** (erro, sem truncar) ([09:23] Sofia, [09:24] Diego, [09:24] Larissa). Tratado como requisito não funcional, não decisão arquitetural separada ([09:24] Larissa).

## Alternativas Consideradas

1. **Incluir `items` no payload** — descartada: infla o evento; o cliente consulta `GET /orders/:id` ([09:43] Diego).
2. **Truncar payloads grandes** — descartada: Sofia defende erro, pois evento grande indica algo errado ([09:23] Sofia).
3. **Timeout maior ou sem timeout** — não discutido; 10 s foi o valor proposto e aceito ([09:42] Diego, Sofia).

## Consequências

**Positivas**
- Payload pequeno, previsível e barato de armazenar na outbox; cliente tem tudo para rotear e deduplicar via headers.
- Timeout limita o tempo que um cliente lento ocupa o worker single-thread.

**Negativas / trade-offs**
- Cliente que precise de itens faz uma chamada extra a `GET /orders/:id` (acoplamento de leitura e consumo extra de API).
- Com worker único, um cliente que sempre estoura 10 s atrasa a fila dos demais (head-of-line blocking) até o evento ser reagendado.
- Um evento acima de 64 KB falha de forma permanente, sem entrega.
