# FDD — Feature Design Document: Sistema de Webhooks de Notificação de Pedidos

**Documentos relacionados:** [RFC-001](RFC.md) (proposta técnica de arquitetura) · [ADR-001](adrs/ADR-001-outbox-no-mysql.md) a [ADR-007](adrs/ADR-007-reuso-de-padroes-existentes-do-projeto.md) (decisões fechadas, tratadas aqui como vinculantes) · [`TRANSCRICAO.md`](../TRANSCRICAO.md) (fonte primária).

Este documento não reabre nenhuma decisão arquitetural — todas estão fechadas nas ADRs referenciadas. O objetivo aqui é especificar contratos, fluxos, dados e comportamento em nível suficiente para implementação direta. Onde uma escolha de implementação não é uma decisão literal da reunião, mas uma consequência necessária para tornar a feature construível (ex.: nome exato de uma tabela de log, biblioteca HTTP a usar), isso é sinalizado explicitamente como **decisão desta FDD**, para não ser confundido com decisão da reunião.

## 1. Contexto e Motivação Técnica

Hoje `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126-179`) executa, dentro de `this.prisma.$transaction(async (tx) => {...})`: leitura do pedido, validação de transição (`canTransition`, `src/modules/orders/order.status.ts`), débito/reposição de estoque (`debitStock`/`replenishStock`) e gravação de `orders` + `order_status_history`. Não existe nenhum mecanismo de evento, fila ou notificação externa na aplicação.

> [09:40] Bruno: Sobre integração com o código atual: a alteração crítica é dentro do service de orders, no método changeStatus. Hoje a transação faz update na order, insere no history e atualiza estoque. A gente vai inserir na webhook_outbox dentro da mesma transação. Se a outbox falhar de inserir, rollback. Não pode ter caso de status mudar e evento não sair.

