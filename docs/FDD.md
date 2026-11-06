# FDD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Status** | Rascunho para revisão (pós-RFC) |
| **Data** | 2026-11-06 |
| **Relacionados** | [PRD](PRD.md) · [RFC](RFC.md) · [TRACKER](TRACKER.md) · [ADRs](adrs/) |

> **Convenção de origem.** Itens marcados **[Reunião]** vêm direto da transcrição (ver [TRACKER](TRACKER.md)). Itens marcados **[Proposta]** são detalhamento de implementação necessário para construir, **sem decisão explícita na reunião**, e devem ser validados na sessão de revisão que a Larissa marcou com Bruno e Diego ([09:50] Larissa). Nomes de tabelas além de `webhook_outbox` e `webhook_dead_letter` são **[Proposta]**.

## 1. Contexto e motivação técnica

O OMS não possui mecanismo de notificação externa. Clientes B2B fazem polling em `GET /orders`. A mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de `prisma.$transaction`, que já atualiza `orders`, insere em `order_status_history` e debita/repõe `stock_quantity`. A feature adiciona entrega de eventos de saída sem acoplar essa transação a chamadas HTTP externas ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)).

## 2. Objetivos técnicos

- OT-01: Nenhuma mudança de status sem evento registrado (quando há webhook interessado), e nenhum evento de transação revertida. **[Reunião]**
- OT-02: Latência fim-a-fim <10 s; polling de 2 s. **[Reunião]**
- OT-03: Entrega at-least-once, assinada, com retry e DLQ. **[Reunião]**
- OT-04: Reuso integral dos padrões do projeto ([ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)). **[Reunião]**
- OT-05: Worker em processo separado, mesmo banco e stack. **[Reunião]**

## 3. Escopo e exclusões

**Dentro:** tabelas de configuração, outbox, entregas e DLQ; módulo `src/modules/webhooks`; `src/worker.ts`; alteração em `changeStatus`; 7 endpoints HTTP; HMAC; retry/DLQ; logs.

**Fora (decididos na reunião):** e-mail de alerta de falha ([09:37] Larissa); rate limiting de saída ([09:39] Larissa); dashboard visual ([09:40] Larissa); arquivamento de linhas entregues ([09:08] Diego); múltiplos workers e ordenação global ([09:13] Diego/Larissa); webhooks de entrada ([09:02] Marcos); exactly-once ([09:25] Diego). **Fora por restrição do desafio:** nenhuma alteração de código é feita nesta entrega, apenas documentação.

## 4. Modelo de dados

Todas as chaves em UUID (`@id @default(uuid()) @db.Char(36)`), como em `prisma/schema.prisma` ([09:51] Larissa). Nomes em `snake_case` via `@@map`, como as tabelas existentes. Os valores de status da outbox são `PENDING`, `PROCESSING`, `FAILED`, `DELIVERED` (tradução de pendente/processando/falhou/entregue [09:08] Diego) **[Proposta de nomes]**.

