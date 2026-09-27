# Da Reunião ao Documento: Design Docs Gerados por IA

> Este README documenta o processo de produção da entrega. O enunciado original do desafio está preservado em [docs/ENUNCIADO.md](docs/ENUNCIADO.md).

## Sobre o desafio

O ponto de partida era uma reunião técnica de ~55 minutos, registrada apenas como transcrição ([TRANSCRICAO.md](TRANSCRICAO.md)), em que tech lead, PM, engenheiros e segurança decidiram como construir um Sistema de Webhooks de Notificação de Pedidos para um Order Management System já em produção. A tarefa foi transformar essa conversa, junto com o código existente da aplicação, em um pacote de design docs acionável: PRD, RFC, FDD, ADRs e um tracker de rastreabilidade.

A restrição central é que nada pode ser inventado: toda decisão, requisito ou restrição precisa apontar para um momento da transcrição (timestamp + falante) ou para um arquivo real do código. Por isso, boa parte do trabalho não foi gerar texto, e sim verificar o que a IA gerou, separando o que foi decidido na reunião do que foi descartado, adiado ou simplesmente não discutido.

## Ferramentas de IA utilizadas

- **Claude Code (extensão do VS Code):** ferramenta principal. Leu a transcrição e o código, gerou todos os documentos e fez as correções a cada rodada de revisão.
- **Modelos Claude Sonnet 5 e Claude Opus 5.5:** alternados ao longo do trabalho pelo comando `/model` do Claude Code.
- **Subagente de exploração do Claude Code:** mapeou o código-fonte antes da escrita (módulos, transação do `changeStatus`, hierarquia de `AppError`, `requireRole`, logger Pino, schema Prisma), para que os documentos citassem arquivos, classes e funções reais.
- **Scripts Node.js de verificação, gerados pela IA:** conferiram automaticamente, a cada documento, que as citações batiam caractere por caractere com a transcrição, que cada timestamp e falante existia, que cada caminho de arquivo citado existia no repositório e que os totais do tracker estavam corretos.

## Workflow adotado

Segui a ordem sugerida pelo enunciado, das decisões para o detalhe:

1. **Contextualização:** leitura da transcrição e mapeamento do código pelo subagente de exploração.
2. **ADRs primeiro:** as 7 decisões principais, que formam o esqueleto do resto.
3. **RFC:** proposta de arquitetura consolidada em cima das ADRs, com alternativas descartadas e questões em aberto.
4. **FDD:** detalhamento de implementação (fluxos, contratos HTTP, matriz de erros, integração com o código).
5. **PRD:** por último entre os grandes documentos, como consolidação em nível de produto.
6. **Tracker:** varredura dos documentos prontos, ligando cada item à sua origem.
7. **README:** este documento.

A interação com a IA seguiu sempre o mesmo ciclo: um prompt de geração com regras explícitas contra invenção, seguido de um prompt de auditoria pedindo para conferir o resultado contra a transcrição, o código e os outros documentos. Cada documento só avançou depois de passar nessa auditoria. Todos os prompts usados estão registrados em [docs/prompts.md](docs/prompts.md).

## Prompts customizados

**Geração das ADRs** — define o conteúdo mínimo e as regras contra invenção que depois foram reaproveitadas nos outros documentos:

```text
Com base em `TRANSCRICAO.md` e no código-fonte do projeto, gere ADRs para a feature de Webhooks de Notificação de Pedidos.

Documente as decisões sobre: Outbox, worker com polling, retry com backoff exponencial, Dead Letter Queue com replay restrito a admin, assinatura HMAC-SHA256 com rotação de secret, entrega at-least-once com deduplicação e reuso dos padrões existentes do projeto.

Cada ADR deve conter:
- Status
- Contexto
- Decisão
- Alternativas Consideradas
- Consequências

Regras:
- Cite literalmente a transcrição, incluindo timestamp e falante.
- Referencie arquivos, classes ou funções reais do código quando aplicável.
- Baseie-se apenas em evidências da transcrição e do código.
- Não invente decisões, números, justificativas ou trechos não encontrados nas fontes.
- Se não houver evidência suficiente para uma decisão, deixe isso explícito.
```

**Geração da RFC** — trata as ADRs como fonte oficial das decisões e exige que divergências e lacunas de evidência sejam registradas, não escondidas:

```text
Com base em `TRANSCRICAO.md`, no código-fonte do projeto e nas ADRs já geradas, produza uma RFC (Request for Comments) para a feature de Webhooks de Notificação de Pedidos.

A RFC deve consolidar a solução proposta de ponta a ponta, descrevendo o problema, a arquitetura, os fluxos, os impactos e as decisões adotadas. As ADRs devem ser tratadas como a fonte oficial das decisões arquiteturais.

Regras:
- Referencie explicitamente as ADRs relevantes ao longo do documento, explicando como cada uma contribui para a solução final.
- Inclua uma seção de rastreabilidade contendo a lista completa das ADRs utilizadas, com um breve resumo de cada decisão.
- Toda citação associada a timestamp e falante deve ser reproduzida literalmente da transcrição.
- Não utilize paráfrases como se fossem citações.
- Não invente requisitos, decisões, justificativas ou detalhes técnicos não sustentados pelas ADRs, pela transcrição ou pelo código.
- Quando houver divergência entre transcrição, código e ADRs, destaque explicitamente a inconsistência.
- Quando não houver evidência suficiente para uma afirmação, registre a limitação.
```

