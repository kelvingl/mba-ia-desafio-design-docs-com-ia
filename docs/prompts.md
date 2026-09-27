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
 - |
    Produza o arquivo docs/FDD.md detalhando o "como implementar" da feature. O FDD é o documento mais técnico e precisa estar acionável o suficiente para um desenvolvedor pegar e começar a codar. Deve seguir o formato apresentado no curso e incluir, no mínimo:

    Contexto e motivação técnica
    Objetivos técnicos
    Escopo e exclusões
    Fluxos detalhados (criação do evento na outbox, processamento pelo worker, retry, DLQ)
    Contratos públicos (endpoints HTTP com payloads de exemplo, headers, status codes, semântica)
    Matriz de erros previstos com códigos no padrão WEBHOOK_*
    Estratégias de resiliência (timeouts, retries, backoff, fallback)
    Observabilidade (métricas, logs, tracing)
    Dependências e compatibilidade
    Critérios de aceite técnicos
    Riscos e mitigação
    Seção obrigatória adicional, específica deste desafio: "Integração com o sistema existente". Esta seção deve nomear pelo menos 4 caminhos de arquivo reais do código base e descrever como o módulo de webhooks vai se integrar com cada um (por exemplo, como o método changeStatus será estendido, como as classes de erro existentes serão reutilizadas).
 - |
    garanta que a RFC segue esse padrão:

    Produza o arquivo docs/RFC.md com a proposta técnica da solução, no formato de um documento submetido à equipe para revisão. O RFC opera em nível de arquitetura: apresenta a abordagem escolhida, as alternativas que foram colocadas na mesa e as questões deixadas em aberto. É um documento conciso (2 a 4 páginas); o detalhamento de implementação fica no FDD. Deve seguir o formato apresentado no curso e incluir, no mínimo:

    Metadados (autor, status, data, revisores); use os participantes da reunião como revisores
    Resumo executivo (TL;DR) da proposta
    Contexto e problema
    Proposta técnica (visão geral da solução, sem descer ao detalhe de implementação do FDD)
    Alternativas consideradas (pelo menos 2 alternativas reais discutidas e descartadas na reunião, cada uma com o trade-off que levou ao descarte)
    Questões em aberto (pelo menos 2 pontos levantados na reunião e não decididos ou adiados)
    Impacto e riscos
    Decisões relacionadas (links para os ADRs correspondentes)
    O RFC não deve duplicar o detalhamento do FDD. Ele responde "o que propomos e por quê"; o "como construir" em detalhe fica no FDD.
 - |
    agora produza a PRD:

    Produza o arquivo docs/PRD.md cobrindo a feature de Sistema de Webhooks de Notificação de Pedidos. O PRD deve seguir o formato apresentado no curso e incluir, no mínimo, as seguintes seções:

    Resumo e contexto da feature
    Problema e motivação
    Público-alvo e cenários de uso
    Objetivos e métricas de sucesso
    Escopo (incluso e fora de escopo)
    Requisitos funcionais
    Requisitos não funcionais
    Decisões e trade-offs principais
    Dependências
    Riscos e mitigação
    Critérios de aceitação
    Estratégia de testes e validação
    A seção "Fora de escopo" deve listar explicitamente pelo menos 2 itens descartados ou adiados durante a reunião.
 - |
    Agora produza o TRACKER:

    Produza o arquivo docs/TRACKER.md, uma tabela markdown que mapeia cada item registrado nos seus documentos à origem na transcrição ou no código. O tracker funciona como uma referência cruzada: permite que qualquer leitor entenda de onde veio cada decisão, requisito ou restrição, e garante que a documentação está alinhada com o que foi efetivamente discutido e com o que existe no código.

    O tracker não é um conceito padrão do mercado nem é um documento abordado diretamente no curso. É uma exigência específica deste desafio que ajuda a manter a integridade da documentação contra alucinações da IA.

    Formato obrigatório da tabela:

    ID	Documento	Tipo	Conteúdo (resumo)	Fonte	Localização
    Onde:

    ID: identificador único do item (ex: PRD-FR-01, RFC-ALT-02, FDD-CONTRATO-03, ADR-002)
    Documento: arquivo onde o item aparece (docs/PRD.md, docs/RFC.md, docs/FDD.md, docs/adrs/ADR-002-...md)
    Tipo: Requisito Funcional, Requisito Não Funcional, Decisão, Restrição, Trade-off, entre outros
    Conteúdo (resumo): descrição de uma linha do item
    Fonte: TRANSCRICAO ou CODIGO
    Localização: para TRANSCRICAO, timestamp + nome do falante (ex: [09:17] Diego). Para CODIGO, caminho do arquivo (ex: src/modules/orders/order.service.ts).
    Cobertura mínima: pelo menos 80% dos itens identificáveis nos seus documentos devem ter linha correspondente no tracker.
 - |
   depois, documente todos os prompts que eu utilizei aqui nesse arquivo
   docs\prompts.md

