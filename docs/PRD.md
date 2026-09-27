# PRD — Sistema de Webhooks de Notificação de Pedidos

**Documentos relacionados:** [RFC-001](RFC.md) (proposta técnica) · [ADR-001 a ADR-007](adrs/) (decisões de arquitetura) · [FDD](FDD.md) (detalhamento de implementação) · [`TRANSCRICAO.md`](../TRANSCRICAO.md) (fonte primária).

## 1. Resumo e Contexto da Feature

O Order Management System (OMS) da empresa hoje não notifica ninguém quando um pedido muda de status — clientes integrados precisam consultar `GET /orders` repetidamente para descobrir mudanças. Esta feature adiciona um sistema de **webhooks outbound**: quando o status de um pedido muda, a plataforma envia automaticamente uma notificação HTTP assinada para os endpoints cadastrados pelo cliente. A decisão técnica de como construir isso já foi fechada em reunião entre Tech Lead, PM, engenharia e segurança (`TRANSCRICAO.md`) e está detalhada na [RFC](RFC.md) e nas [ADRs](adrs/); este PRD cobre o problema, o público, o escopo e os critérios de sucesso do ponto de vista de produto.

## 2. Problema e Motivação

Três clientes B2B pediram formalmente para ser avisados em tempo real sobre mudança de status, e um deles já ameaçou migrar para um concorrente:

> [09:00] Marcos: Bom dia. Então, a gente recebeu na semana passada um pedido formal de três clientes B2B: Atlas Comercial, MaxDistribuição e Nova Cargo. Os três querem ser notificados em tempo real quando o status dos pedidos deles muda na nossa plataforma. Hoje eles ficam batendo no GET /orders de tempos em tempos pra ver se mudou alguma coisa, e isso tá deixando a integração lenta e cara pra eles. A Atlas chegou a sugerir que se a gente não entregar isso até fim do trimestre, eles podem migrar pro nosso concorrente.

O requisito de "tempo real" foi quantificado junto aos clientes:

> [09:02] Marcos: Eu perguntei especificamente isso. Pra eles, qualquer coisa abaixo de 10 segundos já é "tempo real". O importante é que não fique pendurado e eles tenham que ficar atualizando manualmente.

Motivação de negócio: reduzir o custo de integração para clientes B2B, eliminar a dependência de polling, e reter clientes que já sinalizaram risco de churn por causa dessa lacuna.

## 3. Público-Alvo e Cenários de Uso

**Público-alvo:** sistemas de clientes B2B integrados à plataforma (inicialmente Atlas Comercial, MaxDistribuição e Nova Cargo) que consomem pedidos via API. O cadastro do webhook é feito por um usuário operador autenticado normalmente (JWT do próprio sistema), não pelo cliente final diretamente:

> [09:32] Larissa: Então é endpoint autenticado normal, e o customer_id é passado no body ou no path. Não vem do JWT.

**Cenários de uso:**

1. Um operador cadastra um webhook para um customer, informando URL e os status de pedido que deseja acompanhar; recebe a secret gerada pela plataforma na resposta.
2. O status de um pedido do customer muda para um dos status inscritos; o sistema do cliente recebe uma notificação HTTP assinada em até 10 segundos no caso comum, sem precisar consultar a API.
3. O endpoint do cliente fica temporariamente indisponível; a plataforma tenta novamente automaticamente, com intervalos crescentes, sem intervenção humana.
4. O cliente quer auditar o que foi enviado a ele; consulta o histórico das últimas 100 entregas (sucesso/falha, payload, tempo de resposta).
5. O cliente suspeita que sua secret vazou; solicita rotação pela API sem perder notificações durante a transição.
6. Um endpoint fica indisponível por tempo suficiente para esgotar todas as tentativas automáticas; um administrador identifica o caso e reprocessa manualmente.

## 4. Objetivos e Métricas de Sucesso

