# ADR-003: Autenticação das entregas com HMAC-SHA256 e secret por endpoint

- **Status:** Aceito
- **Data:** 2026-11-06
- **Decisores:** Sofia (Segurança), Larissa, Diego
- **Fonte:** TRANSCRICAO [09:19]–[09:23], [09:36], [09:44]

## Contexto

Os eventos carregam dados de pedidos para um endpoint fora da nossa infraestrutura. O cliente precisa validar que a requisição veio de nós e que o payload não foi adulterado ([09:19] Sofia). Já houve cliente que vazou secret em log de aplicação ([09:22] Diego).

## Decisão

- Assinar o **corpo do request com HMAC-SHA256**, enviando a assinatura no header `X-Signature` ([09:20] Sofia).
- **Uma secret única por endpoint de webhook**, nunca uma secret global da plataforma; a secret é gerada pela plataforma e devolvida na criação ([09:21] Sofia, [09:31] Marcos).
- **Rotação via API**: o cliente solicita nova secret; a anterior permanece válida por **24 horas** (grace period) em paralelo e depois deixa de valer ([09:21] Sofia, [09:22] Sofia "Decidido").
- O header `X-Timestamp` (momento do envio) permite ao cliente detectar replay attack, se quiser ([09:44] Diego).
- A URL cadastrada deve ser **https**; `http` é recusado por validação no schema Zod. Sofia classificou isto como validação, não decisão arquitetural ([09:23] Sofia).
- A revisão de segurança de HMAC e geração de secret ocorre antes do deploy (mínimo 2 dias úteis) ([09:46] Sofia).

## Alternativas Consideradas

1. **Secret global da plataforma** — descartada: se vaza uma, vaza tudo ([09:21] Sofia).
2. **Sem assinatura, confiando apenas em TLS** — descartada implicitamente: TLS não prova origem nem impede adulteração por quem controla o destino; a assinatura foi tratada como requisito ([09:19] Sofia).
3. **Rotação com corte imediato da secret antiga** — descartada: o cliente precisa de tempo para migrar seus sistemas ([09:21] Sofia).

## Consequências

**Positivas**
- Vazamento de uma secret afeta um único endpoint.
- Padrão de mercado; clientes têm biblioteca para verificar HMAC-SHA256 ([09:20] Sofia).
- Rotação sem janela de indisponibilidade para o cliente.

**Negativas / trade-offs**
- Para assinar, a plataforma precisa recuperar a secret em texto claro (diferente de uma senha com hash), o que exige cuidado de armazenamento. **Como proteger a secret em repouso não foi decidido na reunião** e está em aberto no [RFC](../RFC.md) (RFC-OQ-02).
- Durante o grace period há duas secrets ativas por endpoint, aumentando a superfície; **o mecanismo exato de assinatura durante o grace (uma ou duas assinaturas) também ficou em aberto** (RFC-OQ-04).
- O logger Pino atual (`src/shared/logger/index.ts`) não redige o campo `secret`; será preciso estender a lista `redactPaths` para não vazar secrets em log.
