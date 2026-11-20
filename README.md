# Da Reunião ao Documento — Design Docs do Sistema de Webhooks

## Sobre o desafio

Este repositório nasceu de um exercício de engenharia de documentação: transformar a transcrição de uma reunião técnica de aproximadamente 55 minutos em um pacote de design docs acionável para o time de desenvolvimento. O cenário é um Order Management System (OMS) em Node.js/TypeScript que vai ganhar um **Sistema de Webhooks de Notificação de Pedidos** — e a única fonte de decisão disponível era a conversa entre produto, engenharia e segurança.

A tarefa foi ler a transcrição e o código existente, separar o que foi **decidido**, **descartado** e **adiado**, e produzir PRD, RFC, FDD, ADRs e um tracker de rastreabilidade. A regra central: nenhum requisito, decisão ou restrição poderia existir sem origem identificável na transcrição (`[hh:mm] Nome`) ou em um arquivo real do repositório. A entrega é puramente documental; `src/`, `prisma/`, `tests/` e configurações não foram alterados.

---

## Ferramentas de IA utilizadas

| Ferramenta | Papel na produção |
| --- | --- |
| **OpenCode** | Ambiente de agente usado para ler o repositório, navegar entre arquivos, executar buscas e escrever os documentos Markdown. |
| **DeepSeek V4.1 Flash Free** (modelo via OpenCode) | Modelo de linguagem responsável por analisar a transcrição, mapear o código, propor a estrutura dos documentos e gerar o conteúdo inicial. |
| **Leitura direta do repositório** | Verificação factual de caminhos, classes, métodos e padrões citados nos documentos — usada como contrapeso às respostas do modelo. |
| **Revisão manual dirigida** | Ciclos de crítica em que eu comparava cada afirmação com a transcrição ou com o código e pedia correção pontual. |

A IA foi usada como **coprodutora**, não como fonte da verdade. Toda saída passou por conferência contra `TRANSCRICAO.md` e contra os arquivos do projeto antes de ser aceita.

---

## Workflow adotado

A ordem de produção seguiu a lógica de que decisões vêm antes de narrativa: primeiro o esqueleto técnico, depois os documentos que o explicam.

1. **Contextualização** — leitura integral de `requisitos_desafio.txt`, `TRANSCRICAO.md` e dos arquivos centrais: `src/modules/orders/order.service.ts`, `order.status.ts`, `prisma/schema.prisma`, `src/shared/errors/`, `src/middlewares/`, `src/app.ts`, `src/routes/index.ts`, `src/shared/logger/index.ts`, `src/server.ts`, `src/config/database.ts` e os testes.
2. **Extração de decisões** — varredura da transcrição em busca de falas que fechavam decisão, descartavam alternativa ou adiavam escopo. Cada candidato foi anotado com timestamp e falante.
3. **ADRs (7 arquivos)** — produção das decisões isoladas em `docs/adrs/`, no formato MADR, cada uma com alternativas reais e trade-offs explícitos. Essa etapa definiu o vocabulário usado no resto do pacote.
4. **RFC** — consolidação da proposta em nível de arquitetura, com alternativas descartadas, questões em aberto e links para os ADRs. Sem descer ao detalhe de contrato.
5. **FDD** — detalhamento de fluxos, modelo de dados, contratos HTTP, matriz de erros `WEBHOOK_*`, resiliência, observabilidade e a seção obrigatória de integração com caminhos reais do código.
6. **PRD** — consolidação de problema, público, escopo, métricas, requisitos e riscos, já coerente com o que RFC, FDD e ADRs tinham fechado.
7. **Tracker** — varredura dos documentos prontos, atribuindo ID a cada item e preenchendo fonte e localização. Itens sem origem identificável foram corrigidos ou removidos.
8. **README do processo** — este documento, escrito por último.
9. **Revisão final** — checklist de aceite item por item, conferência de links, contagem de ADRs, caminhos citados e consistência entre documentos.

A interação com a IA foi organizada em **prompts de escopo estreito**: em vez de pedir "gere o PRD", cada rodada pedia uma coisa verificável — mapear decisões com timestamp, listar alternativas descartadas, preencher a matriz de erros, auditar o tracker. Isso reduziu respostas genéricas e facilitou a checagem.

---

## Prompts customizados

### Prompt 1 — Extração de decisões e descartes da transcrição