| Objetivo | Métrica | Meta |
| --- | --- | --- |
| Notificar clientes B2B sobre mudança de status sem exigir polling | Latência entre mudança de status e recebimento da notificação, no caso comum | **Inferior a 10 segundos** (`[09:02] Marcos`, `[09:10] Larissa`) |
| Não perder notificações por indisponibilidade temporária do cliente | Tentativas automáticas antes de exigir intervenção manual | **5 tentativas, cobrindo até ~15h de indisponibilidade** (`[09:17] Diego`) |
| Reter os clientes que sinalizaram risco de churn | Entrega da feature dentro do prazo comercial combinado | **Até fim de novembro / 3 sprints** (`[09:45]-[09:47]`) |
| Garantir autenticidade das notificações para o cliente | Padrão de assinatura adotado | **HMAC-SHA256 em 100% das entregas** (`[09:20] Sofia`) |

Não há, na transcrição, uma meta formal de taxa de sucesso de entrega pós-lançamento (ex.: "99% das notificações entregues em X tentativas"); se essa métrica for necessária, deve ser definida em uma revisão futura deste PRD — não é inventada aqui por falta de evidência.

## 5. Escopo

### Incluso

- Cadastro, edição, remoção e listagem de webhooks por customer (URL, secret, filtro de eventos por status).
- Entrega automática de notificação quando o status do pedido muda para um status inscrito.
- Retry automático com backoff exponencial e Dead Letter Queue para falhas persistentes.
- Reprocessamento manual de eventos falhos, restrito a administradores.
- Assinatura HMAC-SHA256 das notificações, com secret exclusiva por endpoint e rotação sem downtime.
- Garantia de entrega at-least-once, com identificador estável para o cliente deduplicar.
- Histórico de entregas por webhook (últimas 100).

### Fora de Escopo (nesta fase)

- **Alerta por e-mail ao cliente em caso de falhas repetidas de entrega** — adiado explicitamente para uma fase futura:

  > [09:37] Marcos: Última pergunta de requisito. Tem como avisar o cliente quando o webhook dele tá com problema? Tipo se ele falhou 3 vezes seguidas, mandar email pra ele.

  > [09:37] Larissa: Não. Email tá fora de escopo dessa fase. Talvez próxima fase, depois que a gente medir o impacto.

- **Dashboard visual para o cliente acompanhar seus webhooks** — descartado nesta fase, tratado como projeto separado do time de frontend:

  > [09:39] Marcos: Dashboard visual? Tipo painel pro cliente ver os webhooks dele?

  > [09:40] Larissa: Não, agora não. Só endpoints. Painel é projeto separado do time de frontend.

- **Webhooks inbound** (cliente enviando dados para a plataforma) — descartado no início da reunião; o escopo é exclusivamente outbound (`[09:02]-[09:03] Sofia/Marcos`).
- **Rate limiting de envio ao cliente** — levantado como preocupação, mas não decidido; a equipe optou por observar antes de agir (`[09:38]-[09:39] Diego/Larissa`).
- **Arquivamento automático de eventos já entregues** — mencionado como necessidade futura, mas fora do escopo desta feature (`[09:08] Diego`).
- **Restrição de papel no CRUD de webhooks** — nesta fase qualquer usuário autenticado configura webhooks; endurecer essa permissão ficou para depois (`[09:37] Sofia`).

## 6. Requisitos Funcionais

| ID | Requisito | Fonte |
| --- | --- | --- |
| RF-01 | Cadastrar webhook (URL, status desejados, customer_id); secret é gerada pela plataforma e devolvida apenas na criação | `[09:31]-[09:32] Marcos` |
| RF-02 | Editar um webhook cadastrado | `[09:33] Bruno` |
| RF-03 | Remover um webhook cadastrado | `[09:33] Bruno` |
| RF-04 | Listar os webhooks cadastrados de um customer | `[09:33] Bruno` |
| RF-05 | Filtrar por webhook quais status de pedido geram notificação, aplicado na inserção do evento (não no envio) | `[09:33]-[09:34] Marcos/Bruno/Diego` |
| RF-06 | Consultar histórico das últimas 100 entregas de um webhook (sucesso/falha, payload, response, tempo de resposta) | `[09:34] Marcos` |
| RF-07 | Reprocessar manualmente um evento definitivamente falho, restrito a usuários com papel ADMIN, com auditoria de quem executou | `[09:18] Diego`, `[09:35]-[09:36] Larissa/Sofia` |
| RF-08 | Assinar cada notificação enviada com HMAC-SHA256 | `[09:20] Sofia` |
| RF-09 | Permitir rotação de secret pela API, mantendo a secret anterior válida por 24h em paralelo | `[09:21] Sofia` |
| RF-10 | Garantir que o cliente consiga identificar e deduplicar notificações repetidas (entrega at-least-once) | `[09:24]-[09:26] Diego/Larissa` |
| RF-11 | Tentar novamente automaticamente uma notificação que falhou, antes de considerá-la falha permanente | `[09:14]-[09:17] Larissa/Diego/Bruno` |
| RF-12 | Notificação contém dados básicos do pedido (identificador do evento, tipo, timestamp, pedido, status anterior/novo, customer, total) sem a lista de itens | `[09:43] Diego` |