```prisma
enum WebhookEventStatus { PENDING PROCESSING FAILED DELIVERED }

model WebhookEndpoint {            // [Proposta] nome; campos: [09:21] Bruno
  id                      String   @id @default(uuid()) @db.Char(36)
  customerId              String   @db.Char(36)
  url                     String   @db.VarChar(2048)
  secret                  String   @db.VarChar(255)       // ver RFC-OQ-02
  previousSecret          String?  @db.VarChar(255)       // grace 24h
  previousSecretExpiresAt DateTime?
  events                  Json                            // lista de OrderStatus assinados [09:33]
  active                  Boolean  @default(true)
  createdAt               DateTime @default(now())
  updatedAt               DateTime @updatedAt
  customer                Customer @relation(fields: [customerId], references: [id])
  @@index([customerId, active])
  @@map("webhook_endpoints")
}

model WebhookOutbox {              // nome da tabela: [09:06] Diego
  id            String   @id @default(uuid()) @db.Char(36)  // = X-Event-Id [09:25]
  webhookId     String   @db.Char(36)                        // sem FK: ver 6.4
  orderId       String   @db.Char(36)
  payload       Json                                          // snapshot [09:52]
  status        WebhookEventStatus @default(PENDING)
  attempts      Int      @default(0)
  nextAttemptAt DateTime @default(now())                      // [Proposta]
  lastError     String?  @db.VarChar(500)
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt
  @@index([status, nextAttemptAt])   // índice em status [09:08]
  @@index([createdAt])               // índice em created_at [09:08]
  @@map("webhook_outbox")
}

model WebhookDelivery {            // [Proposta] derivado de [09:34] Marcos
  id           String   @id @default(uuid()) @db.Char(36)
  eventId      String   @db.Char(36)
  webhookId    String   @db.Char(36)
  attempt      Int
  success      Boolean
  statusCode   Int?
  responseBody String?  @db.Text      // truncado
  durationMs   Int
  error        String?  @db.VarChar(500)
  createdAt    DateTime @default(now())
  @@index([webhookId, createdAt])
  @@map("webhook_deliveries")
}

model WebhookDeadLetter {          // nome: [09:18] Diego
  id            String   @id @default(uuid()) @db.Char(36)
  eventId       String   @db.Char(36)
  webhookId     String   @db.Char(36)
  payload       Json
  failureReason String   @db.VarChar(500)
  failedAt      DateTime @default(now())
  replayedAt    DateTime?             // [Proposta]
  replayedById  String?  @db.Char(36) // auditoria [09:36] Sofia
  @@map("webhook_dead_letter")
}
```

A migration é nova (`prisma/migrations/<timestamp>_webhooks/`), sem alterar tabelas existentes.

## 5. Contratos públicos

Prefixo `/api/v1` (`src/routes/index.ts`). Todas as rotas usam `authenticate` (JWT Bearer) e `validate` com Zod. Respostas de erro no formato `{ "error": { "code", "message", "details?" } }` do `errorMiddleware`. Campos de API em camelCase, como `createOrderSchema`; o payload **enviado ao cliente** usa snake_case conforme [09:43] Diego. Roteador novo: `src/modules/webhooks/webhook.routes.ts`, registrado em `buildApiRouter`.

### 5.1 `POST /api/v1/webhooks` — cadastrar webhook **[Reunião]**
Qualquer role autenticada ([09:36] Marcos, [09:37] Sofia). `customerId` vem no body, não do JWT ([09:32] Larissa).

Request:
```json
{
  "customerId": "6f1c2b9e-0b0e-4c5f-9a43-0d6b2a1f7e10",
  "url": "https://erp.atlas.example/hooks/oms",
  "events": ["SHIPPED", "DELIVERED"]
}
```
Response `201 Created` (única vez em que a secret é devolvida):
```json
{
  "id": "b7a1e0c4-52d2-4f0e-8d3a-2f7d9c1e5a11",
  "customerId": "6f1c2b9e-0b0e-4c5f-9a43-0d6b2a1f7e10",
  "url": "https://erp.atlas.example/hooks/oms",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_9d2f...",
  "createdAt": "2026-11-06T12:00:00.000Z"
}
```
Status: `201`; `400` `VALIDATION_ERROR` / `WEBHOOK_INVALID_URL` (http ou URL inválida); `401`; `404` `WEBHOOK_CUSTOMER_NOT_FOUND`.

### 5.2 `GET /api/v1/webhooks?customerId=...&page=&pageSize=` — listar **[Reunião]**
Lista os webhooks de um customer ([09:33] Bruno), paginação padrão `paginated()` (`src/shared/http/response.ts`). A secret **não** é retornada.
```json
{
  "data": [{ "id": "b7a1e0c4-...", "customerId": "6f1c2b9e-...", "url": "https://erp.atlas.example/hooks/oms",
             "events": ["SHIPPED","DELIVERED"], "active": true, "createdAt": "2026-11-06T12:00:00.000Z" }],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```
Status: `200`; `400`; `401`.

### 5.3 `PATCH /api/v1/webhooks/:id` — editar **[Reunião]**
Request (campos opcionais): `{ "url": "https://novo.example/hooks", "events": ["PAID"], "active": false }`
Response `200`: objeto do webhook como em 5.2. Status: `200`; `400` `WEBHOOK_INVALID_URL`; `401`; `404` `WEBHOOK_NOT_FOUND`.

