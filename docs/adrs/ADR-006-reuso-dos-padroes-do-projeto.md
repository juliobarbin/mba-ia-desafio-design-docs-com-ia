# ADR-006: Reuso máximo dos padrões existentes do projeto no módulo de webhooks

- **Status:** Aceito
- **Data:** 2026-11-06
- **Decisores:** Larissa, Bruno, Diego
- **Fonte:** TRANSCRICAO [09:27]–[09:30], [09:36], [09:51]; CODIGO

## Contexto

A base de código segue padrões claros: cada domínio é um módulo em `src/modules/<dominio>/` com `controller`, `service`, `repository`, `routes` e `schemas` (ex.: `src/modules/orders/`); erros derivam de `AppError` (`src/shared/errors/app-error.ts`) com códigos como `INSUFFICIENT_STOCK` e `INVALID_STATUS_TRANSITION` (`src/shared/errors/http-errors.ts`); o logger é Pino (`src/shared/logger/index.ts`); o `errorMiddleware` central (`src/middlewares/error.middleware.ts`) já trata `AppError`, `ZodError` e erros do Prisma; autorização usa `authenticate` e `requireRole` (`src/middlewares/auth.middleware.ts`); validação usa Zod via `validate` (`src/middlewares/validate.middleware.ts`); IDs são UUID (`prisma/schema.prisma`).

## Decisão

O módulo de webhooks segue os padrões existentes, sem introduzir novas bibliotecas de erro, log ou validação ([09:30] Larissa):

- Pasta **`src/modules/webhooks`** com controller, service, repository, routes e schemas ([09:27] Bruno).
- Erros como subclasses de **`AppError`** com prefixo **`WEBHOOK_`** (`WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`, etc.) ([09:28] Bruno, [09:29] Larissa).
- **Pino** para todo log; o **error middleware central** captura os novos erros sem alteração ([09:29] Bruno).
- Schemas **Zod** (inclusive a validação de URL https) ([09:23] Sofia).
- **`requireRole('ADMIN')`** existente no endpoint de replay ([09:36] Larissa).
- IDs em **UUID** ([09:51] Larissa).
- Registro do novo módulo em `src/app.ts` (`buildControllers`) e `src/routes/index.ts` (`buildApiRouter`), como os demais módulos (consequência direta do padrão).

## Alternativas Consideradas

1. **Biblioteca/framework dedicado de filas e jobs (ex.: BullMQ)** — plausível, não escolhida: exigiria Redis, já descartado no [ADR-001](ADR-001-outbox-no-mysql.md).
2. **Módulo isolado com convenções próprias de erro e log** — descartada em favor de consistência: quem conhece o OMS já sabe operar o módulo novo.

## Consequências

**Positivas**
- Curva de aprendizado quase nula, revisão mais fácil, comportamento uniforme de erro (formato `{ error: { code, message, details } }`).
- Sem dependências novas para erro/log/validação.

**Negativas / trade-offs**
- Herda limitações atuais: por exemplo, `AppError` só carrega códigos HTTP e o logger não redige o campo `secret` (precisa de extensão de `redactPaths`).
- O worker não passa pelo ciclo HTTP; precisa reutilizar `logger` e `AppError` fora do Express, e a configuração `env` (`src/config/env.ts`) é compartilhada, então variáveis novas do worker precisam entrar no mesmo schema Zod.
