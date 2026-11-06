# ADR-005: Worker em processo separado com polling de 2 segundos

- **Status:** Aceito
- **Data:** 2026-11-06
- **Decisores:** Diego, Larissa, Bruno
- **Fonte:** TRANSCRICAO [09:08]–[09:13], [09:28]–[09:30]

## Contexto

Precisamos de um componente que leia a `webhook_outbox` ([ADR-001](ADR-001-outbox-no-mysql.md)) e dispare as chamadas HTTP. O requisito de negócio é latência inferior a 10 segundos para ser percebido como "tempo real" ([09:02] Marcos). O projeto hoje tem um único entry point, `src/server.ts`, que sobe o Express e encerra o Prisma no shutdown.

## Decisão

- **Polling em loop a cada 2 segundos**: busca os eventos pendentes mais antigos, processa, marca ([09:09] Diego). Latência mínima no pior caso de 2 s, aceita ([09:10] Larissa); Marcos confirmou ([09:10]).
- O worker roda como **processo separado da API**, para que reiniciar a API não derrube o worker ([09:11] Diego).
- Novo entry point **`src/worker.ts`**, com script `npm run worker`; a lógica de processamento fica em `src/modules/webhooks/webhook.worker.ts` ou `webhook.processor.ts` ([09:11] Larissa, [09:28] Bruno).
- Mesmo banco e mesma `DATABASE_URL`, porém **uma instância própria de `PrismaClient`**, pois o client é por processo ([09:30] Bruno).
- **Single-worker por enquanto**; ordenação por `created_at` da outbox, com garantia **apenas por `order_id` e apenas enquanto houver um único worker** (limitação conhecida, ver abaixo) ([09:12] Diego, [09:13] Larissa).

## Alternativas Consideradas

1. **Trigger de banco / listener reativo** — descartada: MySQL não tem LISTEN/NOTIFY; trigger só executa SQL e teria que "improvisar" escrita em arquivo ou chamada a endpoint ([09:09] Diego).
2. **Worker dentro do processo da API** — descartada: reinício da API perde o worker ([09:11] Diego).
3. **Múltiplos workers em paralelo desde já** — adiada: perderia a ordenação; futuro particionamento por `order_id` ou lock pessimista ([09:13] Diego).

## Consequências

**Positivas**
- Simples, sem infraestrutura nova, previsível e dentro do requisito de <10 s.
- Isolamento de falhas: API e worker reiniciam independentemente.

**Negativas / trade-offs**
- Latência de até 2 s adicionada em todo evento; consultas periódicas ao MySQL mesmo sem eventos.
- **Single point of throughput**: escalar horizontalmente exige redesenho (particionar por `order_id` ou lock pessimista) e perde a ordenação atual.
- **Sem garantia de ordenação global**, só por `order_id` e só com um worker — registrada como limitação conhecida; clientes nunca pediram ordenação global ([09:14] Marcos).
- Dois processos para operar e monitorar (API + worker).
