# ADR-005: Assinatura HMAC-SHA256 com secret por endpoint e rotação com grace period

## Status

Aceito.

## Contexto

Como os webhooks entregam dados de pedidos para sistemas fora da infraestrutura da empresa, a engenheira de segurança levantou a necessidade de o cliente conseguir validar autenticidade e integridade de cada requisição recebida.

> [09:19] Sofia: Boa. Primeira coisa: a gente tá expondo eventos com dados de pedidos pra um endpoint fora da nossa infra. O cliente tem que conseguir validar que a requisição veio realmente da gente, e que ninguém adulterou o payload no meio.

> [09:20] Sofia: Padrão é HMAC. A gente assina o payload com uma secret compartilhada entre nós e o cliente, manda a assinatura num header tipo X-Signature. Cliente verifica do lado dele.

> [09:20] Bruno: HMAC com qual algoritmo?

> [09:20] Sofia: SHA-256. HMAC-SHA256 é o padrão de mercado, todo cliente sério tem biblioteca pra isso.

A segunda decisão foi que a secret não pode ser global da plataforma, e sim única por endpoint de webhook cadastrado, para conter o dano em caso de vazamento:

> [09:21] Sofia: Outra coisa importante: cada endpoint de webhook do cliente tem que ter uma secret única. Não é uma secret global da nossa plataforma. Senão se vaza uma, vaza tudo.

> [09:21] Bruno: Então a tabela de configuração de webhook armazena url + secret + customer_id + estado ativo?

> [09:21] Sofia: Sim. E a secret tem que ser rotacionável. Endpoint pro cliente conseguir pedir nova secret pela API. Quando ele rotaciona, a antiga fica válida por 24 horas em paralelo, pra ele ter tempo de migrar os sistemas dele. Depois disso, a antiga morre.

A motivação para exigir rotação (e não apenas uma secret fixa por endpoint) veio de um incidente real:

> [09:22] Diego: Isso é importante. A gente já teve cliente que vazou secret em log de aplicação dele uma vez.

> [09:22] Sofia: Pois é. Decidido: HMAC-SHA256 sobre o corpo do request, secret por endpoint, suporte a rotação com grace period de 24h.

Vale registrar que a exigência de TLS obrigatório no cadastro da URL do webhook foi discutida na mesma linha de segurança, mas o próprio time classificou-a como validação de schema, não como decisão arquitetural separada, e por isso não vira um ADR próprio:

> [09:23] Sofia: TLS obrigatório. URL do webhook tem que ser https. Se o cliente cadastrar http, recusamos com erro de validação. Isso na verdade nem é decisão arquitetural, é só uma validação no schema Zod.

## Decisão

Assinar todo payload de webhook enviado com **HMAC-SHA256**, calculado sobre o corpo (body) serializado da requisição, usando uma **secret exclusiva por endpoint de webhook cadastrado** (não uma secret global da plataforma). A assinatura resultante é enviada no header `X-Signature`; o cliente recalcula o HMAC do lado dele com a mesma secret e compara para validar autenticidade e integridade.

A tabela de configuração de webhook (`customer_id`, `url`, `secret`, estado ativo) suporta **rotação de secret sob demanda pela API**: ao rotacionar, a secret anterior permanece válida em paralelo por um **grace period de 24 horas**, permitindo ao cliente migrar seus sistemas sem downtime de validação; após esse período, apenas a nova secret é aceita.

Este módulo segue o padrão de módulos já existente no projeto (schemas de validação em Zod, ex. `src/middlewares/validate.middleware.ts` e `src/modules/orders/order.schemas.ts`), incluindo a validação de que a `url` cadastrada é obrigatoriamente `https`.

## Alternativas Consideradas

**1. Secret única e global para toda a plataforma.** Considerada e descartada por concentrar risco: o vazamento de uma única secret comprometeria a validação de assinatura de todos os clientes simultaneamente, em vez de apenas um endpoint.

**2. Secret por endpoint sem suporte a rotação (secret fixa até ser recriada manualmente).** Implicitamente descartada, pois o time já tinha um incidente concreto de vazamento de secret em log de cliente; sem rotação com grace period, a única forma de reagir a um vazamento seria invalidar a secret imediatamente, quebrando a integração do cliente até ele atualizar sua configuração — o grace period de 24h evita essa janela de indisponibilidade forçada.

## Consequências

**Positivas:**

- Cliente consegue verificar de forma criptográfica que a notificação realmente partiu da plataforma e que o payload não foi adulterado em trânsito, usando um algoritmo padrão de mercado (HMAC-SHA256) amplamente suportado por bibliotecas de terceiros.
- Isolamento de blast radius: vazamento da secret de um cliente não compromete os demais, já que cada endpoint tem sua própria secret.
- Rotação com grace period de 24h permite reagir a um vazamento de secret (cenário já ocorrido, segundo Diego) sem quebrar a integração do cliente durante a transição.

**Negativas (trade-off explícito):**

- Exige manter duas secrets válidas simultaneamente por até 24 horas após uma rotação, aumentando a complexidade de validação (o worker/endpoint de recebimento de rotação precisa checar contra ambas dentro da janela) e a superfície de armazenamento de segredos.
- A responsabilidade de guardar a secret com segurança (não logar, não expor) recai também sobre o sistema do cliente externo, fora do controle direto da plataforma — o incidente relatado por Diego mostra que esse risco é real e não totalmente mitigável do lado da plataforma.