```text
Leia TRANSCRICAO.md integralmente e produza uma tabela com três blocos:

1. DECISÕES FECHADAS: o que foi decidido, quem decidiu e o timestamp exato.
2. ALTERNATIVAS DESCARTADAS: qual era a alternativa, qual trade-off motivou o descarte
   e o timestamp. Não invente alternativas que não apareçam na conversa.
3. ITENS ADIADOS OU FORA DE ESCOPO: o que ficou para depois, com a fala de origem.

Regras:
- Use somente o formato [hh:mm] Nome na coluna de origem.
- Se uma fala for ambígua, marque como "ambíguo" em vez de escolher por conta própria.
- Não transforme sugestão em decisão.
- No fim, liste separadamente as falas que mencionam código existente
  (métodos, transações, erros, middleware, logger), com o caminho provável do arquivo.
```

Esse prompt foi o mais importante do processo: ele produziu o material bruto que alimentou ADRs, tracker e a seção "Fora de escopo" do PRD.

### Prompt 2 — ADR com rastreabilidade e trade-off explícito

```text
Escreva docs/adrs/ADR-NNN-<titulo-em-kebab-case>.md no formato MADR, em português.

Contexto: reunião técnica sobre webhooks de pedidos. A decisão é <decisão>.
Fontes permitidas: as falas [hh:mm] Nome de TRANSCRICAO.md e os arquivos reais do repositório.

Exija de si mesmo:
- Status, Contexto, Decisão, Alternativas Consideradas e Consequências.
- Pelo menos uma alternativa REAL discutida na reunião, com o trade-off que levou ao descarte.
- Consequências positivas E negativas, com o custo aceito explicitamente.
- Nenhuma biblioteca, tabela ou endpoint inventado. Se precisar propor algo de implementação,
  marque como "proposto no FDD" e ancore na decisão da reunião.
- Referências finais com timestamps e caminhos de arquivo que existam no repositório.
```

### Prompt 3 — Auditoria do FDD contra o código

```text
Audite docs/FDD.md contra o repositório real.

Verifique:
1. Todo caminho de arquivo citado existe? Liste os que não existem.
2. Os métodos e classes citados (changeStatus, AppError, requireRole, createPrismaClient,
   paginated, redact) existem com esses nomes?
3. A seção "Integração com o sistema existente" cobre pelo menos 4 arquivos reais
   e explica a integração, não apenas lista caminhos?
4. Os contratos HTTP têm request, response e status codes para pelo menos 4 endpoints?
5. A matriz de erros usa exclusivamente o prefixo WEBHOOK_* e não contradiz os erros já
   existentes em src/shared/errors/?

Para cada problema, aponte a linha e proponha a correção mínima. Não reescreva o documento inteiro.
```

---

## Iterações e ajustes

Foram **cinco ciclos principais** de geração, revisão e correção. Os momentos mais relevantes:

### 1. Separação entre decidido, descartado e adiado

A transcrição mistura decisões fechadas com ideias exploratórias. Rate limiting, e-mail de alerta e dashboard chegaram a ser mencionados, mas não foram aprovados como escopo. A correção foi classificar cada fala em **decidido**, **descartado** ou **adiado** antes de escrever qualquer documento. O efeito aparece no PRD: e-mail, rate limiting, dashboard, webhooks inbound, arquivamento, multi-worker e endurecimento de roles ficaram em **Fora de escopo**; os requisitos funcionais receberam apenas o que foi efetivamente combinado.

### 2. Ambiguidade de "5 tentativas" no retry

A transcrição fala em "5 tentativas" e lista cinco intervalos de espera, o que é internamente ambíguo. Interpretar como 5 chamadas totais não fecharia com a janela de "quase 15 horas" mencionada na reunião. A solução foi adotar **tentativa inicial + 5 retries** (1 m / 5 m / 30 m / 2 h / 12 h, janela de 14 h 36 min), registrar a leitura como **nota de interpretação** no ADR-003 e repetir a mesma leitura no RFC e no FDD. A ambiguidade não foi escondida: ficou documentada e justificada.

### 3. Fronteira entre RFC e FDD

O RFC corria o risco de virar uma versão menor do FDD, repetindo tabelas, payloads e código. O critério de corte passou a ser explícito: o RFC responde **o que propomos e por quê**, com alternativas e questões em aberto; contratos, modelos, fluxos e snippets ficam no FDD. Sempre que um parágrafo só fazia sentido para quem vai codar, ele foi movido para o FDD.

