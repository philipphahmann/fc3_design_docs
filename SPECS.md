# SPECS.md — Especificação e Plano Mestre de Execução (Design Docs com IA)

> **Projeto:** Full Cycle 3 — Da Reunião ao Documento: Design Docs Gerados por IA  
> **Feature F001:** Sistema de Webhooks de Notificação de Pedidos (OMS)  
> **Status Geral:** Em Planejamento / Especificação Inicial  
> **Última Atualização:** 2026-10-11  

---

## 1. Visão Geral e Objetivo

O objetivo deste projeto é transformar a gravação de alinhamento técnico transcrita em [`TRANSCRICAO.md`](file:///c:/Users/hahma/Documents/Projetos/FullCycle/FC3%20-%20Da%20Reunião%20ao%20Documento%20Design%20Docs%20Gerados%20por%20IA/fc3_design_docs/TRANSCRICAO.md) e o código base existente em um pacote profissional e acionável de **Design Docs** de engenharia de software para o **Sistema de Webhooks de Notificação de Pedidos**.

A estratégia é orientada a **Engenharia de Prompts Especializados**:
1. Desenvolver prompts estruturados e reutilizáveis (um para cada tipo de documento), contendo o papel do documento, modelo canônico, regras estritas de não-alucinação e seções mandatórias.
2. Executar os prompts sequencialmente fornecendo a transcrição e o contexto do repositório como entrada.
3. Revisar, refinar e validar cada documento contra os critérios de aceite e rastreabilidade definidos em [`REQUIREMENTS.md`](file:///c:/Users/hahma/Documents/Projetos/FullCycle/FC3%20-%20Da%20Reunião%20ao%20Documento%20Design%20Docs%20Gerados%20por%20IA/fc3_design_docs/REQUIREMENTS.md).

---

## 2. Diretrizes Inegociáveis e Limites do Projeto

> [!CAUTION]
> **Proibição Estrita de Modificação de Código:**  
> É **terminantemente vedado alterar arquivos de código da aplicação** (`src/`, `prisma/`, `tests/` ou configurações de build/docker). Todo o esforço deste projeto é estritamente documental e analítico. O código serve exclusivamente como insumo de contexto e integração.

> [!IMPORTANT]
> **Rastreabilidade Bidirecional e Anti-Alucinação:**  
> Qualquer requisito, decisão arquitetural, restrição, endpoint ou payload deve ter origem comprovável na [`TRANSCRICAO.md`](file:///c:/Users/hahma/Documents/Projetos/FullCycle/FC3%20-%20Da%20Reunião%20ao%20Documento%20Design%20Docs%20Gerados%20por%20IA/fc3_design_docs/TRANSCRICAO.md) (com timestamp `[hh:mm] Nome`) ou no código-fonte da aplicação (com caminho real de arquivo). Não inventar requisitos inexistentes.

> [!WARNING]
> **Fronteira Rígida de Escopo (O que NÃO entra):**  
> Identificar o que **não faz parte da V1** ou o que foi **descartado/adiado** na reunião é crucial (ex.: mensageria externa pesada Redis Streams/Kafka/RabbitMQ, notificações por e-mail, dashboard visual, rate limiting de envio imediato, ordering global multi-worker). Itens descartados/adiados não devem ser descritos como entregáveis da feature.

---

## 3. Estrutura de Diretórios e Nomenclatura

```text
.
├── SPECS.md                         # Guia mestre e checklist do projeto (este arquivo)
├── REQUIREMENTS.md                  # Requisitos oficiais consolidados do desafio
├── TRANSCRICAO.md                   # Transcrição da reunião técnica (fonte primária)
├── README.md                        # Relatório final da jornada com IA (substituirá o original)
│
├── docs/
│   ├── prompts/                     # Prompts especializados para geração de cada artefato
│   │   ├── prompt_prd.md            # Prompt para geração do PRD
│   │   ├── prompt_rfc.md            # Prompt para geração da RFC
│   │   ├── prompt_fdd.md            # Prompt para geração do FDD
│   │   ├── prompt_adrs.md           # Prompt para geração das ADRs
│   │   └── prompt_tracker.md        # Prompt para geração do Tracker
│   │
│   └── F001/                        # Artefatos gerados da Feature de Webhooks
│       ├── PRD.md                   # Product Requirements Document
│       ├── RFC.md                   # Request for Comments
│       ├── FDD.md                   # Feature Design Document
│       ├── TRACKER.md               # Matriz de Rastreabilidade
│       └── adrs/                    # Architecture Decision Records
│           ├── ADR-001-outbox-no-mysql.md
│           ├── ADR-002-politica-retry-backoff-dlq.md
│           ├── ADR-003-autenticacao-hmac-sha256.md
│           ├── ADR-004-entrega-at-least-once-idempotencia.md
│           ├── ADR-005-worker-processo-separado-polling.md
│           ├── ADR-006-reuso-padroes-codigo-existente.md
│           └── ADR-007-snapshot-payload-insercao-outbox.md (opcional/adicional)
│
├── src/                             # Código base existente (apenas leitura / contexto)
├── prisma/                          # Modelagem de dados atual (apenas leitura / contexto)
└── examples/PRD/                    # Exemplos de PRD fornecidos pelo curso (referência)
```

> **Nota de Alinhamento de Diretórios:** O repositório base foi inicializado com a pasta `docs/F001 - Sistema de Webhooks de Notificação de Pedidos/`. Para garantir uniformidade com a instrução do projeto (`/docs/F001`), os artefatos serão direcionados para `docs/F001/` (ou mapeados como padrão consensual).

---

## 4. Cronologia e Checklist de Desenvolvimento

A cronologia segue uma abordagem estruturada em 4 macrofases sequenciais:

```mermaid
flowchart TD
    subgraph Fase0["Fase 0: Mapeamento de Contexto"]
        F0A["Análise do Código Base (src, prisma)"] --> F0B["Mapeamento da Transcrição e Decisões"]
    end

    subgraph Fase1["Fase 1: Construção dos Prompts (/docs/prompts)"]
        P1["1. prompt_prd.md"] --> P2["2. prompt_adrs.md"]
        P2 --> P3["3. prompt_rfc.md"]
        P3 --> P4["4. prompt_fdd.md"]
        P4 --> P5["5. prompt_tracker.md"]
    end

    subgraph Fase2["Fase 2: Execução dos Prompts & Geração (/docs/F001)"]
        G1["1. Gerar docs/F001/PRD.md"] --> G2["2. Gerar docs/F001/adrs/ADR-*.md"]
        G2 --> G3["3. Gerar docs/F001/RFC.md (com links para ADRs existentes)"]
        G3 --> G4["4. Gerar docs/F001/FDD.md"]
        G4 --> G5["5. Gerar docs/F001/TRACKER.md"]
    end

    subgraph Fase3["Fase 3: Refinamento, Amarração & Auditoria"]
        R1["Validação Cruzada de Links & Consistência"] --> R2["Auditoria de Cobertura do TRACKER"]
    end

    subgraph Fase4["Fase 4: Documentação Final & Entrega"]
        E1["README.md do Processo com IA"] --> E2["Checklist Final de Submissão"]
    end

    Fase0 --> Fase1 --> Fase2 --> Fase3 --> Fase4
```

---

### Fase 0: Mapeamento de Contexto e Extração Primária
- [ ] Mapear entidades do `prisma/schema.prisma` (`Order`, `Customer`, `UserRole`, `OrderStatus`, etc.).
- [ ] Mapear arquivos-chave em `src/` para integração futura:
  - `src/modules/orders/order.service.ts` (`changeStatus`, controle de transação `prisma.$transaction`).
  - `src/middlewares/auth.middleware.ts` e `requireRole`.
  - `src/middlewares/error.middleware.ts`.
  - `src/shared/errors/app-error.ts` e classes de erro de domínio.
  - `src/config/logger.ts` (Pino).
  - `src/server.ts` (modelo de inicialização para o futuro `src/worker.ts`).
- [ ] Catalogar participantes da call: Larissa (Tech Lead), Marcos (PM), Bruno (Eng. Pedidos), Diego (Eng. Plataforma), Sofia (Segurança).
- [ ] Catalogar decisões, números e prazos:
  - Latência aceitável < 10s (polling worker a cada 2s).
  - Backoff: 5 tentativas (1m, 5m, 30m, 2h, 12h) -> DLQ em tabela separada.
  - Replay de DLQ: endpoint restrito a `ADMIN`.
  - HMAC-SHA256: secret por endpoint, rotação com 24h de transição, payload max 64KB, HTTPS obrigatório.
  - Headers: `X-Event-Id` (UUID, idempotência/dedup), `X-Signature`, `X-Timestamp`, `X-Webhook-Id`.
  - Prazo: 3 sprints (meta: fim de novembro, revisão de 2 dias úteis com Sofia).

---

### Fase 1: Criação dos Prompts Especializados (`docs/prompts/`)

Nesta fase focaremos na engenharia de cada prompt individual antes de rodar a geração dos documentos, em ordem natural e fluida:

- [ ] **1.1 Prompt de PRD (`docs/prompts/prompt_prd.md`)**
  - Papel: Definir requisitos de produto, problema, objetivos, métricas quantitativas, público e escopo.
  - Critérios embutidos no prompt:
    - Seguir padrão dos templates em `examples/PRD/`.
    - Conter todas as 12 seções mandatórias do `REQUIREMENTS.md`.
    - Mínimo de 8 Requisitos Funcionais extraídos da reunião.
    - Pelo menos 1 métrica quantitativa clara (ex: latência p95 de entrega < 10s, disponibilidade, etc.).
    - Seção "Fora de Escopo" com no mínimo 2 itens descartados/adiados (e-mails, dashboard visual, rate limiting de saída).
    - Mínimo de 2 riscos com probabilidade, impacto e mitigação.
    - Regras estritas de anti-alucinação.
  - Status: *Pendente*

- [ ] **1.2 Prompt de ADRs (`docs/prompts/prompt_adrs.md`)**
  - Papel: Registrar decisões arquiteturais individuais e isoladas no formato MADR antes da proposta geral da RFC.
  - Critérios embutidos no prompt:
    - Formato padrão MADR (Status, Contexto, Decisão, Alternativas Consideradas, Consequências).
    - Quantidade: entre 5 e 8 ADRs independentes (`ADR-001` a `ADR-00N`).
    - Cobertura obrigatória de pelo menos 5 das 6 decisões-chave:
      1. Outbox Pattern no MySQL existente;
      2. Política de Retry exponencial (1m/5m/30m/2h/12h) e Dead Letter Queue (DLQ);
      3. Autenticação HMAC-SHA256 com secret rotacionável por endpoint;
      4. Garantia at-least-once com cabeçalho `X-Event-Id` para desduplicação;
      5. Worker desacoplado em processo separado com polling de 2 segundos;
      6. Reuso integral dos padrões da codebase (`AppError`, Pino, `requireRole`, Zod).
    - Pelo menos 1 ADR referenciando diretamente arquivos/classes do código base.
  - Status: *Pendente*

- [ ] **1.3 Prompt de RFC (`docs/prompts/prompt_rfc.md`)**
  - Papel: Proposta arquitetural concisa (2 a 4 páginas) para discussão e revisão da equipe, integrando as decisões já registradas nas ADRs.
  - Critérios embutidos no prompt:
    - Metadados com revisores reais da call (Larissa, Marcos, Bruno, Diego, Sofia).
    - Resumo executivo (TL;DR).
    - Proposta técnica de alto nível (Outbox + Worker polling + HMAC-SHA256 + At-least-once).
    - No mínimo 2 alternativas reais descartadas com análise de trade-offs (ex: disparo síncrono no `OrderService`, mensageria externa Redis Streams/Kafka, triggers de banco de dados).
    - No mínimo 2 questões em aberto ou pontos futuros (ex: rate limiting de disparo para clientes com muitas mudanças, particionamento/escala multi-worker).
    - Links diretos para no mínimo 2 ADRs já consolidadas no pacote.
  - Status: *Pendente*

- [ ] **1.4 Prompt de FDD (`docs/prompts/prompt_fdd.md`)**
  - Papel: Especificação técnica profunda e acionável para o desenvolvedor codificar.
  - Critérios embutidos no prompt:
    - Fluxos detalhados: criação do evento na outbox, ciclo do worker de polling, política de retry e migração para DLQ.
    - Contratos de API: no mínimo 4 endpoints HTTP completos (ex: POST `/webhooks`, GET `/webhooks`, GET `/webhooks/:id/deliveries`, POST `/admin/webhooks/dead-letter/:id/replay`, rotação de secret).
    - Matriz de erros obrigatória com códigos padronizados no prefixo `WEBHOOK_*`.
    - Resiliência e observabilidade (métricas, logs estruturados com Pino, tracing).
    - Seção mandatória de integração nomeando e integrando no mínimo 4 arquivos reais do código base (`src/modules/orders/order.service.ts`, etc.).
  - Status: *Pendente*

- [ ] **1.5 Prompt do TRACKER (`docs/prompts/prompt_tracker.md`)**
  - Papel: Matriz bidirecional de rastreabilidade para auditoria e controle de alucinações.
  - Critérios embutidos no prompt:
    - Estrutura de tabela com 6 colunas estritas: `| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |`.
    - IDs padronizados (`PRD-FR-*`, `ADR-00*`, `RFC-ALT-*`, `FDD-CONTRATO-*`, etc.).
    - Cobertura mínima de 80% dos elementos dos documentos gerados.
    - Mínimo de 70% das linhas apontando para `Fonte = TRANSCRICAO` com `[hh:mm] Nome`.
    - Mínimo de 5 linhas apontando para `Fonte = CODIGO` com caminhos reais de arquivos existentes.
  - Status: *Pendente*

---

### Fase 2: Execução dos Prompts e Geração dos Documentos Técnicos (`docs/F001/`)

Execução sequencial dos prompts gerados, garantindo que cada documento se apoie nos artefatos precedentes:

- [ ] **2.1 Execução do Prompt PRD -> `docs/F001/PRD.md`**
  - Executar prompt usando `TRANSCRICAO.md` e referências de produto.
  - Revisar seções: problema, personas, métricas quantitativas, escopo (incluso/excluso), 8+ requisitos funcionais, riscos e estratégias de teste.
  - Status: *Pendente*

- [ ] **2.2 Execução do Prompt ADRs -> `docs/F001/adrs/ADR-001` a `ADR-00N.md`**
  - Executar prompt gerando entre 5 e 8 arquivos `.md` individuais.
  - Validar estrutura MADR e cobrir as decisões de outbox, retry/DLQ, HMAC, idempotência, worker isolado e arquitetura modular existente.
  - Status: *Pendente*

- [ ] **2.3 Execução do Prompt RFC -> `docs/F001/RFC.md`**
  - Executar prompt combinando `PRD.md`, as ADRs recém-geradas e `TRANSCRICAO.md`.
  - Garantir extensão entre 2 e 4 páginas, tom de proposta técnica aberta, alternativas descartadas, questões em aberto e links diretos para os arquivos de ADRs existentes.
  - Status: *Pendente*

- [ ] **2.4 Execução do Prompt FDD -> `docs/F001/FDD.md`**
  - Executar prompt combinando `PRD.md`, `RFC.md`, as ADRs, `TRANSCRICAO.md` e código existente (`src/`, `prisma/schema.prisma`).
  - Validar detalhamento dos 4+ contratos HTTP, matriz com códigos `WEBHOOK_*`, estratégias de backoff e os 4+ arquivos reais de integração.
  - Status: *Pendente*

- [ ] **2.5 Execução do Prompt TRACKER -> `docs/F001/TRACKER.md`**
  - Compilar todos os IDs de requisitos funcionais, decisões das ADRs, alternativas da RFC e contratos do FDD.
  - Mapear cada linha com timestamp `[hh:mm] Nome` ou caminho de arquivo real do repositório.
  - Validar índices de cobertura (>= 80% total, >= 70% transcrição, >= 5 código).
  - Status: *Pendente*

---

### Fase 3: Amarração Cruzada e Auditoria de Consistência
- [ ] **Validação de Links Cruzados:** Garantir que links da RFC (`docs/F001/RFC.md`) para as ADRs (`adrs/ADR-*.md`) funcionem perfeitamente.
- [ ] **Consistência Semântica:** Verificar se nomes de tabelas (`webhook_outbox`, `webhook_dead_letter`), payloads e nomes de headers (`X-Event-Id`, `X-Signature`, etc.) estão 100% alinhados entre PRD, ADRs, RFC e FDD.
- [ ] **Auditoria de Anti-Alucinação:** Checar se nenhum arquivo inventado foi referenciado e se nenhum requisito descartado na call foi colocado como requisito da v1.
- [ ] **Auditoria Numérica do Tracker:** Conferir proporções de linhas da transcrição e do código.

---

### Fase 4: Documentação do Processo e Preparação de Entrega
- [ ] **Geração do `README.md` Raiz:**
  - Substituir o README do boilerplate pelo relato detalhado da jornada de engenharia com IA.
  - Descrição da dinâmica de maestro (definição, revisão crítica, iterações e refinamento).
  - Ferramentas de IA utilizadas e papel de cada uma.
  - Workflow adotado (cronologia de prompts e geração: PRD -> ADRs -> RFC -> FDD -> TRACKER).
  - Blocos de código com pelo menos 2 prompts customizados reais.
  - Pelo menos 2 iterações concretas relatando correções de alucinações ou aprofundamento técnico.
  - Guia de navegação completo para os avaliadores.
- [ ] **Checklist de Aceite Final:** Revisão de todos os 25 itens do checklist oficial de `REQUIREMENTS.md`.

---

## 5. Matriz de Critérios de Aceite por Documento

| Documento | Caminho | Regras Críticas Obrigatórias | Status |
| :--- | :--- | :--- | :--- |
| **PRD** | `docs/F001/PRD.md` | 12 seções; ≥ 8 Requisitos Funcionais; ≥ 1 meta quantitativa; ≥ 2 fora de escopo; ≥ 2 riscos com probabilidade/impacto. | ⏳ Pendente |
| **ADRs** | `docs/F001/adrs/ADR-*.md` | 5 a 8 arquivos MADR; cobrir ≥ 5 das 6 decisões nucleares da call; ≥ 1 ADR com referência explícita a arquivos/módulos existentes. | ⏳ Pendente |
| **RFC** | `docs/F001/RFC.md` | 2 a 4 páginas; revisores reais da call; TL;DR; ≥ 2 alternativas descartadas com trade-offs; ≥ 2 questões em aberto; links para ≥ 2 ADRs existentes. | ⏳ Pendente |
| **FDD** | `docs/F001/FDD.md` | Fluxos de outbox/worker/retry/DLQ; ≥ 4 contratos HTTP completos com payloads; matriz de erros prefixada `WEBHOOK_*`; ≥ 4 caminhos reais de código base; observabilidade. | ⏳ Pendente |
| **TRACKER** | `docs/F001/TRACKER.md` | Tabela com 6 colunas; ≥ 80% de cobertura de itens; ≥ 70% com `[hh:mm] Nome`; ≥ 5 com caminho real de código. | ⏳ Pendente |
| **README** | `README.md` | Jornada do processo; ferramentas de IA; workflow; ≥ 2 prompts em blocos de código; ≥ 2 iterações/correções documentadas; guia de navegação. | ⏳ Pendente |

---

## 6. Pontos de Atenção e Decisões de Alinhamento

1. **Caminho da Pasta de Artefatos:**  
   - O projeto possui `docs/F001 - Sistema de Webhooks de Notificação de Pedidos/`.  
   - Definido pelo usuário: direcionar para `/docs/F001`. Manteremos a padronização alinhada para evitar arquivos duplicados ou referências quebradas.
2. **Fluxo Natural (ADRs antes da RFC):**  
   - Ao gerar as ADRs antes da RFC, garantimos um fluxo 100% natural e sem antecipações artificiais: as decisões arquiteturais pontuais (`ADR-001` a `ADR-00N`) já estarão consolidadas e salvas no diretório `adrs/`, permitindo que a RFC linke diretamente para os arquivos reais existentes, e que o FDD posterior se baseie em ambas.
3. **Insumo de Código Obrigatório:**  
   - Embora o arquivo `TRANSCRICAO.md` contenha as discussões, o código em `src/` e `prisma/schema.prisma` é insumo obrigatório para FDD (4 caminhos de arquivo), ADRs e TRACKER (mínimo de 5 linhas).