### 5.4 `DELETE /api/v1/webhooks/:id` — remover **[Reunião]**
Response `204 No Content` (mesmo padrão do `DELETE /orders/:id`). Status: `204`; `401`; `404` `WEBHOOK_NOT_FOUND`.

### 5.5 `POST /api/v1/webhooks/:id/rotate-secret` — rotacionar secret **[Reunião: existência]** / rota **[Proposta]**
Gera nova secret, move a atual para `previousSecret` com `previousSecretExpiresAt = agora + 24h` ([09:21] Sofia).
Response `200`:
```json
{ "id": "b7a1e0c4-...", "secret": "whsec_new...", "previousSecretExpiresAt": "2026-11-07T12:00:00.000Z" }
```
Status: `200`; `401`; `404` `WEBHOOK_NOT_FOUND`.

### 5.6 `GET /api/v1/webhooks/:id/deliveries` — histórico de entregas **[Reunião]**
Últimas 100 entregas ([09:34] Marcos), ordenadas da mais recente, com sucesso/falha, payload, resposta e tempo.
```json
{
  "data": [{
    "eventId": "0c9e3a52-8b1f-4e6a-9f0d-6a8d1d2c4b77", "attempt": 1, "success": true,
    "statusCode": 200, "durationMs": 184, "responseBody": "ok", "error": null,
    "payload": { "event_type": "order.status_changed", "to_status": "SHIPPED" },
    "createdAt": "2026-11-06T12:05:02.000Z"
  }]
}
```
Status: `200`; `401`; `404` `WEBHOOK_NOT_FOUND`.

### 5.7 `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — reprocessar DLQ **[Reunião]**
Exige `requireRole('ADMIN')` ([09:36] Larissa). Recoloca o evento na outbox como `PENDING` (`attempts = 0`) e registra em log o `userId` que executou ([09:36] Sofia).
Response `200`:
```json
{ "eventId": "0c9e3a52-...", "status": "PENDING", "replayedBy": "8d1f...", "replayedAt": "2026-11-06T14:00:00.000Z" }
```
Status: `200`; `401`; `403` `FORBIDDEN` (OPERATOR); `404` `WEBHOOK_DEAD_LETTER_NOT_FOUND`; `409` `WEBHOOK_REPLAY_NOT_ALLOWED` **[Proposta]** (já reprocessado).

### 5.8 Contrato de saída (requisição do worker ao cliente) **[Reunião]**
`POST <url do cliente>`, `Content-Type: application/json`, timeout 10 s, corpo ≤ 64 KB ([ADR-008](adrs/ADR-008-payload-headers-e-limites-de-entrega.md)).

| Header | Conteúdo |
|---|---|
| `X-Event-Id` | UUID do evento (= `webhook_outbox.id`) |
| `X-Signature` | HMAC-SHA256 do corpo bruto com a secret do endpoint (hex; formato exato **[Proposta]**) |
| `X-Timestamp` | Momento do envio |
| `X-Webhook-Id` | Id do `WebhookEndpoint` |
| `Content-Type` | `application/json` |

Corpo:
```json
{
  "event_id": "0c9e3a52-8b1f-4e6a-9f0d-6a8d1d2c4b77",
  "event_type": "order.status_changed",
  "timestamp": "2026-11-06T12:05:00.000Z",
  "order_id": "d3f6a2c1-77aa-4a0e-b1c9-5e6f7a8b9c0d",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "6f1c2b9e-0b0e-4c5f-9a43-0d6b2a1f7e10",
  "total_cents": 125000
}
```
Sem `items` ([09:43] Diego). **Critério de sucesso: resposta HTTP 2xx** **[Proposta]**; qualquer outro status, timeout ou erro de rede é falha e entra no retry.

## 6. Fluxos detalhados

