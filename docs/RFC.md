# RFC-001: Sistema de Webhooks de Notificação de Pedidos

**Detalhamento de implementação:** [FDD](FDD.md). **Decisões fechadas:** [ADR-001](adrs/ADR-001-outbox-no-mysql.md) a [ADR-007](adrs/ADR-007-reuso-de-padroes-existentes-do-projeto.md). Esta RFC opera em nível de arquitetura e não repete o detalhe de fluxos, contratos HTTP ou modelagem de dados, que estão no FDD.

## Metadados

| Campo | Valor |
| --- | --- |
| **Status** | Proposto — submetido à equipe para revisão |
| **Autor** | Larissa (Tech Lead) |
| **Data** | Reunião de kickoff técnico, quinta-feira, 09:00 (a transcrição não registra data de calendário) |
| **Revisores** | Marcos (PM), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Eng. Segurança) |

## Resumo Executivo (TL;DR)

A plataforma vai notificar clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) sobre mudanças de status de pedido via **webhooks outbound**, substituindo o polling atual em `GET /orders`. A entrega usa **Outbox transacional no MySQL** ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)) lida por um **worker dedicado em polling de 2s** ([ADR-002](adrs/ADR-002-worker-dedicado-com-polling.md)), com **retry exponencial de 5 tentativas** e **Dead Letter Queue com replay restrito a ADMIN** ([ADR-003](adrs/ADR-003-retry-com-backoff-exponencial.md), [ADR-004](adrs/ADR-004-dead-letter-queue-com-replay-restrito-a-admin.md)). Autenticidade via **HMAC-SHA256 com secret por endpoint e rotação (grace period de 24h)** ([ADR-005](adrs/ADR-005-assinatura-hmac-sha256-com-rotacao-de-secret.md)); entrega **at-least-once com deduplicação por `X-Event-Id`** ([ADR-006](adrs/ADR-006-entrega-at-least-once-com-deduplicacao-por-event-id.md)). O módulo novo reaproveita integralmente a estrutura, erros, logger e middlewares já existentes ([ADR-007](adrs/ADR-007-reuso-de-padroes-existentes-do-projeto.md)). Estimativa: **três sprints**, incluindo revisão de segurança.

## Contexto e Problema

Três clientes B2B pediram para ser avisados em tempo real sobre mudança de status, alegando que o polling atual é lento e caro:

> [09:00] Marcos: Bom dia. Então, a gente recebeu na semana passada um pedido formal de três clientes B2B: Atlas Comercial, MaxDistribuição e Nova Cargo. Os três querem ser notificados em tempo real quando o status dos pedidos deles muda na nossa plataforma. Hoje eles ficam batendo no GET /orders de tempos em tempos pra ver se mudou alguma coisa, e isso tá deixando a integração lenta e cara pra eles. A Atlas chegou a sugerir que se a gente não entregar isso até fim do trimestre, eles podem migrar pro nosso concorrente.

"Tempo real" foi quantificado como abaixo de 10 segundos (`[09:02] Marcos`), e o escopo foi delimitado logo no início como **exclusivamente outbound** — a plataforma envia, o cliente não envia para a plataforma (`[09:02]-[09:03] Sofia/Marcos`).

A aplicação hoje não possui nenhum mecanismo de notificação externa: `OrderService.changeStatus` (`src/modules/orders/order.service.ts`) apenas atualiza `orders`, insere em `order_status_history` e ajusta estoque, tudo em uma única transação Prisma. Essa transação é o ponto de integração central da feature (ver [ADR-001](adrs/ADR-001-outbox-no-mysql.md)).

## Proposta Técnica