Cobertura: 12 requisitos funcionais identificados na reunião (mínimo exigido: 8).

## 7. Requisitos Não Funcionais

| ID | Requisito | Fonte |
| --- | --- | --- |
| RNF-01 | Latência de notificação inferior a 10 segundos no caso comum | `[09:02] Marcos`, `[09:09]-[09:10] Diego/Marcos` |
| RNF-02 | Timeout de 10 segundos por tentativa de entrega ao cliente | `[09:42] Diego` |
| RNF-03 | Consistência transacional: nunca deve existir mudança de status sem o evento correspondente registrado (nem o inverso) | `[09:40]-[09:41] Bruno/Diego` |
| RNF-04 | URLs de webhook devem ser obrigatoriamente HTTPS | `[09:23] Sofia` |
| RNF-05 | Cada endpoint de webhook deve ter uma secret exclusiva, nunca compartilhada globalmente | `[09:21] Sofia` |
| RNF-06 | Payload de notificação limitado a 64KB; excedentes são rejeitados, não truncados | `[09:23]-[09:24] Sofia/Diego/Larissa` |
| RNF-07 | O processo responsável pelo envio das notificações deve ser independente do processo da API (resiliência a reinícios/deploys) | `[09:11] Diego` |
| RNF-08 | Toda ação de reprocessamento manual deve ser auditável (usuário, timestamp) | `[09:36] Sofia` |

## 8. Decisões e Trade-offs Principais

Decisões técnicas fechadas na reunião; detalhamento completo em cada ADR. Aqui, apenas o trade-off relevante para produto:

| Decisão | Trade-off aceito | Detalhe |
| --- | --- | --- |
| Entrega assíncrona (Outbox + worker), não síncrona | Notificação não é instantânea (latência mínima de ~2s) em troca de não arriscar travar a operação de pedidos | [ADR-001](adrs/ADR-001-outbox-no-mysql.md), [ADR-002](adrs/ADR-002-worker-dedicado-com-polling.md) |
| Retry limitado (5 tentativas) em vez de indefinido | Cliente indisponível por mais de ~15h perde a notificação automática (recuperável só via replay manual) | [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial.md), [ADR-004](adrs/ADR-004-dead-letter-queue-com-replay-restrito-a-admin.md) |
| Entrega at-least-once em vez de exactly-once | Cliente pode receber a mesma notificação duas vezes; precisa deduplicar do seu lado | [ADR-006](adrs/ADR-006-entrega-at-least-once-com-deduplicacao-por-event-id.md) |
| Secret por endpoint com rotação, em vez de secret fixa/global | Mais complexidade de gestão de segredos, em troca de conter o impacto de um vazamento (já ocorreu uma vez) | [ADR-005](adrs/ADR-005-assinatura-hmac-sha256-com-rotacao-de-secret.md) |
| Reuso total dos padrões do projeto em vez de convenção própria do módulo | Herda também as limitações atuais do projeto (sem abstrações genéricas de repository/service) | [ADR-007](adrs/ADR-007-reuso-de-padroes-existentes-do-projeto.md) |

## 9. Dependências

- `OrderService.changeStatus` (`src/modules/orders/order.service.ts`) — único ponto de disparo do evento; a feature depende dessa transação existente.
- Cadastro de `Customer` e `User` já existentes — o webhook é sempre associado a um `customerId` já cadastrado, criado por um `User` autenticado.
- Revisão de segurança dedicada antes do deploy (HMAC e geração de secret), alocada explicitamente no cronograma:

  > [09:46] Sofia: Reservem pelo menos dois dias úteis pra eu revisar o código de segurança antes do deploy. HMAC e geração de secret eu quero olhar com calma.