### 6.1 Criação do evento na outbox (dentro de `changeStatus`)
1. `changeStatus` valida a transição e atualiza estoque, pedido e histórico (código atual).
2. Em seguida, chama `publishWebhookEvent(tx, order, from, to)` ([ADR-007](adrs/ADR-007-publish-webhook-event-na-transacao.md)).
3. A função busca `webhook_endpoints` com `customerId = order.customerId`, `active = true` e `to` presente em `events`. Sem resultados: retorna sem inserir ([09:34] Bruno).
4. Para cada endpoint, monta o snapshot (seção 5.8; `event_id` = UUID novo). O limite de 64 KB é verificado no **envio** (6.2), não aqui, para que um evento anômalo nunca reverta a mudança de status **[Proposta: ponto de verificação]**.
5. Insere em `webhook_outbox` com `status = PENDING`, `attempts = 0`, `nextAttemptAt = now()`.
6. Qualquer erro propaga e **desfaz a transação inteira** ([09:40] Bruno).

```
changeStatus → tx{ update order · insert history · publishWebhookEvent → insert outbox*N } → commit
```

### 6.2 Processamento pelo worker
`src/worker.ts` cria seu próprio `PrismaClient` ([09:30] Bruno), instala handlers de SIGINT/SIGTERM como `src/server.ts` e entra no loop (a cada 2 s, [09:09] Diego), sem sobreposição de ciclos:
1. Busca lote pequeno: `status = PENDING AND nextAttemptAt <= now()` ordenado por `createdAt` ASC ([09:12] Diego). Tamanho do lote **[Proposta]**.
2. Marca como `PROCESSING` e carrega o endpoint. Se o endpoint não existe/está inativo, descarta com `WEBHOOK_ENDPOINT_INACTIVE` (sem retry) **[Proposta]**.
3. Serializa o corpo; se > 64 KB, **não envia**: falha permanente com `WEBHOOK_PAYLOAD_TOO_LARGE`, sem retry, evento vai direto à DLQ ([09:23] Sofia). Senão calcula HMAC e faz `POST` com timeout de 10 s.
4. Registra uma linha em `webhook_deliveries` com resultado, status code e tempo.
5. Sucesso (2xx): `status = DELIVERED`. Falha: ir para 6.3.
6. Na inicialização, eventos presos em `PROCESSING` (crash do worker) voltam a `PENDING` **[Proposta]**; reforça a necessidade de dedup via `X-Event-Id` (at-least-once).

### 6.3 Retry
Falha → `attempts += 1`. Se restam tentativas, `status = PENDING` e `nextAttemptAt = now() + backoff[attempts]`, com `backoff = [1m, 5m, 30m, 2h, 12h]` ([09:17] Larissa). Total de ~15 h entre a primeira falha e a última tentativa. **Contagem a confirmar (RFC-OQ-03):** assumimos tentativa inicial + 5 retries.

### 6.4 DLQ
Esgotadas as tentativas, em uma transação: insere em `webhook_dead_letter` (payload, motivo, timestamp, [09:18] Diego) e marca a outbox como `FAILED`. O replay (5.7) cria novo ciclo: `PENDING`, `attempts = 0`. Sem FK de `webhookId` para `webhook_endpoints` nas tabelas de evento, para que remover a configuração não apague evidência **[Proposta]**.

## 7. Matriz de erros

Subclasses de `AppError` com prefixo `WEBHOOK_` ([09:29] Larissa), em `src/modules/webhooks/webhook.errors.ts` **[Proposta: local do arquivo]**, exportadas junto às existentes. Os 3 primeiros códigos foram citados na reunião ([09:28] Bruno); os demais são **[Proposta]**.

| Código | HTTP | Onde | Quando | Retry |
|---|---|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | API | Webhook inexistente | n/a |
| `WEBHOOK_INVALID_URL` | 400 | API | URL não https ou inválida ([09:23] Sofia) | n/a |
| `WEBHOOK_SECRET_REQUIRED` | 400/500 | API/worker | Endpoint sem secret utilizável para assinar | Não |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | API | `customerId` inexistente no cadastro | n/a |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | API | Id de DLQ inexistente | n/a |
| `WEBHOOK_REPLAY_NOT_ALLOWED` | 409 | API | Evento da DLQ já reprocessado | n/a |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | interno | worker | Snapshot > 64 KB, não enviado ([09:23] Sofia) | Não (vai à DLQ) |
| `WEBHOOK_DELIVERY_TIMEOUT` | interno | worker | Sem resposta em 10 s ([09:42] Diego) | Sim |
| `WEBHOOK_DELIVERY_FAILED` | interno | worker | Resposta não 2xx ou erro de rede | Sim |
| `WEBHOOK_ENDPOINT_INACTIVE` | interno | worker | Endpoint removido/desativado antes do envio | Não |
| `WEBHOOK_OUTBOX_INSERT_FAILED` | 500 | `changeStatus` | Falha ao inserir outbox; rollback do status | n/a |

