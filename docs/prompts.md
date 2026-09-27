---
 - |
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
 - |
    revise novamente pra ver se isso está atendido:
    Produza entre 5 e 8 ADRs em arquivos separados dentro de docs/adrs/, nomeados no formato ADR-NNN-titulo-em-kebab-case.md (ex: ADR-001-outbox-no-mysql.md).

    Cada ADR deve seguir o formato MADR (ou variante padrão) com no mínimo as seções: Status, Contexto, Decisão, Alternativas Consideradas (pelo menos 1 alternativa real discutida ou plausível), Consequências (positivas e negativas, com trade-off explícito).

    Pelo menos 1 ADR deve referenciar explicitamente arquivos, módulos ou padrões do código existente.

    O conjunto de ADRs deve cobrir, no mínimo, 5 das 6 decisões principais discutidas na reunião:

    Padrão Outbox no MySQL
    Política de retry com backoff e DLQ
    Autenticação HMAC-SHA256 com secret por endpoint
    Garantia at-least-once com X-Event-Id
    Worker em processo separado em polling
    Reuso dos padrões existentes do projeto
    Decisões técnicas secundárias (formato de payload, timeouts, headers, entre outras) podem virar ADRs adicionais ou ficar apenas no FDD, conforme você considerar mais adequado.

 - |
    Com base em `TRANSCRICAO.md`, no código-fonte do projeto e nas ADRs já geradas, produza uma RFC (Request for Comments) para a feature de Webhooks de Notificação de Pedidos.

    A RFC deve consolidar a solução proposta de ponta a ponta, descrevendo o problema, a arquitetura, os fluxos, os impactos e as decisões adotadas. As ADRs devem ser tratadas como a fonte oficial das decisões arquiteturais.

    Estrutura mínima:

    - Título
    - Status
    - Contexto e Problema
    - Objetivos
    - Escopo
    - Arquitetura Proposta
    - Fluxo de Processamento
    - Decisões Arquiteturais
    - Alternativas Avaliadas e Descartadas
    - Questões em Aberto
    - Impactos Técnicos e Operacionais
    - Plano de Implementação
    - Referências

    Regras:

    - Referencie explicitamente as ADRs relevantes ao longo do documento, explicando como cada uma contribui para a solução final.
    - Inclua uma seção de rastreabilidade contendo a lista completa das ADRs utilizadas, com um breve resumo de cada decisão.
    - Utilize as ADRs como fonte principal das decisões já aprovadas e a transcrição como fonte complementar de contexto e justificativas.
    - Registre alternativas descartadas e tópicos discutidos que não resultaram em decisão final.
    - Toda citação associada a timestamp e falante deve ser reproduzida literalmente da transcrição.
    - Não utilize paráfrases como se fossem citações.
    - Não invente requisitos, decisões, justificativas ou detalhes técnicos não sustentados pelas ADRs, pela transcrição ou pelo código.
    - Quando houver divergência entre transcrição, código e ADRs, destaque explicitamente a inconsistência.
    - Quando não houver evidência suficiente para uma afirmação, registre a limitação.

    O resultado deve ser uma RFC autocontida, capaz de explicar a solução completa para um leitor que não tenha participado da reunião, mantendo rastreabilidade clara para as ADRs correspondentes e para as evidências da transcrição e do código.
 - |
   garanta que o que está nas ADRs e na RFC possuem link com o timestamp da trasncrição. tudo deve existir na transcrição. caso algo não bata, me avise
 - |
   garanta que a RFC e as ADRs estão consistentes entre si. caso algo não bata, me avise

