# ADR-001: Padrão Outbox no MySQL para publicação de eventos de pedido

## Status

Aceito.

## Contexto

A feature de Webhooks de Notificação de Pedidos precisa disparar uma chamada HTTP para sistemas externos (Atlas Comercial, MaxDistribuição, Nova Cargo) sempre que o status de um pedido muda. A mudança de status já ocorre dentro de uma transação de banco relativamente pesada, hoje implementada em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), que roda dentro de `this.prisma.$transaction(async (tx) => {...})` e executa, na mesma transação: leitura do pedido, validação de transição (`canTransition`, `src/modules/orders/order.status.ts`), débito/reposição de estoque (`debitStock`/`replenishStock`, que fazem `tx.product.update` com `decrement`/`increment` em `stockQuantity`), `tx.order.update` do novo status e `tx.orderStatusHistory.create` do registro de auditoria.

A equipe discutiu se o disparo do webhook deveria ser síncrono, dentro dessa mesma transação/service, ou assíncrono via algum mecanismo de fila.

> [09:03] Larissa: Acho que a primeira pergunta é: a gente dispara isso sincronamente no service de orders quando o status muda, ou faz algum tipo de fila/outbox?

> [09:04] Bruno: Síncrono não rola. A transação de mudança de status hoje já é pesada — atualiza orders, insere na order_status_history, decrementa stock_quantity dos produtos do pedido. Se a gente acrescentar um HTTP call no meio disso, qualquer cliente lento vai travar mudança de status pra outros pedidos.

> [09:04] Bruno: Sem falar que se o cliente tiver fora do ar, o que a gente faz, dá rollback na mudança de status? Não dá.

Diego propôs especificamente o padrão Outbox transacional, evitando tanto o disparo síncrono quanto a introdução de uma fila externa:

> [09:06] Diego: Síncrono está fora de questão. Aliás, eu nem chamaria de "fila" — o que a gente quer aqui é um padrão outbox.

> [09:06] Diego: Outbox é o seguinte: quando o status do pedido muda, dentro da mesma transação SQL que atualiza orders e order_status_history, a gente também insere uma linha numa tabela tipo webhook_outbox com o evento. Um worker separado fica lendo essa tabela e disparando as chamadas HTTP. Garante que se a transação principal commitou, o evento foi registrado, e se ela deu rollback, o evento some junto. Não tem inconsistência possível.

Também ficou definido que o evento gravado na outbox deve ser o payload já renderizado (snapshot), não apenas o `order_id`, para não refletir um estado do pedido posterior ao momento da mudança de status:

> [09:51] Bruno: Eu também tenho uma dúvida última: o evento da outbox guarda o payload renderizado já, ou guarda só order_id e renderiza na hora do envio?

> [09:52] Larissa: Boa pergunta. Eu prefiro renderizado já, na hora da inserção. Se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou. Senão tem caso esquisito.

> [09:52] Diego: Concordo, snapshot na inserção.

E que o identificador da linha da outbox segue a convenção de UUID já usada em todas as tabelas do projeto (`String @id @default(uuid()) @db.Char(36)` em `User`, `Customer`, `Product`, `Order`, `OrderItem`, `OrderStatusHistory`, em `prisma/schema.prisma`):

> [09:51] Diego: Quando a gente for modelar a outbox, prefere id auto incremental ou UUID?

> [09:51] Larissa: UUID, segue o padrão do resto do projeto. Tudo é uuid.

## Decisão

Adotar o padrão **Transactional Outbox** sobre o MySQL já utilizado pelo projeto (via Prisma), sem introduzir infraestrutura de mensageria nova.

Dentro da mesma transação Prisma que hoje é executada em `OrderService.changeStatus` — junto com `tx.order.update` e `tx.orderStatusHistory.create` — será inserida uma linha em uma nova tabela `webhook_outbox`, contendo o `event_id` (UUID), o `event_type` (ex.: `order.status_changed`), o payload já renderizado (snapshot) e o status de processamento (pendente/processando/falhou/entregue). Se a transação principal reverter, o evento nunca existiu; se ela commitar, o evento está garantidamente registrado. Um worker separado (ver ADR-002) é o único responsável por ler essa tabela e efetivamente entregar o evento por HTTP — a transação de mudança de status nunca faz chamada de rede.

O `id` da tabela `webhook_outbox` segue a convenção UUID (`@default(uuid())`) já usada em todas as demais tabelas do schema.

## Alternativas Consideradas

**1. Disparo síncrono dentro do `OrderService.changeStatus`.** Descartada porque acopla a latência e a disponibilidade de um sistema externo à transação crítica de mudança de status, arriscando travar a mudança de status de outros pedidos caso o cliente externo esteja lento, e sem uma resposta razoável para o caso de falha (não é possível/desejável dar rollback da mudança de status por causa de um cliente HTTP fora do ar).

**2. Fila externa dedicada (ex.: Redis Streams).** Considerada e descartada por custo operacional: exigiria subir e manter infraestrutura nova para um time pequeno, sem necessidade comprovada de throughput que justifique o overhead.

> [09:07] Larissa: Faz sentido. A alternativa seria botar Redis Streams ou alguma coisa parecida, mas a gente acabaria precisando subir mais infra.

> [09:07] Diego: Exato, e a gente é um time pequeno. Subir Redis Cluster pra isso é overengineering. Outbox no MySQL existente resolve.

## Consequências

**Positivas:**

- Garantia forte de consistência entre a mudança de status e o registro do evento: não há caso possível de status mudado sem evento gravado, nem evento gravado sem mudança de status efetivada (ambos vivem/morrem na mesma transação SQL).
- Nenhuma infraestrutura nova é introduzida; o time reaproveita o MySQL e o Prisma já existentes, reduzindo custo operacional e superfície de operação.
- O snapshot do payload na inserção evita inconsistências caso o pedido seja alterado novamente antes do envio do evento.

**Negativas (trade-off explícito):**

- A entrega deixa de ser instantânea: existe uma etapa intermediária de leitura por um worker (ver ADR-002), introduzindo latência mínima adicional em vez de notificação imediata.
- A tabela `webhook_outbox` cresce continuamente e precisa de uma estratégia de manutenção (arquivamento de linhas já entregues); a reunião definiu isso como fora do escopo desta feature (arquivamento após ~30 dias mencionado como ideia futura, não uma decisão fechada — ver `docs/RFC.md`, seção de questões em aberto).
- Acopla a modelagem de eventos de webhook ao mesmo banco transacional da aplicação principal, em vez de um sistema de mensageria dedicado — aceitável para o volume atual, mas é um limite conhecido de escala.