Erros genéricos (`VALIDATION_ERROR`, `UNAUTHORIZED`, `FORBIDDEN`) continuam vindo das classes atuais.

## 8. Estratégias de resiliência

| Aspecto | Estratégia | Origem |
|---|---|---|
| Timeout | 10 s por chamada, falha vai para retry | [09:42] Diego |
| Retry | 5 tentativas, backoff 1m/5m/30m/2h/12h | [09:17] Larissa |
| Fallback | DLQ em `webhook_dead_letter` + replay ADMIN | [09:18] Diego |
| Atomicidade | Evento na mesma transação do status | [09:40] Bruno |
| Isolamento | Worker em processo separado; API independente | [09:11] Diego |
| Crash do worker | Reset de `PROCESSING` → `PENDING` no boot | **[Proposta]** |
| Cliente lento | Timeout limita o bloqueio do worker único | [09:42] Diego |
| Ordenação | Por `createdAt` e `order_id`, só com um worker; retries podem reordenar | [09:12] Diego |

Fora do escopo: rate limiting, circuit breaker e e-mail de alerta.

## 9. Observabilidade

O projeto só possui **logs com Pino**. Métricas e tracing **não existem** no código atual e **não foram discutidos** na reunião; as definições abaixo são **[Proposta]** e devem ser revisadas (a adoção de qualquer biblioteca nova é decisão em aberto).

- **Logs (Pino, `src/shared/logger/index.ts`):** eventos estruturados `webhook_event_enqueued`, `webhook_delivery_attempt`, `webhook_delivery_failed`, `webhook_moved_to_dlq`, `webhook_replayed` (com `userId`, exigência de auditoria [09:36] Sofia), `webhook_worker_started/stopped`. Campos: `eventId`, `webhookId`, `orderId`, `attempt`, `statusCode`, `durationMs`. **Adicionar `secret`/`previousSecret` à lista `redactPaths`** e nunca logar o corpo de resposta do cliente completo.
- **Métricas (derivadas das tabelas e logs):** eventos por status (`PENDING`, `FAILED`, `DELIVERED`), idade do evento `PENDING` mais antigo (alvo <10 s), taxa de sucesso por endpoint, tamanho da DLQ, latência de entrega.
- **Tracing:** propagar `eventId` como correlation id entre `publishWebhookEvent`, a linha da outbox, os logs do worker e o header `X-Event-Id`; o `requestId` do `pino-http` (`src/middlewares/request-logger.middleware.ts`) liga o evento à requisição que mudou o status. Tracing distribuído (ex.: OpenTelemetry) fica como decisão futura.

## 10. Integração com o sistema existente