**Auditorias** — prompts curtos, usados depois de cada geração, que encontraram a maior parte dos erros:

```text
garanta que o que está nas ADRs e na RFC possuem link com o timestamp da trasncrição. tudo deve existir na transcrição.
caso algo não bata, me avise
```

```text
garanta que a RFC e as ADRs estão consistentes entre si. caso algo não bata, me avise
```

## Iterações e ajustes

Foram **6 iterações principais** até o resultado final. Os erros mais relevantes que precisaram de correção:

1. **Citações truncadas nas ADRs.** A auditoria automática comparou as 98 citações das ADRs e da RFC com a transcrição e encontrou 3 falas cortadas ou incompletas: a fala de Larissa às `[09:12]` sem a palavra inicial "Anotado.", e as falas de `[09:17] Larissa` e `[09:28] Diego` interrompidas no meio. As três foram restauradas para a linha completa da transcrição.
2. **Paráfrase disfarçada de citação na RFC.** A primeira versão inseriu um trecho entre colchetes dentro de uma citação de Diego (`[09:39]`), o que violava a regra de citação literal. A citação foi restaurada e o contexto passou para fora do bloco.
3. **RFC longa demais e duplicando o FDD.** Ao conferir a RFC contra o formato exigido pelo enunciado, ela tinha 3.184 palavras, acima do limite de 2 a 4 páginas, e repetia no "Fluxo de Processamento" detalhes que pertencem ao FDD (headers HTTP, timeouts, campos de tabela). Foi reescrita para 1.705 palavras, com as seções renomeadas para o formato pedido e uma tabela única de decisões relacionadas no lugar de duas seções redundantes.
4. **Detalhes de implementação sem origem no FDD.** O FDD precisa ir além do que foi dito na reunião (nome de uma tabela de histórico de entregas, encoding da assinatura, biblioteca HTTP). Em vez de apresentar essas escolhas como decisões da reunião, cada uma foi marcada como "decisão desta FDD" e listada em uma seção de limitações. A auditoria também encontrou mais 2 citações truncadas no FDD, que foram corrigidas.
5. **Atribuições e números errados no PRD e no Tracker.** No PRD, um requisito citava "`[09:24]-[09:26] Diego`" quando quem fecha a decisão em `[09:26]` é a Larissa. No Tracker, a IA escreveu totais que não batiam com as linhas reais (111 linhas em vez de 143); o script de verificação detectou e os números foram corrigidos.
6. **Rodada final de refinamento.** Uma revisão do pacote completo trouxe itens que tinham ficado de fora: a questão em aberto sobre restringir quem pode configurar webhooks (`[09:37] Sofia`), mais três alternativas descartadas na RFC, um diagrama do fluxo, o reuso do envelope de paginação existente (`paginated()` em `src/shared/http/response.ts`) e do `requestId` do `request-logger.middleware.ts` para correlação de logs.

Estado final verificado por script: as 105 citações literais batem com a transcrição, todos os timestamps citados existem, e todos os caminhos de código citados existem no repositório, exceto `src/worker.ts`, o novo ponto de entrada do worker proposto na própria reunião (`[09:11] Larissa`).

## Como navegar a entrega

| Ordem | Arquivo | O que responde |
| --- | --- | --- |
| 1 | [docs/PRD.md](docs/PRD.md) | Por que e o quê: problema, público, escopo, requisitos e métricas |
| 2 | [docs/RFC.md](docs/RFC.md) | Como pretendemos resolver: arquitetura, alternativas descartadas e questões em aberto |
| 3 | [docs/adrs/](docs/adrs/) | Por que cada decisão foi tomada exatamente assim (ADR-001 a ADR-007) |
| 4 | [docs/FDD.md](docs/FDD.md) | Como construir: fluxos, contratos HTTP, erros `WEBHOOK_*` e integração com o código |
| 5 | [docs/TRACKER.md](docs/TRACKER.md) | De onde veio cada item: 150 linhas ligando os documentos à transcrição ou ao código |

ADRs do pacote:

- [ADR-001 — Padrão Outbox no MySQL](docs/adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Worker dedicado com polling](docs/adrs/ADR-002-worker-dedicado-com-polling.md)
- [ADR-003 — Retry com backoff exponencial](docs/adrs/ADR-003-retry-com-backoff-exponencial.md)
- [ADR-004 — Dead Letter Queue com replay restrito a admin](docs/adrs/ADR-004-dead-letter-queue-com-replay-restrito-a-admin.md)
- [ADR-005 — Assinatura HMAC-SHA256 com rotação de secret](docs/adrs/ADR-005-assinatura-hmac-sha256-com-rotacao-de-secret.md)
- [ADR-006 — Entrega at-least-once com deduplicação por event-id](docs/adrs/ADR-006-entrega-at-least-once-com-deduplicacao-por-event-id.md)
- [ADR-007 — Reuso de padrões existentes do projeto](docs/adrs/ADR-007-reuso-de-padroes-existentes-do-projeto.md)

Arquivos de apoio: [TRANSCRICAO.md](TRANSCRICAO.md) (fonte primária), [docs/prompts.md](docs/prompts.md) (todos os prompts usados) e [docs/ENUNCIADO.md](docs/ENUNCIADO.md) (enunciado original).