A solução combina sete decisões já fechadas (ver [Decisões Relacionadas](#decisões-relacionadas)): o evento é capturado dentro da transação de mudança de status via padrão Outbox, entregue por um worker dedicado com retry e Dead Letter Queue, assinado com HMAC-SHA256, e identificado por um `event_id` estável para deduplicação no cliente. O módulo novo (`src/modules/webhooks/`) segue a mesma composição (controller, service, repository, routes, schemas) já usada em `src/modules/orders/`, sem introduzir convenções novas.

O ponto de integração mais sensível com o código existente é `OrderService.changeStatus`: a inserção do evento na outbox deve ocorrer dentro da mesma transação que já atualiza `orders` e `order_status_history`, para que status alterado sempre implique evento gravado (e vice-versa):

> [09:40] Bruno: Sobre integração com o código atual: a alteração crítica é dentro do service de orders, no método changeStatus. Hoje a transação faz update na order, insere no history e atualiza estoque. A gente vai inserir na webhook_outbox dentro da mesma transação. Se a outbox falhar de inserir, rollback. Não pode ter caso de status mudar e evento não sair.

> [09:41] Diego: Essencial. Se ficar fora da transação, perde a garantia toda.

Visão geral do fluxo:

```
changeStatus (tx) ──insere──> webhook_outbox ──lê a cada 2s──> worker ──HTTP + HMAC──> cliente
                                    │                             │ falhou
                              snapshot do payload                 ├─ retry com backoff (até 5x)
                                                                  └─ esgotou ─> webhook_dead_letter
                                                                                      │
                            webhook_outbox <── replay manual (ADMIN) ─────────────────┘
```

O detalhamento de fluxos passo a passo (criação do evento, processamento pelo worker, retry, DLQ), contratos HTTP com payloads de exemplo, modelagem de dados e matriz de erros está no [FDD](FDD.md) — não é repetido aqui.

## Decisões Relacionadas

Todas as decisões abaixo estão fechadas; esta RFC não as reabre, apenas consolida como elas compõem a solução.

| ADR | Decisão | Contribuição para a solução |
| --- | --- | --- |
| [ADR-001](adrs/ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL | Captura o evento dentro da transação de mudança de status, sem infraestrutura nova. |
| [ADR-002](adrs/ADR-002-worker-dedicado-com-polling.md) | Worker dedicado, polling de 2s | Processo separado da API lê a outbox e entrega os eventos por HTTP. |
| [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial.md) | Retry com backoff exponencial | 5 tentativas (1m/5m/30m/2h/12h) antes de considerar falha permanente. |
| [ADR-004](adrs/ADR-004-dead-letter-queue-com-replay-restrito-a-admin.md) | DLQ com replay restrito a ADMIN | Eventos esgotados vão para tabela separada; reprocessamento manual, auditado. |
| [ADR-005](adrs/ADR-005-assinatura-hmac-sha256-com-rotacao-de-secret.md) | HMAC-SHA256 com rotação de secret | Garante autenticidade; secret por endpoint isola vazamentos; rotação com grace period de 24h. |
| [ADR-006](adrs/ADR-006-entrega-at-least-once-com-deduplicacao-por-event-id.md) | At-least-once com `X-Event-Id` | Define a semântica de entrega e o mecanismo de deduplicação no cliente. |
| [ADR-007](adrs/ADR-007-reuso-de-padroes-existentes-do-projeto.md) | Reuso de padrões existentes | Módulo novo herda estrutura, erros, logger e middlewares já validados no projeto. |

## Alternativas Consideradas

**1. Disparo síncrono dentro de `OrderService.changeStatus`.** Descartada: acoplaria a disponibilidade de um sistema externo à transação crítica de mudança de status, sem resposta viável para o caso de o cliente estar fora do ar (trade-off: simplicidade vs. risco de indisponibilidade da própria plataforma).

> [09:04] Bruno: Síncrono não rola. A transação de mudança de status hoje já é pesada — atualiza orders, insere na order_status_history, decrementa stock_quantity dos produtos do pedido. Se a gente acrescentar um HTTP call no meio disso, qualquer cliente lento vai travar mudança de status pra outros pedidos.

**2. Fila externa dedicada (ex.: Redis Streams) em vez de Outbox no MySQL.** Descartada por custo operacional de subir e manter infraestrutura nova para um time pequeno (trade-off: escala futura vs. overhead operacional imediato).

> [09:07] Larissa: Faz sentido. A alternativa seria botar Redis Streams ou alguma coisa parecida, mas a gente acabaria precisando subir mais infra.

> [09:07] Diego: Exato, e a gente é um time pequeno. Subir Redis Cluster pra isso é overengineering. Outbox no MySQL existente resolve.

**3. Garantia de entrega exactly-once.** Descartada por exigir coordenação transacional entre plataforma e cliente para eliminar uma fração pequena de casos que o padrão de mercado já resolve com at-least-once + deduplicação (trade-off: simplicidade vs. responsabilidade adicional transferida ao cliente).

> [09:25] Diego: Joga, mas é o padrão de mercado. Stripe faz assim, GitHub faz assim. Garantir exactly-once exigiria coordenação dos dois lados e fica muito mais complexo. At-least-once com event_id resolve 99% dos casos.

**4. Secret HMAC única e global para toda a plataforma.** Descartada por concentrar risco: vazamento de uma única secret comprometeria todos os clientes (trade-off: menor complexidade de armazenamento vs. blast radius de um vazamento).

> [09:21] Sofia: Outra coisa importante: cada endpoint de webhook do cliente tem que ter uma secret única. Não é uma secret global da nossa plataforma. Senão se vaza uma, vaza tudo.

**5. Trigger de banco para entrega reativa, em vez de polling.** Descartada porque o MySQL não tem listener nativo equivalente ao `NOTIFY/LISTEN` do Postgres; um trigger só executa SQL e não avisa um processo externo (trade-off: reatividade vs. soluções improvisadas para notificar o worker). Ver [ADR-002](adrs/ADR-002-worker-dedicado-com-polling.md) (`[09:09] Diego`).

**6. Teto de 3 tentativas de retry.** Descartada por esgotar em cerca de 30 minutos, insuficiente para indisponibilidades reais de horas (trade-off: falha mais rápida vs. perda de eventos legítimos). Ver [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial.md) (`[09:16] Bruno/Diego`).

**7. Retry indefinido.** Descartada porque deixaria o evento pendurado para sempre se o cliente sumisse (trade-off: cobertura total vs. ausência de ponto de falha definitiva). Ver [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial.md) (`[09:15] Diego`).

## Questões em Aberto

**1. Rate limiting de envio para o cliente.** Levantado como preocupação, mas explicitamente não decidido — a equipe optou por observar antes de implementar.

> [09:38] Diego: Outra coisa que ficou na minha cabeça: rate limiting de envio pra cliente. Se o cliente tem 50 pedidos mudando de status em um minuto, a gente bombardeia ele com 50 chamadas?

> [09:39] Larissa: Tá. Fica como "observar e decidir depois".

**2. Garantia de ordenação ao escalar para múltiplos workers.** A ordenação por `order_id` só vale com um único worker (ADR-002); o mecanismo para múltiplos workers em paralelo não foi decidido.

> [09:13] Diego: Aí dá pra particionar por order_id, ou usar lock pessimista. Mas isso é problema do futuro, não agora.

**3. Política de arquivamento de eventos já entregues na outbox.** Mencionada apenas de forma aproximada, sem prazo ou mecanismo fechados, e fora do escopo desta feature.

> [09:08] Diego: A tabela tem índice no campo de status (pendente, processando, falhou, entregue) e em created_at. Worker lê só os pendentes em batch pequeno, processa, marca como entregue. Linhas entregues a gente arquiva depois de 30 dias ou assim, fora do escopo dessa feature.

**4. Endurecimento da autorização do CRUD de webhooks.** Nesta fase, qualquer papel autenticado pode configurar webhooks (só o replay exige ADMIN); a segurança sinalizou que isso pode ser restringido depois, sem definir quando nem como.

> [09:36] Marcos: O resto do CRUD de configuração de webhook pode ser qualquer role autenticada?

> [09:37] Sofia: Por enquanto sim. Mais pra frente a gente pode endurecer.

## Impacto e Riscos

- **Novo processo em produção:** o worker exige deploy e monitoramento independentes da API (ADR-002).
- **Novo esquema de dados:** novas tabelas no MySQL de produção, exigindo migração Prisma (todas seguem a convenção de `id` UUID já usada no schema).
- **Risco de segurança conhecido:** o time já teve um incidente de vazamento de secret de cliente; a rotação com grace period de 24h reduz mas não elimina esse risco, pois depende também da postura do sistema do cliente (ADR-005). Por isso, a revisão de segurança é bloqueio explícito do cronograma:

  > [09:46] Sofia: Reservem pelo menos dois dias úteis pra eu revisar o código de segurança antes do deploy. HMAC e geração de secret eu quero olhar com calma.

- **Risco de negócio:** atraso na entrega tem risco explícito de perda de cliente (`[09:00] Marcos` — ameaça de migração da Atlas para concorrente até fim do trimestre).
- **Sem novas dependências de infraestrutura de mensageria**, mantendo o custo operacional baixo, mas acoplando a entrega ao mesmo banco transacional da aplicação principal.
- **Auditoria:** o replay administrativo deve registrar o usuário que executou a ação (ADR-004).

## Plano de Implementação

> [09:46] Larissa: Vou estimar. Modelagem de outbox e DLQ é uma sprint. Worker e retry é uma sprint. CRUD de configuração e deliveries é meio sprint. Integração no order.service e testes ponta a ponta é mais meio. HMAC, schemas, validações, mais um pouco. Eu chuto três sprints incluindo revisão da Sofia.

> [09:47] Larissa: Combinado. Três sprints com a revisão da Sofia incluída no fim.

Prazo alvo: fim de novembro, a pedido da Atlas (`[09:45] Marcos`). A transcrição não detalha cronograma dia a dia; a decomposição por sprint acima é a única granularidade sustentada pela fonte.

## Consistência

Nenhuma divergência foi identificada entre a transcrição, as ADRs e o código-fonte examinado. Os pontos sem evidência suficiente para afirmação definitiva estão listados em [Questões em Aberto](#questões-em-aberto) e na data de calendário em [Metadados](#metadados) (a transcrição só registra "quinta-feira, 09:00", sem data absoluta).

## Referências

- Transcrição: [`TRANSCRICAO.md`](../TRANSCRICAO.md) — todos os timestamps citados nesta RFC referem-se a este arquivo.
- ADRs: [ADR-001](adrs/ADR-001-outbox-no-mysql.md) a [ADR-007](adrs/ADR-007-reuso-de-padroes-existentes-do-projeto.md).
- Detalhamento de implementação: [`docs/FDD.md`](FDD.md).
- Visão de produto e escopo de negócio: [`docs/PRD.md`](PRD.md).
- Código-fonte referenciado: [`order.service.ts`](../src/modules/orders/order.service.ts), [`app-error.ts`](../src/shared/errors/app-error.ts), [`error.middleware.ts`](../src/middlewares/error.middleware.ts), [`prisma/schema.prisma`](../prisma/schema.prisma).
