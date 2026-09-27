# ADR-003: Retry com backoff exponencial (5 tentativas, 1m/5m/30m/2h/12h)

## Status

Aceito.

## Contexto

Como a entrega de webhooks depende de um sistema HTTP de terceiros, o cliente pode estar temporariamente indisponível no momento em que o worker (ADR-002) tenta entregar um evento da outbox (ADR-001). O time precisava definir a política de nova tentativa antes de considerar a entrega uma falha permanente.

> [09:14] Larissa: Beleza. Vamos pra retry. Se o cliente tá offline, o que a gente faz?

> [09:15] Diego: Backoff exponencial. Tenta de novo depois de algum tempo, vai aumentando o intervalo, e depois de um teto de tentativas considera falha permanente e move pra DLQ.

O número de tentativas foi discutido com dois valores concorrentes, 3 e 5, com argumentos baseados em um incidente real de indisponibilidade prolongada de um cliente:

> [09:15] Bruno: Quantas tentativas?

> [09:15] Diego: Eu sugiro 5. Algumas pessoas defendem retry indefinido com backoff, mas isso traz o problema de evento ficar pendurado pra sempre se o cliente sumiu. Cinco já dá pra cobrir uma janela de até 12 ou 24 horas.

> [09:16] Bruno: 3 não é melhor? Mais agressivo.

> [09:16] Diego: 3 é pouco. Se o cliente teve indisponibilidade de manhã, a gente retentaria três vezes em 30 minutos e mataria. Já tinha cliente nosso com indisponibilidade de duas horas em manutenção planejada.

A progressão de backoff e o total de tempo coberto foram então definidos e validados pelo PM em relação à tolerância aceitável para o cliente final:

> [09:16] Larissa: Cinco fica bom. Qual a progressão do backoff?

> [09:17] Diego: Eu pensei em 1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas. Total de quase 15 horas entre primeira falha e última tentativa.

> [09:17] Marcos: Se um cliente meu cair por 15 horas, ele já tá com problema sério dele. Acho aceitável.

> [09:17] Larissa: Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h. Próximo: DLQ. Faz numa tabela separada ou marca como "failed" na própria outbox?

Na mesma fala, Larissa já emenda para o próximo tópico — Dead Letter Queue —, tratado na [ADR-004](ADR-004-dead-letter-queue-com-replay-restrito-a-admin.md).

Complementarmente, foi definido um timeout por tentativa de entrega, tratando resposta lenta do cliente como falha sujeita a retry:

> [09:42] Sofia: Timeout do HTTP call do worker. Quanto?

> [09:42] Diego: 10 segundos. Cliente lento que não responde em 10s a gente trata como falha e marca pra retry.

## Decisão

Adotar **retry com backoff exponencial e teto fixo de 5 tentativas** para a entrega de cada evento de webhook, com os seguintes intervalos entre tentativas sucessivas: **1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas** (aproximadamente 15 horas de janela total entre a primeira falha e a última tentativa). Esgotadas as 5 tentativas sem sucesso, o evento é movido para a Dead Letter Queue (ver ADR-004) e a entrega automática cessa.

Cada tentativa de entrega tem um **timeout de 10 segundos**: se o endpoint do cliente não responder dentro desse limite, a tentativa é tratada como falha e segue para o próximo intervalo de backoff.

## Alternativas Consideradas

**1. Teto de 3 tentativas.** Considerada e descartada por ser agressiva demais: cobriria apenas cerca de 30 minutos entre tentativas, insuficiente para cenários reais de indisponibilidade prolongada já observados pelo time (ex.: cliente com manutenção planejada de duas horas), levando eventos válidos a serem descartados prematuramente para a DLQ.

**2. Retry indefinido (sem teto de tentativas).** Considerada e descartada por deixar eventos pendurados indefinidamente caso o cliente externo desapareça, sem sinalização clara de falha permanente nem necessidade de intervenção manual.

## Consequências

**Positivas:**

- Cobre indisponibilidades transitórias razoavelmente longas (até ~15 horas) sem intervenção manual, incluindo casos reais já observados pelo time, sem exigir retry indefinido.
- Falha permanente é sinalizada de forma explícita (transição para DLQ, ver ADR-004) em vez de o evento ficar pendurado indefinidamente na outbox.
- O timeout de 10 segundos por tentativa evita que um cliente lento prenda o worker além do necessário, mantendo o polling do ADR-002 fluindo.

**Negativas (trade-off explícito):**

- Um cliente que estiver indisponível por mais de ~15 horas perde o evento automaticamente (ele só é recuperável manualmente via replay da DLQ, ADR-004); a reunião considerou esse risco aceitável ("já tá com problema sério dele").
- A progressão de backoff é fixa e configurada por código/constante, não por cliente; um cliente com padrão de indisponibilidade diferente do assumido na reunião não tem tratamento diferenciado nesta primeira versão.