A motivação de negócio (clientes B2B pedindo notificação abaixo de 10s) está detalhada no [PRD](PRD.md) e na [RFC](RFC.md#contexto-e-problema); esta seção foca no que já foi decidido tecnicamente e no que precisa ser construído em cima disso.

## 2. Objetivos Técnicos

- Inserir o evento de webhook na `webhook_outbox` sem adicionar I/O de rede à transação de `changeStatus` (ADR-001).
- Processar a entrega em um processo desacoplado (`src/worker.ts`), sem competir por recursos com a API HTTP (ADR-002).
- Garantir que cada tentativa de entrega seja autenticável pelo cliente via HMAC-SHA256 (ADR-005) e identificável de forma estável via `event_id` (ADR-006).
- Esgotar falhas de forma previsível (retry + backoff) e nunca perder um evento silenciosamente — todo evento termina em `entregue` ou em `webhook_dead_letter` (ADR-003, ADR-004).
- Não introduzir nenhuma abstração, biblioteca de log/erro ou convenção que não exista hoje no projeto (ADR-007).

## 3. Escopo e Exclusões

**Neste documento:**

- Modelagem de dados das tabelas novas.
- Fluxos de criação de evento, processamento pelo worker, retry e DLQ.
- Contratos HTTP dos 7 endpoints do módulo (CRUD de configuração, rotação de secret, histórico de entregas, replay administrativo).
- Matriz de erros do módulo (`WEBHOOK_*`).
- Estratégias de resiliência, observabilidade, dependências e critérios de aceite técnicos.

**Fora de escopo** (idêntico à RFC, ver [RFC — Escopo](RFC.md#escopo), sem repetir a justificativa de produto aqui):

- Webhooks inbound (cliente → plataforma).
- Alerta por e-mail em falhas repetidas.
- Dashboard visual para o cliente.
- Rate limiting de envio (observação futura, não implementado nesta fase).
- Arquivamento automático de eventos entregues na outbox.
- Tracing distribuído (não existe infraestrutura de tracing no projeto hoje; ver seção [Observabilidade](#9-observabilidade)).

## 4. Modelagem de Dados

Todas as tabelas novas seguem a convenção de `id` UUID (`String @id @default(uuid()) @db.Char(36)`) já usada em `User`, `Customer`, `Product`, `Order`, `OrderItem`, `OrderStatusHistory` (`prisma/schema.prisma`).

### `webhook_subscription` (configuração do webhook por customer)

| Campo | Tipo | Observação |
| --- | --- | --- |
| `id` | UUID | PK |
| `customerId` | UUID | FK para `Customer` |
| `url` | String | HTTPS obrigatório (validação Zod — `[09:23] Sofia`, não é decisão arquitetural própria) |
| `secretHash` / `secret` atual | String | secret vigente (ver nota de segurança abaixo) |
| `previousSecret` | String? | secret anterior, válida durante o grace period de 24h (ADR-005) |
| `previousSecretExpiresAt` | DateTime? | quando a secret anterior expira |
| `events` | JSON / tabela associativa | lista de `OrderStatus` que o endpoint quer receber (`[09:33] Marcos`) |
| `active` | Boolean | estado ativo/inativo (`[09:21] Bruno`) |
| `createdAt`, `updatedAt` | DateTime | padrão do projeto |

**Nota de segurança (decisão desta FDD, não literal da reunião):** a secret em si precisa estar disponível em texto puro no momento da assinatura HMAC pelo worker, então não pode ser armazenada apenas como hash (diferente de senha de usuário). Recomenda-se armazenamento cifrado em repouso (encryption at rest no nível de coluna ou do banco), decisão de infraestrutura que a revisão de segurança da Sofia deve validar antes do deploy (`[09:46] Sofia`, já registrada como bloqueio de cronograma na [RFC](RFC.md#plano-de-implementação)).

### `webhook_outbox` (ADR-001)

| Campo | Tipo | Observação |
| --- | --- | --- |
| `id` | UUID | PK |
| `eventId` | UUID | valor enviado em `X-Event-Id`; gerado uma única vez, estável entre retries (ADR-006) |
| `subscriptionId` | UUID | FK para `webhook_subscription` |
| `eventType` | String | ex. `order.status_changed` (`[09:43] Diego`) |
| `payload` | JSON | snapshot renderizado no momento da inserção (`[09:52] Larissa`/`Diego`) |
| `status` | Enum | `PENDENTE` \| `PROCESSANDO` \| `ENTREGUE` \| `FALHOU` (`[09:08] Diego`) |
| `attemptCount` | Int | quantas tentativas já ocorreram (para decidir o próximo backoff, ADR-003) |
| `nextAttemptAt` | DateTime | quando a próxima tentativa pode ocorrer (implementa o backoff) |
| `requestId` | String? | `requestId` da requisição que mudou o status (`request-logger.middleware.ts`), para correlação de logs — decisão desta FDD |
| `createdAt` | DateTime | usado para ordenação do polling (`[09:12] Diego`, ordem por `order_id`) |

Índices em `status` e `createdAt`, conforme decidido: `[09:08] Diego: A tabela tem índice no campo de status (pendente, processando, falhou, entregue) e em created_at.`

### `webhook_dead_letter` (ADR-004)

| Campo | Tipo | Observação |
| --- | --- | --- |
| `id` | UUID | PK — é o `:id` usado em `POST /admin/webhooks/dead-letter/:id/replay` |
| `outboxEventId` | UUID | referência ao evento original |
| `payload` | JSON | payload que falhou |
| `failureReason` | String | motivo da última falha (`[09:18] Diego`) |
| `failedAt` | DateTime | timestamp da falha definitiva |
| `replayedAt` | DateTime? | preenchido quando um admin reprocessa |
| `replayedByUserId` | UUID? | auditoria do replay (`[09:36] Sofia`) |

### `webhook_delivery` — decisão de modelagem desta FDD

A transcrição pede explicitamente um histórico de entregas por webhook (`[09:34] Marcos: "esses são os últimos 100 webhooks que vocês mandaram pra mim, sucesso/falha, payload, response, tempo de resposta"`), mas **não nomeia uma tabela específica** para isso. Para atender ao requisito, esta FDD introduz `webhook_delivery` como log de cada tentativa de entrega (sucesso ou falha), distinta da `webhook_outbox` (que representa o estado atual do evento, não o histórico de tentativas):

| Campo | Tipo | Observação |
| --- | --- | --- |
| `id` | UUID | PK |
| `outboxEventId` | UUID | referência ao evento |
| `subscriptionId` | UUID | referência ao webhook |
| `attemptNumber` | Int | 1 a 5 (ADR-003) |
| `success` | Boolean | |
| `httpStatusCode` | Int? | nulo em caso de timeout/erro de rede |
| `responseTimeMs` | Int | tempo de resposta (requisito explícito de Marcos) |
| `responseBodySnippet` | String? | truncado para evitar payloads grandes no log |
| `attemptedAt` | DateTime | |

## 5. Fluxos Detalhados

### 5.1 Criação do evento na outbox

1. `PATCH /api/v1/orders/:id/status` chama `OrderController.changeStatus` → `OrderService.changeStatus`.
2. Dentro da mesma `this.prisma.$transaction`, após `tx.orderStatusHistory.create` (`order.service.ts`), o service chama uma nova função pura proposta na própria reunião:

   > [09:41] Bruno: Vai me obrigar a passar um repository do webhook pro OrderService ou uma função de "enqueue event". Vou propor uma função publishWebhookEvent(tx, order, fromStatus, toStatus) que aceita o tx client da transação atual. Aí o order.service chama isso.

   `publishWebhookEvent(tx, order, fromStatus, toStatus)` (proposta em `src/modules/webhooks/webhook.events.ts` ou arquivo equivalente dentro do novo módulo):
   - Busca em `webhook_subscription` os endpoints `active = true` do `customerId` do pedido cujo `events` contém `toStatus`.
   - Se nenhum endpoint quiser aquele status, **não insere nada** (`[09:33]–[09:34] Bruno/Diego`).
   - Para cada endpoint encontrado, monta o payload snapshot (ver [Contratos Públicos](#6-contratos-públicos)) e insere uma linha em `webhook_outbox` com `status = PENDENTE`, `eventId = randomUUID()`.
   - Se a inserção falhar, a exceção propaga e a transação inteira reverte (`[09:40]–[09:41] Bruno/Diego`).

### 5.2 Processamento pelo worker

1. `src/worker.ts` inicia um loop (`setInterval` ou `while` com `await sleep(2000)`), reaproveitando `createPrismaClient()` (`src/config/database.ts`) e `logger` (`src/shared/logger/index.ts`).
2. A cada ciclo de 2 segundos (ADR-002), o worker seleciona um lote de eventos com `status = PENDENTE` (ou `FALHOU` com `nextAttemptAt <= now()`), ordenados por `createdAt`, e marca-os como `PROCESSANDO` antes de processar (evita reprocessamento duplo se o ciclo demorar mais que 2s).
3. Para cada evento: monta os headers (ver contrato de payload/headers abaixo), assina com HMAC-SHA256 usando a secret vigente da `webhook_subscription`, e faz a chamada HTTP com timeout de 10s (`[09:42] Diego`), usando `fetch` nativo do Node (ver [Dependências e Compatibilidade](#10-dependências-e-compatibilidade)).
4. Grava uma linha em `webhook_delivery` com o resultado da tentativa (sucesso/falha, status code, tempo de resposta).
5. Se sucesso (2xx): marca o evento como `ENTREGUE`.
6. Se falha: incrementa `attemptCount`; se `attemptCount < 5`, define `nextAttemptAt` conforme a tabela de backoff (5.3) e marca `status = FALHOU` (aguardando próxima tentativa); se `attemptCount == 5`, move o evento para `webhook_dead_letter` (5.4).

### 5.3 Retry (ADR-003)

Tabela de backoff por `attemptCount`:

| Tentativa | Intervalo até a próxima |
| --- | --- |
| 1 → 2 | 1 minuto |
| 2 → 3 | 5 minutos |
| 3 → 4 | 30 minutos |
| 4 → 5 | 2 horas |
| 5 → DLQ | 12 horas |

> [09:17] Diego: Eu pensei em 1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas. Total de quase 15 horas entre primeira falha e última tentativa.

Cada tentativa usa o **mesmo `eventId`** gerado na criação (ADR-006) — nunca é regenerado.

### 5.4 Dead Letter Queue e replay (ADR-004)

1. Ao esgotar as 5 tentativas, o worker move o registro para `webhook_dead_letter` (copiando payload e motivo da última falha) e remove/marca o evento correspondente na `webhook_outbox` como finalizado (não volta a ser lido pelo polling).
2. Um usuário `ADMIN` chama `POST /api/v1/admin/webhooks/dead-letter/:id/replay`.
3. O controller valida `requireRole('ADMIN')` (`src/middlewares/auth.middleware.ts`), busca o registro em `webhook_dead_letter`, cria um novo evento em `webhook_outbox` com `status = PENDENTE` e `attemptCount = 0` (nova janela completa de retry), preenche `replayedAt`/`replayedByUserId` no registro da DLQ, e loga a ação via Pino com o `userId` do administrador (`[09:36] Sofia`).

## 6. Contratos Públicos

Todos os endpoints do módulo são montados sob `/api/v1`, seguindo o padrão de `buildApiRouter` (`src/routes/index.ts`): `router.use('/webhooks', buildWebhookRouter(controllers.webhooks))` e `router.use('/admin/webhooks', buildWebhookAdminRouter(controllers.webhooksAdmin))`. Exceto o endpoint de replay (que exige `ADMIN`), os demais exigem apenas autenticação (`[09:36]–[09:37] Sofia/Marcos: "por enquanto sim"`), sem exigência de papel específico.

### 6.1 `POST /api/v1/webhooks` — cadastrar webhook

Autenticado; `customerId` vem do body, não do JWT (`[09:32]–[09:33] Larissa/Bruno`).

Request:
```json
{
  "customerId": "3f1b6e2a-1234-4c9e-9a11-abc123456789",
  "url": "https://integrations.atlascomercial.com/hooks/orders",
  "events": ["SHIPPED", "DELIVERED", "CANCELLED"]
}
```

Response `201 Created` (a secret só é exposta em texto puro nesta resposta, decisão de segurança desta FDD):
```json
{
  "id": "9c2e7a10-...",
  "customerId": "3f1b6e2a-1234-4c9e-9a11-abc123456789",
  "url": "https://integrations.atlascomercial.com/hooks/orders",
  "events": ["SHIPPED", "DELIVERED", "CANCELLED"],
  "secret": "whsec_3f8a2b1c9d4e5f6a7b8c9d0e1f2a3b4c",
  "active": true,
  "createdAt": "2026-01-15T13:00:00.000Z"
}
```

Erros possíveis: `400 VALIDATION_ERROR` (URL não HTTPS ou `events` com status inválido), `404 WEBHOOK_CUSTOMER_NOT_FOUND`, `409 WEBHOOK_DUPLICATE_SUBSCRIPTION`.

### 6.2 `GET /api/v1/webhooks?customerId=...` — listar webhooks

Response `200 OK`, no envelope paginado já usado pelo projeto (`paginated()` em `src/shared/http/response.ts`). A secret nunca é retornada em listagem (decisão desta FDD):
```json
{
  "data": [
    {
      "id": "9c2e7a10-...",
      "customerId": "3f1b6e2a-...",
      "url": "https://integrations.atlascomercial.com/hooks/orders",
      "events": ["SHIPPED", "DELIVERED", "CANCELLED"],
      "active": true,
      "createdAt": "2026-01-15T13:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

### 6.3 `PATCH /api/v1/webhooks/:id` — editar webhook

Request (todos os campos opcionais):
```json
{ "url": "https://integrations.atlascomercial.com/hooks/orders/v2", "events": ["DELIVERED"], "active": true }
```

Response `200 OK`: mesmo formato do item 6.2 (sem secret).

Erros: `404 WEBHOOK_NOT_FOUND`, `400 VALIDATION_ERROR`.

### 6.4 `DELETE /api/v1/webhooks/:id` — remover webhook

Response `204 No Content`.

Erros: `404 WEBHOOK_NOT_FOUND`.

### 6.5 `POST /api/v1/webhooks/:id/secret/rotate` — rotacionar secret

> [09:21] Sofia: Sim. E a secret tem que ser rotacionável. Endpoint pro cliente conseguir pedir nova secret pela API. Quando ele rotaciona, a antiga fica válida por 24 horas em paralelo, pra ele ter tempo de migrar os sistemas dele. Depois disso, a antiga morre.

Response `200 OK`:
```json
{
  "id": "9c2e7a10-...",
  "secret": "whsec_novaSecretGeradaAgora...",
  "previousSecretExpiresAt": "2026-01-16T13:05:00.000Z"
}
```

Erros: `404 WEBHOOK_NOT_FOUND`, `409 WEBHOOK_INACTIVE` (rotação de webhook desativado — decisão desta FDD).

### 6.6 `GET /api/v1/webhooks/:id/deliveries` — histórico de entregas

> [09:34] Marcos: Mais um: o cliente precisa conseguir ver o histórico de entregas. Tipo "esses são os últimos 100 webhooks que vocês mandaram pra mim, sucesso/falha, payload, response, tempo de resposta". GET /webhooks/:id/deliveries.

Query: `?page=1&pageSize=100` (`pageSize` padrão e máximo 100, conforme literal da fala de Marcos).

Response `200 OK`, no mesmo envelope `paginated()` de `src/shared/http/response.ts`:
```json
{
  "data": [
  {
    "id": "d41f2e90-...",
    "eventId": "7b6c5d4e-...",
    "success": true,
    "httpStatusCode": 200,
    "responseTimeMs": 842,
    "attemptedAt": "2026-01-15T13:00:02.100Z",
    "payload": { "event_id": "7b6c5d4e-...", "event_type": "order.status_changed", "to_status": "SHIPPED" }
  },
  {
    "id": "d41f2e91-...",
    "eventId": "7b6c5d4e-...",
    "success": false,
    "httpStatusCode": null,
    "responseTimeMs": 10000,
    "attemptedAt": "2026-01-15T12:59:00.000Z",
    "payload": { "event_id": "7b6c5d4e-...", "event_type": "order.status_changed", "to_status": "SHIPPED" }
  }
  ],
  "pagination": { "page": 1, "pageSize": 100, "total": 2, "totalPages": 1 }
}
```

Erros: `404 WEBHOOK_NOT_FOUND`.

### 6.7 `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — replay administrativo

> [09:18] Diego: Manual via endpoint admin. Tipo um POST /admin/webhooks/dead-letter/:id/replay. Recoloca na outbox como pendente.

Requer `requireRole('ADMIN')` (`[09:36] Larissa`).

Response `202 Accepted`:
```json
{
  "deadLetterId": "a1b2c3d4-...",
  "newOutboxEventId": "e5f6a7b8-...",
  "status": "PENDENTE",
  "replayedBy": "3d2c1b0a-userId",
  "replayedAt": "2026-01-16T09:10:00.000Z"
}
```

Erros: `403 FORBIDDEN` (papel diferente de ADMIN, comportamento padrão de `requireRole`), `404 WEBHOOK_DEAD_LETTER_NOT_FOUND`, `409 WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED`.

### Payload e headers da entrega do webhook (worker → cliente)

Payload (`[09:43] Diego` — deliberadamente sem `items`, para não inflar o payload):
```json
{
  "event_id": "7b6c5d4e-8f9a-4b1c-9d2e-1a2b3c4d5e6f",
  "event_type": "order.status_changed",
  "timestamp": "2026-01-15T13:00:00.000Z",
  "order_id": "1a2b3c4d-...",
  "order_number": "ORD-2026-000123",
  "from_status": "PAID",
  "to_status": "SHIPPED",
  "customer_id": "3f1b6e2a-...",
  "total_cents": 15000
}
```

Headers (`[09:44]–[09:45] Diego/Sofia`):

| Header | Conteúdo |
| --- | --- |
| `Content-Type` | `application/json` |
| `X-Event-Id` | UUID do evento (estável entre retries) |
| `X-Signature` | HMAC-SHA256 do corpo, hex ou base64 (decisão de encoding desta FDD: hex) |
| `X-Timestamp` | timestamp ISO 8601 do envio |
| `X-Webhook-Id` | id da `webhook_subscription` que originou o envio |

Limite de payload: **64KB**; se excedido, o envio **não é feito** (erro, não truncamento) — `[09:23]–[09:24] Sofia/Diego/Larissa`.

## 7. Matriz de Erros (`WEBHOOK_*`)

Segue o padrão de `AppError` (`src/shared/errors/app-error.ts`): `statusCode`, `errorCode`, `details`. Códigos citados literalmente por Bruno (`WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`) têm sua condição de disparo detalhada nesta FDD, já que a reunião só deu o padrão de nomenclatura, não a lista fechada de códigos:

| Código | HTTP | Quando ocorre |
| --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | `id` de `webhook_subscription` não encontrado (GET/PATCH/DELETE/rotate/deliveries) |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `customerId` informado no cadastro não existe |
| `WEBHOOK_DUPLICATE_SUBSCRIPTION` | 409 | já existe cadastro com a mesma combinação `customerId` + `url` |
| `WEBHOOK_INVALID_URL` | 422 | URL passa na validação de formato do Zod mas falha em `new URL()` no service, ou host normalizado colide com outro cadastro |
| `WEBHOOK_SECRET_REQUIRED` | 400 | operação que depende de secret ativa é chamada sobre um registro sem secret configurada (estado inconsistente) |
| `WEBHOOK_INACTIVE` | 409 | operação de rotação/edição sensível tentada sobre webhook com `active = false` |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | payload renderizado do evento excede 64KB (`[09:23]-[09:24]`) |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | `id` informado no replay não existe em `webhook_dead_letter` |
| `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | 409 | replay de um registro de DLQ já reprocessado anteriormente |

`ZodError` (validação de schema, ex. URL não-HTTPS, `events` com valor fora do enum `OrderStatus`) continua caindo no tratamento genérico existente (`VALIDATION_ERROR`, 400) do middleware `src/middlewares/error.middleware.ts` — nenhuma mudança necessária nesse middleware.

## 8. Estratégias de Resiliência

- **Timeout por tentativa:** 10 segundos (`[09:42] Diego`) — chamada HTTP do worker é abortada (`AbortController`) se exceder esse limite; conta como falha para fins de retry.
- **Retry com backoff exponencial:** 5 tentativas, intervalos 1m/5m/30m/2h/12h (ADR-003, seção 5.3).
- **Fallback para falha permanente:** Dead Letter Queue com replay manual restrito a ADMIN (ADR-004).
- **Idempotência do lado do cliente:** `event_id` estável entre tentativas permite deduplicação (ADR-006); a plataforma não implementa exactly-once.
- **Proteção contra reprocessamento duplo pelo worker:** o worker marca o lote como `PROCESSANDO` antes de iniciar as chamadas HTTP, evitando que um ciclo de polling lento faça o próximo ciclo pegar os mesmos eventos (decisão desta FDD, necessária para operacionalizar o polling do ADR-002, que assume single-worker).
- **Rotação de secret sem downtime:** duas secrets válidas em paralelo durante 24h (ADR-005) evita janela de falha de autenticação durante a migração do cliente.
- **Payload grande:** rejeitado (erro), não truncado — decisão explícita da reunião para evitar payloads inconsistentes (`[09:23]-[09:24]`).

## 9. Observabilidade

**Métricas** (sugeridas; o projeto não tem hoje um client de métricas como `prom-client` — introduzir uma dependência de métricas é uma decisão de infraestrutura fora do escopo desta FDD, registrada aqui como recomendação):
- Contagem de eventos por status (`PENDENTE`, `PROCESSANDO`, `ENTREGUE`, `FALHOU`) na `webhook_outbox`.
- Taxa de sucesso de entrega (por `webhook_subscription` e agregada).
- Latência de entrega (da inserção na outbox até `ENTREGUE`).
- Tamanho da fila de pendentes (para detectar acúmulo/atraso do worker).
- Quantidade de itens em `webhook_dead_letter` (alerta operacional de falhas persistentes).

**Logs:** usar o logger Pino já existente (`src/shared/logger/index.ts`), sem nova biblioteca (ADR-007). Cada tentativa de entrega deve logar `eventId`, `subscriptionId`, `customerId`, `attemptNumber`, `httpStatusCode`, `durationMs`, seguindo o mesmo padrão estruturado de `request-logger.middleware.ts` e `error.middleware.ts`. O replay administrativo deve logar o `userId` de quem executou a ação (`[09:36] Sofia`, requisito explícito de auditoria).

**Tracing:** o projeto **não possui hoje** nenhuma instrumentação de tracing distribuído (não há OpenTelemetry ou equivalente no código-fonte examinado), e a transcrição não discute o tema. Nesta fase, a rastreabilidade fim-a-fim de um evento (da inserção na outbox até a entrega ou DLQ) é feita por **correlação via `event_id`** em todos os logs relacionados — não por tracing distribuído formal. Para ligar o evento à requisição HTTP que o originou, o `requestId` já gerado por `src/middlewares/request-logger.middleware.ts` (header `X-Request-Id`) deve ser gravado junto ao evento e repetido nos logs do worker, fechando a cadeia requisição → `changeStatus` → outbox → tentativas de entrega. Isso é registrado aqui como uma limitação conhecida, não uma decisão de arquitetura.

## 10. Dependências e Compatibilidade

- Stack reaproveitada sem mudança de versão: Node ≥20, TypeScript, Express 4.21, Prisma 5.22 (MySQL), Zod 3.23.8, Pino 9.5.0 (ADR-007).
- **Nenhuma nova dependência de runtime é estritamente necessária:** o worker pode usar `fetch` nativo do Node (disponível desde o Node 18) para as chamadas HTTP de entrega, em vez de adicionar `axios`/`undici` como dependência nova — decisão desta FDD alinhada ao princípio de reuso da ADR-007, já que o `package.json` atual não tem nenhum cliente HTTP como dependência.
- Nova migration Prisma aditiva (não quebra o schema existente) para `webhook_subscription`, `webhook_outbox`, `webhook_dead_letter`, `webhook_delivery`.
- Novo script em `package.json` (`"worker": "tsx watch --env-file=.env src/worker.ts"` em dev, e entrada equivalente no build via `tsconfig.build.json`), espelhando o script existente do `server.ts`.
- Compatibilidade: o worker usa sua própria instância de `PrismaClient` (via `createPrismaClient()`, `src/config/database.ts`) apontando para a mesma `DATABASE_URL` da API — nenhuma mudança de infraestrutura de banco é necessária.

## 11. Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Worker como processo único é ponto único de falha para toda a entrega de webhooks | Média | Alto | Healthcheck e restart automático via orquestrador de deploy; limitação conhecida e aceita nesta fase (ADR-002) |
| Contenção de leitura na `webhook_outbox` em alto volume de pedidos | Baixa (volume atual) | Médio | Índices em `status` e `created_at` já definidos na reunião (`[09:08] Diego`); reavaliar se volume crescer |
| Vazamento de secret do lado do sistema do cliente (já ocorreu uma vez) | Média | Alto | Rotação com grace period de 24h (ADR-005); risco residual documentado, não totalmente mitigável pela plataforma |
| Payload de evento cresce além de 64KB por mudança futura de modelo de dados | Baixa | Médio | Erro explícito (`WEBHOOK_PAYLOAD_TOO_LARGE`) em vez de truncar, evitando entrega de dado incompleto silenciosamente |
| Ausência de tracing distribuído dificulta diagnóstico de falhas entre processos (API, worker, cliente externo) | Alta | Baixo/Médio | Correlação manual via `event_id` em todos os logs (ver [Observabilidade](#9-observabilidade)); aceito como limitação desta fase |

## 12. Integração com o Sistema Existente

- **`src/modules/orders/order.service.ts`** — o método `changeStatus` é estendido para chamar `publishWebhookEvent(tx, order, fromStatus, toStatus)` dentro da mesma `this.prisma.$transaction`, logo após `tx.orderStatusHistory.create` (`[09:40]-[09:41] Bruno/Diego`). Nenhuma outra lógica do método muda; a função recebe o `tx` da transação em andamento, não abre transação própria.
- **`src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts`** — todas as classes de erro do novo módulo (`WebhookNotFoundError`, `WebhookInvalidUrlError`, etc., seção 7) estendem `AppError`, no mesmo padrão de `InsufficientStockError`/`InvalidStatusTransitionError` já existentes, com códigos prefixados `WEBHOOK_` (`[09:28]-[09:29] Bruno/Larissa`).
- **`src/middlewares/error.middleware.ts`** — nenhuma alteração de código é necessária: o middleware já trata qualquer `instanceof AppError` (novas classes inclusas automaticamente), `ZodError` e `Prisma.PrismaClientKnownRequestError`.
- **`src/middlewares/auth.middleware.ts`** — o endpoint `POST /api/v1/admin/webhooks/dead-letter/:id/replay` reaproveita `requireRole('ADMIN')` exatamente no padrão já usado em `src/modules/users/user.routes.ts`, sem alteração no middleware.
- **`src/routes/index.ts`** — `buildApiRouter` ganha dois novos mounts (`/webhooks` e `/admin/webhooks`), seguindo o mesmo padrão de `router.use('/orders', buildOrderRouter(controllers.orders))` já existente.
- **`src/config/database.ts`** — `src/worker.ts` reutiliza a função `createPrismaClient()` para abrir sua própria instância de `PrismaClient`, em vez de compartilhar o singleton `prisma` usado pela API (`[09:29]-[09:30] Diego/Bruno`).
- **`src/shared/logger/index.ts`** — o worker e todos os componentes do módulo de webhooks usam a mesma instância `logger` (Pino) já exportada, sem nova biblioteca de logging (`[09:29] Bruno`).
- **`src/shared/http/response.ts`** — os endpoints de listagem (`GET /webhooks` e `GET /webhooks/:id/deliveries`) retornam no envelope de `paginated()`/`buildPagination()` já existente, sem criar um formato novo de paginação.
- **`src/middlewares/request-logger.middleware.ts`** — o `requestId` que o middleware já gera e devolve em `X-Request-Id` é reaproveitado como chave de correlação entre a requisição de mudança de status e os logs do worker.
- **`prisma/schema.prisma`** — recebe os novos modelos `WebhookSubscription`, `WebhookOutbox`, `WebhookDeadLetter` e `WebhookDelivery`, todos seguindo a convenção `String @id @default(uuid()) @db.Char(36)` já usada nos modelos existentes.

## 13. Critérios de Aceite Técnicos

- [ ] A inserção do evento na `webhook_outbox` ocorre estritamente dentro da transação de `changeStatus`; testes de integração confirmam que rollback da transação principal também reverte a inserção do evento (ADR-001).
- [ ] O worker processa eventos pendentes em ciclos de até 2 segundos e nunca processa o mesmo evento pendente duas vezes simultaneamente (ADR-002).
- [ ] Uma tentativa de entrega que excede 10 segundos é tratada como falha e agenda a próxima tentativa conforme a tabela de backoff da seção 5.3 (ADR-003).
- [ ] Após a 5ª tentativa falha, o evento aparece em `webhook_dead_letter` e não é mais lido pelo polling ativo da outbox (ADR-004).
- [ ] `POST /admin/webhooks/dead-letter/:id/replay` retorna 403 para qualquer papel diferente de `ADMIN` e loga o `userId` em caso de sucesso (ADR-004).
- [ ] Toda entrega inclui `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`; a assinatura é verificável recalculando HMAC-SHA256 com a secret vigente (ADR-005).
- [ ] O `event_id` de um evento é idêntico em todas as suas tentativas de retry (ADR-006).
- [ ] Rotação de secret mantém a secret anterior válida por exatamente 24h e a invalida após esse período (ADR-005).
- [ ] O payload de entrega nunca contém o campo `items` e nunca excede 64KB; exceder o limite produz `WEBHOOK_PAYLOAD_TOO_LARGE` sem tentativa de envio (`[09:23]-[09:24]`).
- [ ] Cadastro de webhook com URL não-HTTPS é rejeitado com `400 VALIDATION_ERROR` antes de qualquer persistência.
- [ ] O filtro de eventos é aplicado na inserção da outbox (nenhuma linha é inserida para um status que nenhum webhook do customer deseja), não no momento do envio (`[09:33]-[09:34]`).
- [ ] `GET /webhooks/:id/deliveries` retorna no máximo 100 registros, ordenados do mais recente para o mais antigo.

## Limitações registradas

- A transcrição não especifica o encoding exato da assinatura em `X-Signature` (hex vs. base64) nem o algoritmo de geração da secret (comprimento, charset); esta FDD adota hex e um formato `whsec_<hex>` como exemplo ilustrativo — ambos são decisões de implementação, não decisões da reunião, e devem ser confirmadas na revisão de segurança da Sofia antes do deploy (`[09:46] Sofia`).
- A tabela `webhook_delivery` (seção 4) é uma decisão de modelagem desta FDD, não uma tabela nomeada literalmente na transcrição — foi introduzida para atender ao requisito funcional explícito de histórico de entregas (`[09:34] Marcos`).
- Métricas e tracing (seção 9) são recomendações desta FDD; a reunião não discutiu observabilidade, e o projeto não tem hoje infraestrutura de métricas ou tracing distribuído.