### 4. Caminhos de código fictícios e arquivos futuros

A seção "Integração com o sistema existente" precisava apontar apenas arquivos que existem. A auditoria contra o repositório confirmou os caminhos reais (`src/modules/orders/order.service.ts`, `src/modules/orders/order.status.ts`, `src/shared/errors/http-errors.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/auth.middleware.ts`, `src/app.ts`, `src/routes/index.ts`, `src/config/database.ts`, `src/server.ts`, `src/shared/logger/index.ts`, `prisma/schema.prisma`, `package.json`, `tests/`) e separou o que é módulo novo: `src/worker.ts` e `src/modules/webhooks/*` aparecem marcados como **"a criar"**, para não se confundirem com o que já existe.

### 5. Tracker com números inconsistentes e ADRs sem seção explícita

A última auditoria encontrou dois problemas concretos. Primeiro, o resumo de cobertura do tracker dizia 245 itens, mas a tabela tinha **260** linhas; os números foram corrigidos para 226 `TRANSCRICAO` (86,9%) e 34 `CODIGO` (13,1%). Segundo, os ADRs traziam o status como bullet no cabeçalho, e não como seção; os sete arquivos ganharam a seção `## Status`. Também foram verificados os 226 timestamps `[hh:mm] Nome` contra a transcrição e os 34 caminhos `CODIGO` contra o disco — nenhum inválido.

---

## Como navegar a entrega

```text
.
├── README.md                  # este documento (processo de produção)
├── TRANSCRICAO.md             # fonte original (não alterada)
├── docs/
│   ├── PRD.md                 # problema, público, escopo, métricas e riscos
│   ├── RFC.md                 # proposta técnica, alternativas e questões em aberto
│   ├── FDD.md                 # implementação, contratos, erros e integração
│   ├── TRACKER.md             # rastreabilidade item a item
│   └── adrs/
│       ├── ADR-001-padrao-outbox-no-mysql.md
│       ├── ADR-002-worker-separado-em-polling.md
│       ├── ADR-003-politica-de-retry-e-dlq.md
│       ├── ADR-004-autenticacao-hmac-sha256-por-endpoint.md
│       ├── ADR-005-garantia-at-least-once-com-x-event-id.md
│       ├── ADR-006-reuso-dos-padroes-existentes.md
│       └── ADR-007-snapshot-do-payload-na-insercao.md
├── src/                       # código existente (não alterado)
├── prisma/                    # schema existente (não alterado)
└── tests/                     # testes existentes (não alterado)
```

**Ordem sugerida de leitura:**

1. **`docs/PRD.md`** — entenda o problema, os clientes, o escopo e as metas antes de entrar em tecnologia.
2. **`docs/RFC.md`** — veja a proposta em nível de arquitetura, as alternativas descartadas e o que ficou em aberto.
3. **`docs/adrs/`** — aprofunde cada decisão isolada; o conjunto cobre as seis decisões principais da reunião e uma decisão secundária (snapshot do payload).
4. **`docs/FDD.md`** — use como guia de implementação: fluxos, modelo de dados, contratos, erros, resiliência, observabilidade e integração com o código.
5. **`docs/TRACKER.md`** — confira a origem de qualquer item e use como ferramenta de auditoria contra alucinações.
6. **`TRANSCRICAO.md`** — a fonte primária, para quem quiser conferir o contexto original das falas.

### Escopo da entrega

- **Documentos produzidos:** PRD, RFC, FDD, 7 ADRs e Tracker.
- **Código da aplicação:** não alterado. A feature é especificada, não implementada.
- **Rastreabilidade:** todo item dos documentos tem linha no tracker, com timestamp `[hh:mm] Nome` ou caminho de arquivo real.
- **Decisões cobertas:** outbox no MySQL, worker separado em polling, retry com backoff e DLQ, HMAC-SHA256 por endpoint, at-least-once com `X-Event-Id`, reuso dos padrões existentes e snapshot do payload.

---

## Referência ao enunciado original

O enunciado completo do desafio está preservado em `requisitos_desafio.txt` e no histórico do repositório base: [devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia). Este README o substitui como documentação do processo de produção.