- Comunicação com os clientes sobre o modelo de deduplicação e o portal de desenvolvedor, sob responsabilidade do PM (`[09:26] Marcos`).
- Nenhuma dependência de infraestrutura nova (mensageria externa, cache) — reaproveita MySQL/Prisma já em produção.
- Prazo comercial já comunicado à Atlas para fim de novembro (`[09:45] Marcos`).

## 10. Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Perda de cliente (Atlas e possivelmente outros) por atraso na entrega | Média | Alto | Estimativa de 3 sprints já validada pela Tech Lead, com folga para revisão de segurança; PM confirma prazo com o cliente (`[09:46]-[09:47]`) |
| Vazamento de secret do lado do sistema do cliente (já ocorreu antes) | Média | Alto | Secret exclusiva por endpoint + rotação com grace period de 24h ([ADR-005](adrs/ADR-005-assinatura-hmac-sha256-com-rotacao-de-secret.md)) |
| Indisponibilidade prolongada do cliente causa perda de notificação | Baixa–Média | Médio | Retry cobrindo ~15h + Dead Letter Queue com replay manual ([ADR-003](adrs/ADR-003-retry-com-backoff-exponencial.md), [ADR-004](adrs/ADR-004-dead-letter-queue-com-replay-restrito-a-admin.md)) |
| Worker como processo único é ponto único de falha operacional | Baixa (volume atual) | Médio | Processo desacoplado da API, com restart independente; reavaliação se o volume crescer ([ADR-002](adrs/ADR-002-worker-dedicado-com-polling.md)) |

## 11. Critérios de Aceitação

- Um operador autenticado consegue cadastrar, editar, listar e remover webhooks de um customer.
- Uma mudança de status de pedido gera notificação para todos os endpoints inscritos naquele status em até 10 segundos, no caso comum.
- Toda notificação enviada é assinada com HMAC-SHA256 e verificável pelo cliente.
- Uma falha de entrega é retentada automaticamente antes de exigir qualquer ação manual.
- Um evento que esgota as tentativas automáticas fica disponível para reprocessamento manual por um administrador, com auditoria de quem reprocessou.
- Um cliente consegue rotacionar sua secret sem período de indisponibilidade de validação.
- Um cliente consegue consultar o histórico das últimas 100 entregas de um webhook.
- Nenhuma mudança de status de pedido ocorre sem o evento de notificação correspondente ser registrado (nem o inverso).

## 12. Estratégia de Testes e Validação

- **Testes unitários** das regras de negócio do módulo (filtro de eventos por status, cálculo do intervalo de backoff, geração e verificação de assinatura HMAC), seguindo o padrão de testes já existente no projeto (`tests/orders.test.ts`, `tests/auth.test.ts`, configurado via `vitest.config.ts`).
- **Testes de integração** garantindo que a inserção do evento na outbox e a mudança de status são atômicas (uma reverte a outra).
- **Testes ponta a ponta** do fluxo completo — cadastro de webhook, mudança de status, entrega, verificação de assinatura pelo cliente — explicitamente estimados pela Tech Lead como parte do plano de sprints:

  > [09:46] Larissa: Vou estimar. Modelagem de outbox e DLQ é uma sprint. Worker e retry é uma sprint. CRUD de configuração e deliveries é meio sprint. Integração no order.service e testes ponta a ponta é mais meio. HMAC, schemas, validações, mais um pouco. Eu chuto três sprints incluindo revisão da Sofia.

- **Revisão manual de segurança** pela engenheira de segurança antes do deploy, com foco em HMAC e geração/rotação de secret (`[09:46] Sofia`, dois dias úteis reservados).
- A transcrição não menciona plano de teste de carga/volume para o worker; não é inventado aqui — se necessário, deve ser definido em fase de execução, fora do escopo deste PRD.

## Consistência

Nenhuma informação deste PRD contradiz a RFC, as ADRs ou o código-fonte examinado. Os requisitos funcionais e não funcionais aqui listados são consolidados a partir das mesmas fontes já rastreadas em [RFC](RFC.md) e [ADRs](adrs/); o detalhamento técnico de cada um está no [FDD](FDD.md).
