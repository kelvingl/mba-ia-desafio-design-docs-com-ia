# ADR-004: Dead Letter Queue em tabela separada, com replay manual restrito a ADMIN

## Status

Aceito.

## Contexto

Quando um evento de webhook esgota as 5 tentativas de entrega definidas na ADR-003, o time precisava decidir onde esse evento "morto" seria armazenado e como (e por quem) ele poderia ser reprocessado.

> [09:17] Larissa: Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h. Próximo: DLQ. Faz numa tabela separada ou marca como "failed" na própria outbox?

> [09:18] Diego: Eu fazia uma tabela webhook_dead_letter separada, com a payload, motivo da falha e timestamp. Mais limpa a leitura da outbox principal, e fica como evidence pra debug e reprocessamento.

> [09:18] Bruno: Faz sentido. E quem reprocessa? Tem endpoint?

> [09:18] Diego: Manual via endpoint admin. Tipo um POST /admin/webhooks/dead-letter/:id/replay. Recoloca na outbox como pendente.

A exigência de a operação de replay ser restrita a um papel administrativo, com trilha de auditoria, veio da revisão de segurança:

> [09:35] Larissa: Quem é admin? Tem que ser role ADMIN do JWT?

> [09:36] Sofia: Tem que ser ADMIN sim. Mexer em fila de entrega de notificação não é coisa de operador. E o endpoint de admin tem que logar quem fez o replay, pra auditoria.

> [09:36] Larissa: Decidido, role ADMIN obrigatório no replay e a gente reaproveita o requireRole que já existe.

O projeto já possui exatamente esse mecanismo de controle de acesso por papel: `requireRole(...roles: AuthUser['role'][])`, em `src/middlewares/auth.middleware.ts`, que retorna 401 (`UnauthorizedError`) se não houver usuário autenticado e 403 (`ForbiddenError`) se o papel do usuário não estiver na lista permitida. Ele já é usado hoje para restringir um endpoint a `ADMIN`, por exemplo em `src/modules/users/user.routes.ts`:

```ts
router.get(
  '/:id',
  authenticate,
  requireRole('ADMIN'),
  validate({ params: idParamSchema }),
  controller.getById,
);
```

O papel `ADMIN` já existe como valor do enum de papel de usuário consumido por `AuthUser['role']` (`'ADMIN' | 'OPERATOR'`), refletindo o `UserRole` do Prisma.

## Decisão

Persistir eventos de webhook que esgotaram as tentativas de retry (ADR-003) em uma **tabela separada `webhook_dead_letter`** (não em um status adicional dentro da própria `webhook_outbox`), contendo ao menos o payload do evento, o motivo/detalhe da última falha e o timestamp.

O reprocessamento é **exclusivamente manual**, via endpoint administrativo `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na `webhook_outbox` com status pendente para nova tentativa de entrega pelo worker (ADR-002).

Esse endpoint é protegido por `requireRole('ADMIN')`, reaproveitando o middleware já existente em `src/middlewares/auth.middleware.ts`, no mesmo padrão de uso já presente em `src/modules/users/user.routes.ts`. A operação de replay deve registrar em log (via o logger Pino existente, `src/shared/logger/index.ts`) qual usuário administrador executou o replay, para fins de auditoria.

## Alternativas Consideradas

**1. Marcar o evento como `failed` na própria tabela `webhook_outbox`, sem tabela separada.** Considerada e descartada porque poluiria a leitura da outbox principal (usada pelo worker para buscar pendentes) com eventos mortos, tornando o índice de status menos eficiente e misturando o propósito operacional (fila de trabalho) com o propósito de auditoria/depuração de falhas permanentes.

**2. Reprocessamento automático após esgotar as tentativas (ex.: nova rodada de retries automática).** Não chegou a ser proposta como alternativa concreta na reunião; foi implicitamente descartada em favor de reprocessamento manual, já que o objetivo da DLQ é justamente isolar casos que já passaram por 5 tentativas com backoff de até 12 horas — uma repetição automática adicional teria retorno marginal e mascararia problemas que merecem investigação humana.

## Consequências

**Positivas:**

- Mantém a `webhook_outbox` enxuta e com leitura eficiente para o worker, já que eventos definitivamente falhos saem do fluxo de polling ativo.
- A tabela `webhook_dead_letter` funciona como evidência de auditoria e debug (payload, motivo, timestamp), essencial para investigar por que um cliente específico falhou persistentemente.
- Restringir o replay a `ADMIN` evita que um operador comum reprocesse eventos sensíveis sem critério, reduzindo risco de reenvio indevido de notificações a clientes externos; reaproveita infraestrutura de autorização já testada (`requireRole`) em vez de criar um novo mecanismo de permissão.

**Negativas (trade-off explícito):**

- Um evento que esgotou os retries **não é reentregue automaticamente**: exige intervenção manual de um administrador para ser recolocado na fila, o que pode significar atraso adicional (além das ~15 horas de retry) até alguém perceber e agir sobre a entrada na DLQ.
- Não há, nesta decisão, um mecanismo automático de alerta quando um evento cai na DLQ (o aviso ao cliente final sobre falhas persistentes, ex. por e-mail, foi explicitamente adiado para uma fase futura — ver `docs/RFC.md`).