| Arquivo (existe no repositório) | Como o módulo se integra |
|---|---|
| `src/modules/orders/order.service.ts` | `changeStatus` chama `publishWebhookEvent(tx, order, from, to)` logo após `tx.orderStatusHistory.create`, dentro da `$transaction`. `create` não muda (RFC-OQ-05). |
| `src/modules/orders/order.status.ts` | Fonte dos valores de `OrderStatus`/transições; a lista `events` do webhook é validada contra o enum `OrderStatus`. Sem alteração. |
| `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` | Erros `WEBHOOK_*` estendem `AppError`/`NotFoundError`/`ConflictError`/`UnprocessableEntityError`, como `InsufficientStockError`. |
| `src/shared/errors/index.ts` | Reexportar os novos erros para consumo pelo módulo. |
| `src/middlewares/error.middleware.ts` | Sem alteração: já serializa `AppError`, `ZodError` e erros Prisma ([09:29] Bruno). |
| `src/middlewares/auth.middleware.ts` | Reuso de `authenticate` em todas as rotas e `requireRole('ADMIN')` no replay. |
| `src/middlewares/validate.middleware.ts` | Reuso de `validate` com schemas Zod do módulo (inclui `https` na URL). |
| `src/app.ts` | `buildControllers` instancia `WebhookRepository`, `WebhookService`, `WebhookController` e os expõe em `Controllers`. |
| `src/routes/index.ts` | `buildApiRouter` monta `/webhooks` e `/admin/webhooks`. |
| `src/server.ts` | Modelo para o novo `src/worker.ts` (bootstrap, SIGINT/SIGTERM, `prisma.$disconnect()`). A API não muda. |
| `src/config/database.ts` | Nova instância de `PrismaClient` no processo do worker ([09:30] Bruno), via `createPrismaClient()`. |
| `src/config/env.ts` | Variáveis novas (intervalo de polling, tamanho do lote) entram no mesmo schema Zod, com defaults (2000 ms) **[Proposta]**. |
| `src/shared/logger/index.ts` | Reuso do Pino no worker e na API; extensão de `redactPaths`. |
| `prisma/schema.prisma` | Novos modelos (seção 4); relação com `Customer`. |
| `package.json` | Novo script `"worker"` (modelado em `dev`/`start`, [09:11] Larissa) ; sem novas dependências para HTTP (`fetch` nativo do Node ≥20, **[Proposta]**) nem HMAC (`node:crypto`). |
| `tests/setup.ts` e `tests/orders.test.ts` | Implementação futura: limpar as novas tabelas no `beforeEach` e cobrir `changeStatus` + outbox. **Não alterados nesta entrega documental.** |

## 11. Dependências e compatibilidade

- Node ≥ 20 (`package.json`), MySQL 8 (`docker-compose.yml`), Prisma 5.22, Zod 3.23, Pino 9.5.
- Compatibilidade retroativa: contratos de `/orders` inalterados; migration apenas aditiva.
- Clientes dependem de verificar HMAC e deduplicar por `X-Event-Id`.
- Revisão de segurança (≥2 dias úteis) antes do deploy ([09:46] Sofia).
- Portal de desenvolvedor documentado pelo PM ([09:40] Marcos).

## 12. Critérios de aceite técnicos

- CA-01: Mudança de status com webhook interessado cria exatamente 1 linha em `webhook_outbox` por endpoint, na mesma transação; falha forçada no insert reverte o status.
- CA-02: Mudança de status sem webhook interessado não cria linha.
- CA-03: Com API e worker ativos, evento é entregue em <10 s (p95) em cenário sem falha.
- CA-04: Corpo assinado: HMAC-SHA256 recomputável pelo cliente com a secret do endpoint; headers conforme 5.8.
- CA-05: Falhas seguem 1m/5m/30m/2h/12h e, esgotadas, o evento aparece em `webhook_dead_letter`.
- CA-06: Replay só por ADMIN (403 para OPERATOR), registra `userId`, reinsere como `PENDING`.
- CA-07: URL `http` rejeitada com `WEBHOOK_INVALID_URL`; payload >64 KB não é enviado e vai à DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`.
- CA-08: Após `rotate-secret`, a secret anterior expira em 24 h.
- CA-09: Parar a API não interrompe o worker.
- CA-10: Todos os erros do módulo usam prefixo `WEBHOOK_` e o formato do `errorMiddleware`.

## 13. Riscos e mitigação

| Risco | Mitigação |
|---|---|
| Falha na outbox bloqueia mudança de status | Testes ponta a ponta; mantê-la simples (insert) |
| Cliente lento atrasa fila (worker único) | Timeout de 10 s; monitorar idade do `PENDING` mais antigo |
| Eventos presos em `PROCESSING` | Reset no boot do worker; dedup por `X-Event-Id` |
| Vazamento de secret em log ou banco | `redactPaths`; decidir proteção em repouso (RFC-OQ-02); revisão de Sofia |
| Crescimento da outbox | Índices; arquivamento planejado como feature futura |
| Ordenação quebrada por retry ou futuro multi-worker | Documentar limitação; ordenação só por `order_id` com um worker |
| Prazo apertado (3 sprints, entrega no fim de novembro) | Priorizar outbox/worker/HMAC; reservar a revisão de segurança |
