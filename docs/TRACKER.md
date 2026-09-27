# Tracker de Rastreabilidade

Mapeia cada item identificável nos documentos do pacote (`docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/adrs/*.md`) à sua origem: um timestamp + falante em `TRANSCRICAO.md`, ou um caminho de arquivo real no código-fonte. Uma decisão registrada em mais de um documento (ex.: uma ADR referenciada pela RFC e pelo PRD) aparece uma única vez, no documento onde a decisão é originalmente registrada, para evitar linhas redundantes.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | docs/PRD.md | Restrição/Contexto | Três clientes B2B pedem notificação em tempo real; Atlas ameaça migrar | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Restrição | "Tempo real" quantificado como abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-PUB-01 | docs/PRD.md | Restrição | Cadastro de webhook via usuário autenticado; customer_id não vem do JWT | TRANSCRICAO | [09:32] Larissa |
| PRD-OBJ-01 | docs/PRD.md | Requisito Não Funcional | Meta de latência de notificação inferior a 10s | TRANSCRICAO | [09:10] Larissa |
| PRD-OBJ-02 | docs/PRD.md | Requisito Não Funcional | Meta de 5 tentativas cobrindo ~15h antes de intervenção manual | TRANSCRICAO | [09:17] Diego |
| PRD-OBJ-03 | docs/PRD.md | Decisão | Meta de entrega até fim de novembro / 3 sprints | TRANSCRICAO | [09:45] Marcos |
| PRD-OBJ-04 | docs/PRD.md | Requisito Não Funcional | Meta de HMAC-SHA256 em 100% das entregas | TRANSCRICAO | [09:20] Sofia |
| PRD-FORA-01 | docs/PRD.md | Restrição | E-mail de alerta ao cliente fora de escopo, adiado | TRANSCRICAO | [09:37] Larissa |
| PRD-FORA-02 | docs/PRD.md | Restrição | Dashboard visual fora de escopo | TRANSCRICAO | [09:40] Larissa |
| PRD-FORA-03 | docs/PRD.md | Restrição | Webhooks inbound fora de escopo | TRANSCRICAO | [09:02] Marcos |
| PRD-FORA-04 | docs/PRD.md | Restrição | Rate limiting de envio não decidido, adiado | TRANSCRICAO | [09:39] Larissa |
| PRD-FORA-05 | docs/PRD.md | Restrição | Arquivamento automático de eventos entregues fora de escopo | TRANSCRICAO | [09:08] Diego |
| PRD-FORA-06 | docs/PRD.md | Restrição | Restrição de papel no CRUD de webhooks adiada | TRANSCRICAO | [09:37] Sofia |
| PRD-RF-01 | docs/PRD.md | Requisito Funcional | Cadastrar webhook (URL, status, secret gerada e devolvida) | TRANSCRICAO | [09:31] Marcos |
| PRD-RF-02 | docs/PRD.md | Requisito Funcional | Editar webhook cadastrado | TRANSCRICAO | [09:33] Bruno |
| PRD-RF-03 | docs/PRD.md | Requisito Funcional | Remover webhook cadastrado | TRANSCRICAO | [09:33] Bruno |
| PRD-RF-04 | docs/PRD.md | Requisito Funcional | Listar webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| PRD-RF-05 | docs/PRD.md | Requisito Funcional | Filtro de eventos por status, aplicado na inserção da outbox | TRANSCRICAO | [09:34] Diego |
| PRD-RF-06 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas por webhook | TRANSCRICAO | [09:34] Marcos |
| PRD-RF-07 | docs/PRD.md | Requisito Funcional | Reprocessamento manual restrito a ADMIN, com auditoria | TRANSCRICAO | [09:36] Larissa |
| PRD-RF-08 | docs/PRD.md | Requisito Funcional | Assinatura HMAC-SHA256 de cada notificação | TRANSCRICAO | [09:20] Sofia |
| PRD-RF-09 | docs/PRD.md | Requisito Funcional | Rotação de secret pela API com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-RF-10 | docs/PRD.md | Requisito Funcional | Entrega at-least-once com identificador para dedup | TRANSCRICAO | [09:26] Larissa |
| PRD-RF-11 | docs/PRD.md | Requisito Funcional | Retry automático antes de falha permanente | TRANSCRICAO | [09:17] Larissa |
| PRD-RF-12 | docs/PRD.md | Requisito Funcional | Payload da notificação com dados básicos do pedido, sem itens | TRANSCRICAO | [09:43] Diego |
| PRD-RNF-01 | docs/PRD.md | Requisito Não Funcional | Latência de notificação inferior a 10s no caso comum | TRANSCRICAO | [09:10] Larissa |
| PRD-RNF-02 | docs/PRD.md | Requisito Não Funcional | Timeout de 10s por tentativa de entrega | TRANSCRICAO | [09:42] Diego |
| PRD-RNF-03 | docs/PRD.md | Requisito Não Funcional | Consistência transacional entre status e evento | TRANSCRICAO | [09:41] Diego |
| PRD-RNF-04 | docs/PRD.md | Requisito Não Funcional | URLs de webhook obrigatoriamente HTTPS | TRANSCRICAO | [09:23] Sofia |
| PRD-RNF-05 | docs/PRD.md | Requisito Não Funcional | Secret exclusiva por endpoint, nunca global | TRANSCRICAO | [09:21] Sofia |
| PRD-RNF-06 | docs/PRD.md | Requisito Não Funcional | Payload limitado a 64KB, rejeitado se exceder | TRANSCRICAO | [09:24] Larissa |
| PRD-RNF-07 | docs/PRD.md | Requisito Não Funcional | Processo de entrega independente do processo da API | TRANSCRICAO | [09:11] Diego |
| PRD-RNF-08 | docs/PRD.md | Requisito Não Funcional | Reprocessamento manual deve ser auditável | TRANSCRICAO | [09:36] Sofia |
| PRD-DEC-01 | docs/PRD.md | Trade-off | Entrega assíncrona (Outbox) em vez de síncrona | TRANSCRICAO | [09:08] Larissa |
| PRD-DEC-02 | docs/PRD.md | Trade-off | Retry limitado (5 tentativas) em vez de indefinido | TRANSCRICAO | [09:17] Larissa |
| PRD-DEC-03 | docs/PRD.md | Trade-off | At-least-once em vez de exactly-once | TRANSCRICAO | [09:26] Larissa |
| PRD-DEC-04 | docs/PRD.md | Trade-off | Secret por endpoint com rotação em vez de secret fixa/global | TRANSCRICAO | [09:22] Sofia |
| PRD-DEC-05 | docs/PRD.md | Trade-off | Reuso total dos padrões do projeto | TRANSCRICAO | [09:30] Larissa |
| PRD-DEP-01 | docs/PRD.md | Dependência | Depende da transação de `OrderService.changeStatus` | CODIGO | src/modules/orders/order.service.ts |
| PRD-DEP-02 | docs/PRD.md | Dependência | Depende dos modelos `Customer`/`User` já existentes | CODIGO | prisma/schema.prisma |
| PRD-DEP-03 | docs/PRD.md | Dependência | Revisão de segurança de 2 dias antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-04 | docs/PRD.md | Dependência | Comunicação do modelo de dedup no portal de desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| PRD-DEP-05 | docs/PRD.md | Dependência | Nenhuma infraestrutura nova necessária (reaproveita MySQL) | TRANSCRICAO | [09:07] Diego |
| PRD-DEP-06 | docs/PRD.md | Dependência | Prazo comercial de fim de novembro já comunicado à Atlas | TRANSCRICAO | [09:45] Marcos |
| PRD-RISK-01 | docs/PRD.md | Risco | Perda de cliente por atraso na entrega | TRANSCRICAO | [09:00] Marcos |
| PRD-RISK-02 | docs/PRD.md | Risco | Vazamento de secret do lado do cliente (incidente real) | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-03 | docs/PRD.md | Risco | Indisponibilidade prolongada do cliente | TRANSCRICAO | [09:16] Diego |
| PRD-RISK-04 | docs/PRD.md | Risco | Worker único como ponto único de falha operacional | TRANSCRICAO | [09:13] Diego |
| PRD-AC-01 | docs/PRD.md | Critério de Aceitação | CRUD completo de webhooks por customer | TRANSCRICAO | [09:33] Bruno |
| PRD-AC-02 | docs/PRD.md | Critério de Aceitação | Notificação entregue em até 10s no caso comum | TRANSCRICAO | [09:10] Larissa |
| PRD-AC-03 | docs/PRD.md | Critério de Aceitação | Notificação assinada e verificável via HMAC | TRANSCRICAO | [09:20] Sofia |
| PRD-AC-04 | docs/PRD.md | Critério de Aceitação | Falha de entrega retentada automaticamente | TRANSCRICAO | [09:15] Diego |
| PRD-AC-05 | docs/PRD.md | Critério de Aceitação | Reprocessamento manual auditado por ADMIN | TRANSCRICAO | [09:36] Sofia |
| PRD-AC-06 | docs/PRD.md | Critério de Aceitação | Rotação de secret sem indisponibilidade de validação | TRANSCRICAO | [09:21] Sofia |
| PRD-AC-07 | docs/PRD.md | Critério de Aceitação | Consulta de histórico de até 100 entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-AC-08 | docs/PRD.md | Critério de Aceitação | Atomicidade entre mudança de status e evento | TRANSCRICAO | [09:41] Diego |
| PRD-TEST-01 | docs/PRD.md | Restrição | Testes unitários seguem padrão de testes já existente | CODIGO | tests/orders.test.ts |
| PRD-TEST-02 | docs/PRD.md | Restrição | Teste de integração de atomicidade da transação | TRANSCRICAO | [09:40] Bruno |
| PRD-TEST-03 | docs/PRD.md | Restrição | Testes ponta a ponta estimados dentro do plano de sprints | TRANSCRICAO | [09:46] Larissa |
| RFC-CTX-01 | docs/RFC.md | Contexto | Contexto de negócio: pedido formal dos 3 clientes B2B | TRANSCRICAO | [09:00] Marcos |
| RFC-CTX-02 | docs/RFC.md | Restrição | Escopo delimitado como exclusivamente outbound | TRANSCRICAO | [09:03] Sofia |
| RFC-PROP-01 | docs/RFC.md | Decisão | Integração dentro da transação de `changeStatus` | TRANSCRICAO | [09:40] Bruno |
| RFC-PROP-02 | docs/RFC.md | Decisão | Ponto de integração confirmado no código existente | CODIGO | src/modules/orders/order.service.ts |
| RFC-ALT-01 | docs/RFC.md | Trade-off | Disparo síncrono descartado | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Trade-off | Fila externa dedicada (Redis Streams) descartada | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Trade-off | Garantia exactly-once descartada | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-04 | docs/RFC.md | Trade-off | Secret HMAC global descartada | TRANSCRICAO | [09:21] Sofia |
| RFC-ALT-05 | docs/RFC.md | Trade-off | Trigger de banco para entrega reativa descartado | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-06 | docs/RFC.md | Trade-off | Teto de 3 tentativas de retry descartado | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-07 | docs/RFC.md | Trade-off | Retry indefinido descartado | TRANSCRICAO | [09:15] Diego |
| RFC-OPEN-01 | docs/RFC.md | Restrição | Rate limiting de envio não decidido | TRANSCRICAO | [09:39] Larissa |
| RFC-OPEN-02 | docs/RFC.md | Restrição | Ordenação global não garantida ao escalar workers | TRANSCRICAO | [09:13] Diego |
| RFC-OPEN-03 | docs/RFC.md | Restrição | Política de arquivamento de eventos não fechada | TRANSCRICAO | [09:08] Diego |
| RFC-OPEN-04 | docs/RFC.md | Restrição | Endurecimento da autorização do CRUD não decidido | TRANSCRICAO | [09:37] Sofia |
| RFC-IMP-01 | docs/RFC.md | Risco | Novo processo em produção (worker) | TRANSCRICAO | [09:11] Diego |
| RFC-IMP-02 | docs/RFC.md | Risco | Risco de segurança conhecido (vazamento de secret) | TRANSCRICAO | [09:22] Diego |
| RFC-IMP-03 | docs/RFC.md | Restrição | Revisão de segurança como bloqueio de cronograma | TRANSCRICAO | [09:46] Sofia |
| RFC-IMP-04 | docs/RFC.md | Risco | Risco de negócio: perda de cliente por atraso | TRANSCRICAO | [09:00] Marcos |
| RFC-PLAN-01 | docs/RFC.md | Decisão | Estimativa de 3 sprints por bloco de trabalho | TRANSCRICAO | [09:46] Larissa |
| RFC-PLAN-02 | docs/RFC.md | Restrição | Prazo alvo de fim de novembro | TRANSCRICAO | [09:45] Marcos |
| FDD-DATA-01 | docs/FDD.md | Decisão | Campos de configuração e rotação de secret na tabela de webhook | TRANSCRICAO | [09:21] Sofia |
| FDD-DATA-02 | docs/FDD.md | Decisão | Índices de status e created_at na outbox | TRANSCRICAO | [09:08] Diego |
| FDD-DATA-03 | docs/FDD.md | Decisão | Tabela separada para Dead Letter Queue | TRANSCRICAO | [09:18] Diego |
| FDD-DATA-04 | docs/FDD.md | Restrição | Tabela de histórico de entregas (nome é decisão desta FDD) | TRANSCRICAO | [09:34] Marcos |
| FDD-FLOW-01 | docs/FDD.md | Decisão | Função `publishWebhookEvent(tx, order, fromStatus, toStatus)` | TRANSCRICAO | [09:41] Bruno |
| FDD-FLOW-02 | docs/FDD.md | Decisão | Worker em loop de polling a cada 2 segundos | TRANSCRICAO | [09:09] Diego |
| FDD-FLOW-03 | docs/FDD.md | Decisão | Tabela de progressão de backoff (1m/5m/30m/2h/12h) | TRANSCRICAO | [09:17] Diego |
| FDD-FLOW-04 | docs/FDD.md | Decisão | Fluxo de replay manual da DLQ | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-01 | docs/FDD.md | Requisito Funcional | `POST /webhooks` — cadastrar webhook | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Requisito Funcional | `GET /webhooks` — listar webhooks | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Requisito Funcional | `PATCH /webhooks/:id` — editar webhook | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Requisito Funcional | `DELETE /webhooks/:id` — remover webhook | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Requisito Funcional | `POST /webhooks/:id/secret/rotate` — rotacionar secret | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Requisito Funcional | `GET /webhooks/:id/deliveries` — histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Requisito Funcional | `POST /admin/webhooks/dead-letter/:id/replay` | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Restrição | Campos do payload do evento (sem `items`) | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-09 | docs/FDD.md | Restrição | Headers da entrega (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`) | TRANSCRICAO | [09:44] Diego |
| FDD-ERR-01 | docs/FDD.md | Restrição | Código `WEBHOOK_NOT_FOUND` (padrão citado literalmente) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Restrição | Código `WEBHOOK_INVALID_URL` (padrão citado literalmente) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-03 | docs/FDD.md | Restrição | Código `WEBHOOK_SECRET_REQUIRED` (padrão citado literalmente) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-04 | docs/FDD.md | Restrição | Prefixo `WEBHOOK_` obrigatório para códigos do módulo | TRANSCRICAO | [09:29] Larissa |
| FDD-ERR-05 | docs/FDD.md | Restrição | Códigos adicionais seguem o padrão de `AppError` existente | CODIGO | src/shared/errors/http-errors.ts |
| FDD-ERR-06 | docs/FDD.md | Restrição | `WEBHOOK_PAYLOAD_TOO_LARGE` (limite de 64KB) | TRANSCRICAO | [09:24] Larissa |
| FDD-RES-01 | docs/FDD.md | Decisão | Marcação `PROCESSANDO` para evitar reprocessamento duplo | TRANSCRICAO | [09:09] Diego |
| FDD-OBS-01 | docs/FDD.md | Restrição | Logging via Pino já existente, sem nova biblioteca | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-02 | docs/FDD.md | Restrição | Replay administrativo deve logar o usuário responsável | TRANSCRICAO | [09:36] Sofia |
| FDD-DEP-01 | docs/FDD.md | Dependência | Stack reaproveitada sem mudança de versão | CODIGO | package.json |
| FDD-DEP-02 | docs/FDD.md | Restrição | Ausência de cliente HTTP como dependência atual | CODIGO | package.json |
| FDD-DEP-03 | docs/FDD.md | Decisão | Worker usa `PrismaClient` próprio, não o singleton da API | TRANSCRICAO | [09:30] Bruno |
| FDD-RISK-01 | docs/FDD.md | Restrição | Ausência de infraestrutura de tracing distribuído no projeto | CODIGO | package.json |
| FDD-INT-01 | docs/FDD.md | Restrição | Extensão de `changeStatus` para publicar evento | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Restrição | Novas classes de erro estendem `AppError` | CODIGO | src/shared/errors/app-error.ts |
| FDD-INT-03 | docs/FDD.md | Restrição | `error.middleware.ts` não precisa de alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-04 | docs/FDD.md | Restrição | Endpoint de replay reaproveita `requireRole('ADMIN')` | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-05 | docs/FDD.md | Restrição | Novos routers montados em `buildApiRouter` | CODIGO | src/routes/index.ts |
| FDD-INT-06 | docs/FDD.md | Restrição | Worker reutiliza `createPrismaClient()` | CODIGO | src/config/database.ts |
| FDD-INT-07 | docs/FDD.md | Restrição | Worker e módulo usam o logger Pino já exportado | CODIGO | src/shared/logger/index.ts |
| FDD-INT-08 | docs/FDD.md | Restrição | Novos modelos seguem convenção de `id` UUID do schema | CODIGO | prisma/schema.prisma |
| FDD-INT-09 | docs/FDD.md | Restrição | Listagens usam o envelope `paginated()` existente | CODIGO | src/shared/http/response.ts |
| FDD-INT-10 | docs/FDD.md | Restrição | `requestId` do middleware reaproveitado para correlação de logs | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-AC-01 | docs/FDD.md | Critério de Aceitação | Payload nunca contém `items` e nunca excede 64KB | TRANSCRICAO | [09:24] Diego |
| FDD-AC-02 | docs/FDD.md | Critério de Aceitação | Filtro de eventos aplicado na inserção, não no envio | TRANSCRICAO | [09:34] Diego |
| FDD-AC-03 | docs/FDD.md | Critério de Aceitação | URL não-HTTPS rejeitada antes de qualquer persistência | TRANSCRICAO | [09:23] Sofia |
| FDD-AC-04 | docs/FDD.md | Critério de Aceitação | Histórico de entregas limitado a 100 registros | TRANSCRICAO | [09:34] Marcos |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão Outbox transacional no MySQL | TRANSCRICAO | [09:08] Larissa |
| ADR-001-ALT-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Alternativa descartada: disparo síncrono | TRANSCRICAO | [09:04] Bruno |
| ADR-001-ALT-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Alternativa descartada: Redis Streams | TRANSCRICAO | [09:07] Diego |
| ADR-001-CODE-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | Transação atual de `changeStatus` como ponto de integração | CODIGO | src/modules/orders/order.service.ts |
| ADR-002 | docs/adrs/ADR-002-worker-dedicado-com-polling.md | Decisão | Worker dedicado, processo separado, polling de 2s | TRANSCRICAO | [09:10] Larissa |
| ADR-002-ALT-01 | docs/adrs/ADR-002-worker-dedicado-com-polling.md | Trade-off | Alternativa descartada: trigger/notificação do banco | TRANSCRICAO | [09:09] Diego |
| ADR-002-ALT-02 | docs/adrs/ADR-002-worker-dedicado-com-polling.md | Trade-off | Alternativa descartada: worker no mesmo processo da API | TRANSCRICAO | [09:11] Diego |
| ADR-002-CODE-01 | docs/adrs/ADR-002-worker-dedicado-com-polling.md | Restrição | Padrão de bootstrap de processo a ser espelhado | CODIGO | src/server.ts |
| ADR-002-CODE-02 | docs/adrs/ADR-002-worker-dedicado-com-polling.md | Restrição | Factory de `PrismaClient` a ser reaproveitada pelo worker | CODIGO | src/config/database.ts |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-exponencial.md | Decisão | Retry com backoff exponencial, 5 tentativas | TRANSCRICAO | [09:17] Larissa |
| ADR-003-ALT-01 | docs/adrs/ADR-003-retry-com-backoff-exponencial.md | Trade-off | Alternativa descartada: teto de 3 tentativas | TRANSCRICAO | [09:16] Bruno |
| ADR-003-ALT-02 | docs/adrs/ADR-003-retry-com-backoff-exponencial.md | Trade-off | Alternativa descartada: retry indefinido | TRANSCRICAO | [09:15] Diego |
| ADR-004 | docs/adrs/ADR-004-dead-letter-queue-com-replay-restrito-a-admin.md | Decisão | Dead Letter Queue em tabela separada, replay restrito a ADMIN | TRANSCRICAO | [09:36] Larissa |
| ADR-004-ALT-01 | docs/adrs/ADR-004-dead-letter-queue-com-replay-restrito-a-admin.md | Trade-off | Alternativa descartada: marcar `failed` na própria outbox | TRANSCRICAO | [09:17] Larissa |
| ADR-004-CODE-01 | docs/adrs/ADR-004-dead-letter-queue-com-replay-restrito-a-admin.md | Restrição | `requireRole` já existente, reaproveitado no replay | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-004-CODE-02 | docs/adrs/ADR-004-dead-letter-queue-com-replay-restrito-a-admin.md | Restrição | Exemplo de uso de `requireRole('ADMIN')` em rota existente | CODIGO | src/modules/users/user.routes.ts |
| ADR-005 | docs/adrs/ADR-005-assinatura-hmac-sha256-com-rotacao-de-secret.md | Decisão | HMAC-SHA256 com secret por endpoint e rotação (grace period 24h) | TRANSCRICAO | [09:22] Sofia |
| ADR-005-ALT-01 | docs/adrs/ADR-005-assinatura-hmac-sha256-com-rotacao-de-secret.md | Trade-off | Alternativa descartada: secret global | TRANSCRICAO | [09:21] Sofia |
| ADR-005-CODE-01 | docs/adrs/ADR-005-assinatura-hmac-sha256-com-rotacao-de-secret.md | Restrição | Padrão de validação Zod a ser seguido no módulo | CODIGO | src/middlewares/validate.middleware.ts |
| ADR-006 | docs/adrs/ADR-006-entrega-at-least-once-com-deduplicacao-por-event-id.md | Decisão | Entrega at-least-once com dedup por `X-Event-Id` | TRANSCRICAO | [09:26] Larissa |
| ADR-006-ALT-01 | docs/adrs/ADR-006-entrega-at-least-once-com-deduplicacao-por-event-id.md | Trade-off | Alternativa descartada: garantia exactly-once | TRANSCRICAO | [09:25] Diego |
| ADR-007 | docs/adrs/ADR-007-reuso-de-padroes-existentes-do-projeto.md | Decisão | Reuso máximo dos padrões arquiteturais existentes | TRANSCRICAO | [09:30] Larissa |
| ADR-007-ALT-01 | docs/adrs/ADR-007-reuso-de-padroes-existentes-do-projeto.md | Trade-off | Alternativa descartada: convenção própria para o módulo | TRANSCRICAO | [09:27] Bruno |
| ADR-007-CODE-01 | docs/adrs/ADR-007-reuso-de-padroes-existentes-do-projeto.md | Restrição | Composição de módulo (controller/service/repository/routes/schemas) | CODIGO | src/modules/orders/order.controller.ts |
| ADR-007-CODE-02 | docs/adrs/ADR-007-reuso-de-padroes-existentes-do-projeto.md | Restrição | Hierarquia de erros (`InsufficientStockError`, `InvalidStatusTransitionError`) | CODIGO | src/shared/errors/http-errors.ts |
| ADR-007-CODE-03 | docs/adrs/ADR-007-reuso-de-padroes-existentes-do-projeto.md | Restrição | Registro de routers do projeto | CODIGO | src/routes/index.ts |

## Cobertura

- **Total de linhas:** 150.
- **Fonte = TRANSCRICAO:** 122 linhas (~81%), acima do mínimo de 70%.
- **Fonte = CODIGO:** 28 linhas (~19%), acima do mínimo de 5.
- **Itens não rastreados individualmente:** consequências (positivas/negativas) de cada ADR e alguns critérios de aceite técnicos do FDD que apenas reformulam, em nível de verificação, um requisito já rastreado acima (ex.: os demais itens de `FDD-AC-*` além dos 4 listados) — omitidos para não duplicar a mesma origem sob um ID diferente, não por falta de rastreabilidade.
