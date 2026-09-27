# ADR-006: Entrega at-least-once com deduplicação por `X-Event-Id` no cliente

## Status

Aceito.

## Contexto

Dado o modelo de outbox + worker com retry (ADR-001, ADR-002, ADR-003), é possível que uma entrega seja considerada falha pelo worker (ex.: timeout na resposta) mesmo que o cliente já tenha recebido e processado a requisição — ou que, por alguma falha entre o envio e a marcação de "entregue", o mesmo evento seja reenviado. O time discutiu explicitamente qual garantia de entrega assumir.

> [09:24] Diego: Voltando à entrega: a gente vai garantir at-least-once. Pode acontecer de o cliente receber o mesmo evento duas vezes. Ele tem que estar preparado.

> [09:25] Bruno: E como ele diferencia?

> [09:25] Diego: A gente manda um event_id no header, X-Event-Id, com um UUID gerado quando o evento entra na outbox. É único por evento. Se o cliente recebeu duas vezes, ele dedupica pelo event_id do lado dele.

A engenheira de segurança apontou o trade-off de responsabilidade que essa escolha impõe ao cliente, e Diego justificou com o padrão de mercado:

> [09:25] Sofia: Isso joga responsabilidade pro cliente.

> [09:25] Diego: Joga, mas é o padrão de mercado. Stripe faz assim, GitHub faz assim. Garantir exactly-once exigiria coordenação dos dois lados e fica muito mais complexo. At-least-once com event_id resolve 99% dos casos.

> [09:26] Marcos: Eu posso documentar isso bem destacado no portal de desenvolvedor pros clientes, sem problema.

> [09:26] Larissa: Beleza. At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão.

## Decisão

Garantir semântica de entrega **at-least-once** (pelo menos uma vez), aceitando que o mesmo evento possa, em cenários de falha/retry, ser entregue mais de uma vez ao cliente. Cada evento carrega um **`event_id` (UUID)**, gerado no momento em que o evento é inserido na `webhook_outbox` (ADR-001), enviado em toda tentativa de entrega no header **`X-Event-Id`**. O `event_id` é estável entre tentativas de retry do mesmo evento (não é regenerado a cada tentativa), permitindo que o cliente detecte e descarte duplicatas do lado dele.

A responsabilidade de deduplicação fica explicitamente do lado do cliente externo; essa responsabilidade será documentada de forma destacada no portal de desenvolvedor voltado aos clientes B2B.

## Alternativas Consideradas

**1. Garantia exactly-once.** Considerada e descartada por exigir coordenação transacional entre os dois lados (plataforma e sistema do cliente), o que aumentaria significativamente a complexidade de implementação para eliminar uma fração pequena de casos — a equipe optou por seguir o padrão adotado por provedores de referência do mercado (citados na reunião: Stripe, GitHub), que resolvem o problema de forma equivalente.

## Consequências

**Positivas:**

- Simplicidade de implementação no lado da plataforma: não é necessário nenhum mecanismo de confirmação distribuída ou transação de duas fases com o cliente para evitar duplicatas.
- Alinhado com o padrão de mercado adotado por provedores de webhook de referência (Stripe, GitHub), o que facilita a integração para clientes já habituados a esse modelo.
- Combinado com o retry da ADR-003, favorece não perder eventos em detrimento de, eventualmente, entregar um evento repetido — trade-off adequado para notificação de mudança de status, onde perder uma atualização é pior que recebê-la duplicada.

**Negativas (trade-off explícito):**

- Transfere para o cliente externo a responsabilidade de implementar deduplicação por `event_id`; um cliente que não implementar essa lógica pode processar o mesmo evento de pedido mais de uma vez (ex.: disparar duas notificações internas para o mesmo status). Esse requisito precisa ser comunicado de forma clara e destacada (responsabilidade assumida pelo PM, via portal de desenvolvedor).
- Não há, nesta decisão, nenhuma validação do lado da plataforma de que o cliente de fato implementou a deduplicação — a garantia depende inteiramente da conformidade do integrador.
