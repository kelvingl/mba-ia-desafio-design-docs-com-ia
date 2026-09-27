# ADR-002: Worker dedicado em processo separado, com polling de 2 segundos

## Status

Aceito.

## Contexto

Com o padrão Outbox definido (ADR-001), era preciso decidir como os eventos pendentes na tabela `webhook_outbox` seriam lidos e entregues.

O time avaliou usar algum mecanismo mais reativo do banco (trigger/notificação), mas o MySQL não oferece um equivalente ao `LISTEN`/`NOTIFY` do PostgreSQL:

> [09:08] Larissa: Tá decidido então: outbox em MySQL. Próximo ponto: como o worker lê isso?

> [09:09] Diego: Polling em loop. A cada 2 segundos, busca os eventos pendentes mais antigos, processa, marca.

> [09:09] Bruno: Não dá pra usar trigger do banco pra ser mais reativo?

> [09:09] Diego: MySQL não tem listener nativo tipo o NOTIFY/LISTEN do Postgres. Trigger no banco a gente até tem, mas ela não notifica processo externo, ela só executa SQL. Pra avisar o worker, a gente teria que improvisar algo tipo escrever em arquivo ou bater num endpoint, fica esquisito. Polling de 2 segundos atende o requisito de "abaixo de 10 segundos" tranquilo.

O requisito de negócio de latência veio do PM, com base no pedido dos clientes B2B (Atlas, MaxDistribuição, Nova Cargo):

> [09:02] Marcos: Eu perguntei especificamente isso. Pra eles, qualquer coisa abaixo de 10 segundos já é "tempo real". O importante é que não fique pendurado e eles tenham que ficar atualizando manualmente.

> [09:10] Marcos: 2 segundos serve, perfeito.

> [09:10] Larissa: Vamos registrar isso como uma decisão. Worker em polling, 2s. A latência mínima vai ser 2 segundos no pior caso. Aceitamos.

Também ficou decidido que esse worker não pode rodar dentro do mesmo processo da API HTTP, para não ser derrubado junto em cada reinício/deploy da API:

> [09:11] Diego: Uma coisa importante: o worker tem que rodar como processo separado, não dentro da mesma instância da API. Senão se a API reinicia, perde o worker.

> [09:11] Larissa: Tem espaço pra ser uma entry-point nova no projeto. Tipo o que a gente já tem em src/server.ts, criar um src/worker.ts e um script "npm run worker".

> [09:11] Bruno: Pode ser, mas vai precisar conectar no mesmo banco e usar o mesmo Prisma client.

> [09:11] Diego: Sim, mesmo banco, mesma stack. Só não pode ser o mesmo processo.

> [09:30] Bruno: Separado. PrismaClient é por processo. Mesmo banco, mesma DATABASE_URL, mas instância nova porque é outro processo Node.

Hoje `src/server.ts` é o único ponto de entrada do processo Node: ele monta a aplicação Express via `buildApp` (`src/app.ts`), abre o servidor HTTP com `app.listen(env.PORT, ...)` e usa a instância singleton `prisma` exportada por `src/config/database.ts` (criada via `createPrismaClient()`). Não existe hoje nenhum segundo ponto de entrada do processo.

Como consequência direta de existir um único worker fazendo polling sequencial (em vez de múltiplos workers concorrentes), a ordenação de entrega por pedido também foi discutida:

> [09:12] Larissa: Anotado. Agora, ordering. Se o pedido X muda PAID, depois PROCESSING, depois SHIPPED em sequência rápida, o cliente recebe na ordem certa?

> [09:12] Diego: Depende. Se a gente tem um único worker rodando, ele processa em ordem de created_at do outbox. Aí o cliente recebe em ordem. Se a gente escala pra múltiplos workers em paralelo no futuro, perde a garantia. Por enquanto, single-worker e ordering implícita por order_id.

> [09:13] Larissa: Documentamos como limitação conhecida. Não é garantia de ordering global, só por order_id e enquanto for single-worker.

> [09:14] Marcos: Os clientes nunca pediram garantia de ordering global, eles só querem saber se cada pedido deles mudou.

## Decisão

Implementar a entrega de webhooks em um **worker dedicado, executado como processo Node separado da API HTTP**, com **polling em loop a cada 2 segundos** sobre a tabela `webhook_outbox` (eventos pendentes ordenados pelo campo de criação).

O worker terá um novo ponto de entrada `src/worker.ts`, espelhando a estrutura de bootstrap de `src/server.ts` (mesmo padrão de log de início via `logger`, mesmo tratamento de `SIGINT`/`SIGTERM` para desligamento gracioso), porém sem `buildApp`/`app.listen`, já que não expõe HTTP. Um script `npm run worker` é adicionado ao `package.json` para subir esse processo de forma independente do `npm run dev`/`start` da API.

O worker abre sua **própria instância de `PrismaClient`**, reutilizando a mesma factory `createPrismaClient()` de `src/config/database.ts` e a mesma `DATABASE_URL` — mas não compartilha a instância singleton `prisma` do processo da API, pois cada processo Node deve ter seu próprio client/pool de conexões.

Enquanto o sistema operar com um único worker, a ordenação de entrega por `order_id` é preservada (processamento em ordem de inserção na outbox), mas não há garantia de ordenação global entre pedidos diferentes. Essa é uma limitação conhecida e aceita, não um requisito quebrado — nenhum cliente solicitou ordenação global.

## Alternativas Consideradas

**1. Notificação reativa via trigger/listener do banco.** Descartada porque o MySQL não possui um mecanismo equivalente ao `LISTEN`/`NOTIFY` do PostgreSQL; um trigger de banco só executa SQL e não consegue notificar um processo externo, exigindo soluções improvisadas (escrever em arquivo, chamar um endpoint a partir do trigger) consideradas inadequadas pelo time ("fica esquisito").

**2. Worker rodando dentro do mesmo processo da API (ex.: `setInterval` iniciado em `server.ts`).** Descartada porque acoplaria o ciclo de vida do worker ao da API: um restart/deploy da API interromperia o processamento da outbox, e picos de carga HTTP competiriam por recursos com o polling de entrega.

## Consequências

**Positivas:**

- Ciclo de vida do worker independente da API: deploys, restarts ou crashes da API não interrompem o processamento de eventos pendentes, e vice-versa.
- Latência de entrega previsível e simples de operar (não depende de infraestrutura de notificação adicional), atendendo confortavelmente o requisito de "abaixo de 10 segundos" definido pelo PM a partir do pedido dos clientes B2B.
- Reaproveita integralmente a stack existente (Prisma, MySQL, Pino) sem novas dependências de infraestrutura.

**Negativas (trade-off explícito):**

- Latência mínima de até 2 segundos é inerente ao modelo (não é notificação instantânea); no pior caso, um evento inserido logo após um ciclo de polling espera quase 2 segundos antes de ser lido.
- Não há garantia de ordenação de entrega entre pedidos diferentes; a ordenação só é preservada por `order_id` e apenas enquanto o sistema operar com um único worker. Escalar para múltiplos workers em paralelo no futuro exigiria mecanismo adicional (particionamento por `order_id` ou lock pessimista), explicitamente adiado pela reunião.
- Introduz um novo processo a ser operado, monitorado e implantado (deploy, healthcheck, restart policy), aumentando a superfície operacional em relação a uma solução de processo único.
