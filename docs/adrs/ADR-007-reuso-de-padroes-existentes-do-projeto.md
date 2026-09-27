# ADR-007: Reuso máximo dos padrões arquiteturais já existentes no projeto

## Status

Aceito.

## Contexto

Ao discutir estrutura de código, o time decidiu explicitamente que o módulo de webhooks não introduziria convenções novas, e sim seguiria os mesmos padrões já usados pelos módulos existentes (`auth`, `users`, `customers`, `products`, `orders`).

> [09:27] Larissa: Estamos a uma hora de reunião, vou tentar fechar mais rápido. Próximo bloco: estrutura do código e padrões. Bruno, fala.

> [09:27] Bruno: A gente tem um padrão claro na codebase. Cada domínio é um módulo em src/modules com controller, service, repository, routes e schemas. Webhook vai seguir igual. Vou propor uma pasta src/modules/webhooks com toda a estrutura. Faz sentido?

> [09:28] Diego: Faz. E o worker fica onde?

Essa pergunta de Diego sobre onde o worker vive é respondida por Bruno na mesma troca e está detalhada na [ADR-002](ADR-002-worker-dedicado-com-polling.md) (`src/worker.ts` como entry-point separado).

De fato, o módulo `src/modules/orders/` segue exatamente essa composição: `order.controller.ts` (controller fino que delega ao service — `OrderController` em `src/modules/orders/order.controller.ts`), `order.service.ts` (regra de negócio, recebe repository e `PrismaClient` por injeção de construtor — `OrderService`), `order.repository.ts` (`OrderRepository`, encapsula acesso ao Prisma), `order.routes.ts` (função `buildOrderRouter(controller)` que monta o `Router` do Express) e `order.schemas.ts` (schemas Zod + tipos inferidos). A composição das dependências é manual, feita em `buildControllers` (`src/app.ts`), e o router do módulo é registrado em `buildApiRouter` (`src/routes/index.ts`) — não existe classe abstrata de repository/service no projeto; cada módulo repete o mesmo padrão de construtor de forma independente.

Sobre o tratamento de erros, o time decidiu reaproveitar a hierarquia de exceções já existente, com o mesmo padrão de prefixo de código por domínio:

> [09:28] Bruno: Sobre erros: a gente já tem um padrão. Tem classe AppError, classes específicas tipo InsufficientStockError, InvalidStatusTransitionError. Todas usam código tipo INSUFFICIENT_STOCK, INVALID_STATUS_TRANSITION. Quero seguir igual pra webhook. Códigos tipo WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED, etc.

> [09:29] Larissa: Prefixo WEBHOOK_ pra tudo do módulo.

> [09:29] Bruno: E o logger, que é Pino, já tá no projeto inteiro. Não vamos botar nada novo. O middleware de erro centralizado já trata AppError, Zod e Prisma. Vai pegar nossos erros sem precisar mudar nada.

Essa hierarquia existe hoje em `src/shared/errors/app-error.ts` (classe base `AppError`, com `statusCode`, `errorCode` e `details`) e `src/shared/errors/http-errors.ts`, com subclasses como `InvalidStatusTransitionError` (código `INVALID_STATUS_TRANSITION`) e `InsufficientStockError` (código `INSUFFICIENT_STOCK`), além de `NotFoundError`, `ConflictError`, `ForbiddenError`, `UnauthorizedError`, `BadRequestError` e `UnprocessableEntityError`, todas re-exportadas por `src/shared/errors/index.ts`. O middleware centralizado `src/middlewares/error.middleware.ts` já trata `instanceof AppError` (responde com `err.statusCode` e `{ error: { code: err.errorCode, message, ...details } }`), `ZodError` (400, `VALIDATION_ERROR`) e `Prisma.PrismaClientKnownRequestError` (`P2002` → 409, `P2025` → 404) — nenhuma mudança nesse middleware é necessária para o módulo de webhooks funcionar.

Por fim, ficou definido que o worker e a API compartilham banco e stack, mas não a instância de `PrismaClient`:

> [09:29] Diego: Sobre infraestrutura compartilhada: o pool de conexão do Prisma já tá lá. O worker abre o mesmo PrismaClient ou um separado?

> [09:30] Bruno: Separado. PrismaClient é por processo. Mesmo banco, mesma DATABASE_URL, mas instância nova porque é outro processo Node.

> [09:30] Larissa: Decisão: reuso máximo do que já existe. AppError, Pino, error middleware, padrão de módulos, padrão de schemas Zod, padrão de códigos de erro. Webhook fica como módulo igual aos outros.

## Decisão

O módulo de webhooks será implementado como **`src/modules/webhooks/`**, seguindo exatamente a mesma composição estrutural de `src/modules/orders/` (controller, service, repository, routes, schemas), com injeção manual de dependências em `buildControllers` (`src/app.ts`) e registro do router em `buildApiRouter` (`src/routes/index.ts`) — sem criar abstrações novas (classes base, interfaces genéricas de repository) que não existem hoje no restante do projeto.

Erros do módulo estendem a `AppError` já existente (`src/shared/errors/app-error.ts`), com códigos prefixados por **`WEBHOOK_`** (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`), no mesmo padrão de `INSUFFICIENT_STOCK` e `INVALID_STATUS_TRANSITION` definidos em `src/shared/errors/http-errors.ts`. O middleware `src/middlewares/error.middleware.ts` não precisa de nenhuma alteração para tratar esses erros, pois já reconhece qualquer subclasse de `AppError`.

Validação de entrada (body/query/params) usa o mesmo `validate()` de `src/middlewares/validate.middleware.ts` com schemas Zod no padrão de `src/modules/orders/order.schemas.ts`. Logging usa o logger Pino já existente (`src/shared/logger/index.ts`), sem introduzir nova biblioteca de log. O endpoint de replay de DLQ (ADR-004) reaproveita `requireRole` de `src/middlewares/auth.middleware.ts`.

O worker (`src/worker.ts`, ADR-002) roda no mesmo banco e mesma `DATABASE_URL` que a API, mas com sua **própria instância de `PrismaClient`**, criada via a mesma factory `createPrismaClient()` de `src/config/database.ts` — nunca compartilhando a instância singleton `prisma` do processo da API.

## Alternativas Consideradas

**1. Criar uma convenção de erros, logging e estrutura de módulo específica para o domínio de webhooks** (por exemplo, uma nova classe de erro base própria, ou um formato de código de erro diferente do padrão `DOMINIO_MOTIVO` já usado). Considerada implicitamente e descartada: o time optou de forma explícita por reuso máximo, justamente para evitar inconsistência dentro do projeto e reduzir a curva de entendimento para quem já conhece os módulos existentes.

## Consequências

**Positivas:**

- Consistência arquitetural: qualquer desenvolvedor familiarizado com `src/modules/orders/` (ou `users`, `customers`, `products`) já entende a estrutura do módulo de webhooks sem curva de aprendizado adicional.
- Zero mudança necessária na infraestrutura transversal já validada em produção: `error.middleware.ts`, logger Pino, `validate.middleware.ts` e `requireRole` funcionam para o novo módulo sem modificação.
- Convenção de código de erro previsível (`WEBHOOK_*`) facilita tanto o consumo interno (testes, documentação) quanto a comunicação de erros para os clientes externos que integrarão com os novos endpoints.

**Negativas (trade-off explícito):**

- Herdar os padrões existentes também significa herdar suas limitações atuais — por exemplo, a ausência de uma classe abstrata de repository/service no projeto obriga o módulo de webhooks a repetir manualmente o mesmo boilerplate de injeção de dependência já replicado em cada módulo existente, em vez de uma solução mais genérica.
- Acoplar o novo módulo ao padrão de módulo único do projeto (sem abstrações compartilhadas) significa que qualquer refatoração futura da estrutura padrão de módulos (ex.: introdução de uma classe base) exigiria tocar também o módulo de webhooks para manter a consistência.
