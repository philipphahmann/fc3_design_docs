# Requisitos e Critérios de Aceite do Desafio (Design Docs com IA)

> **Documento consolidado a partir das diretrizes oficiais do desafio:** `fc3_design_docs_requirements.pdf`  
> **Curso:** MBA em Engenharia de Software com IA (Full Cycle)  
> **Desafio Técnico:** Da Reunião ao Documento: Design Docs Gerados por IA

---

## 1. Visão Geral e Objetivo

### 1.1 Cenário
Uma empresa que opera um **Order Management System (OMS)** em produção vai construir uma nova feature: o **Sistema de Webhooks de Notificação de Pedidos**.  
A decisão técnica já foi tomada em uma reunião entre Tech Lead, PM, Engenheiros e Especialista de Segurança, gravada e transcrita no arquivo [`TRANSCRICAO.md`](docs/F001/TRANSCRICAO.md). Nenhuma outra documentação formal foi registrada além da transcrição da call.

### 1.2 Objetivo
Produzir, a partir da transcrição e do código-fonte existente da aplicação, um pacote completo de documentação técnica (*Design Docs*) em nível **acionável**, suficiente para que o time de engenharia possa iniciar a implementação da feature sem ambiguidades.

### 1.3 Papel da IA e do Desenvolvedor
- **IA como ferramenta principal:** leitura de código, análise da transcrição, estruturação e redação dos documentos técnicos.
- **Desenvolvedor como maestro:** definir estratégia, formular prompts específicos, conduzir revisão crítica, corrigir alucinações/superficialidades e iterar até consistência total.

---

## 2. Limitações e Regras Estritas

> [!CAUTION]
> **Entrega Puramente Documental:**
> É **estritamente proibido** alterar o código da aplicação (`src/`, `prisma/`, `tests/` ou arquivos de configuração do boilerplate). O código serve apenas como **contexto e referência**.

> [!IMPORTANT]
> **Rastreabilidade e Anti-Alucinação:**
> Toda informação registrada nos documentos **deve ser rastreável** à transcrição (`TRANSCRICAO.md`) ou ao código da aplicação. **Não é permitido inventar requisitos, decisões ou restrições** sem origem identificável.

> [!WARNING]
> **Atenção ao que NÃO entra (Fora de Escopo):**
> A reunião descarta ou adia explicitamente várias ideias (ex.: mensageria externa pesada como Kafka/RabbitMQ na v1, múltiplos eventos além de mudanças de status, retries infinitos, etc.). Identificar o que **NÃO entra** é tão importante quanto o que entra. Itens descartados/adiados não devem ser apresentados como requisitos da entrega.

---

## 3. O Pacote de Documentos e o Papel de Cada Um

Os documentos operam em diferentes níveis de abstração e não devem se duplicar:

| Documento | Papel | Altura / Nível | Pergunta que responde |
| :--- | :--- | :--- | :--- |
| **PRD** (`docs/PRD.md`) | Problema, público, escopo e métricas de sucesso | Produto / Negócio | *Por que e o quê?* |
| **RFC** (`docs/RFC.md`) | Proposta técnica da solução para revisão: abordagem geral, alternativas e questões em aberto (2 a 4 páginas) | Arquitetura | *Como pretendemos resolver, e o que ainda está em aberto?* |
| **ADRs** (`docs/adrs/ADR-*.md`) | Cada decisão arquitetural isolada com contexto e consequências | Decisão pontual | *Por que decidimos exatamente assim?* |
| **FDD** (`docs/FDD.md`) | Especificação profunda de implementação: fluxos, contratos, erros, resiliência e integração com o código existente | Implementação | *Como construir, em detalhe?* |
| **Tracker** (`docs/TRACKER.md`) | Rastreabilidade bidirecional de cada item ao código ou à transcrição | Transversal | *De onde veio cada coisa?* |
| **README** (`README.md`) | Documentação da jornada do aluno, ferramentas, prompts e iterações | Meta-processo | *Como o trabalho foi conduzido e refinado?* |

---

## 4. Requisitos Detalhados por Documento

### 4.1 PRD — Product Requirement Document (`docs/PRD.md`)
Deve seguir o formato apresentado no curso e conter, no mínimo:
1. Resumo e contexto da feature
2. Problema e motivação
3. Público-alvo e cenários de uso
4. Objetivos e métricas de sucesso (incluindo pelo menos **1 métrica/meta quantitativa**)
5. Escopo (incluso e fora de escopo — listar explicitamente pelo menos **2 itens descartados ou adiados** na reunião)
6. Requisitos funcionais (mínimo de **8 requisitos funcionais** discutidos na reunião)
7. Requisitos não funcionais
8. Decisões e trade-offs principais
9. Dependências
10. Riscos e mitigação (mínimo de **2 riscos com probabilidade, impacto e mitigação**)
11. Critérios de aceitação
12. Estratégia de testes e validação

---

### 4.2 RFC — Request for Comments (`docs/RFC.md`)
Proposta técnica concisa (**2 a 4 páginas**) submetida à equipe para revisão, respondendo "o que propomos e por quê". Deve conter:
1. **Metadados:** autor, status, data, revisores (usar os participantes da reunião como revisores)
2. **Resumo executivo (TL;DR)** da proposta
3. **Contexto e problema**
4. **Proposta técnica:** visão geral da arquitetura, sem descer ao detalhe de implementação do FDD
5. **Alternativas consideradas:** no mínimo **2 alternativas reais discutidas e descartadas** na reunião, detalhando o trade-off que motivou o descarte
6. **Questões em aberto:** no mínimo **2 pontos levantados** na reunião que foram deixados em aberto ou adiados
7. **Impacto e riscos**
8. **Decisões relacionadas:** links para pelo menos **2 ADRs** do pacote

---

### 4.3 FDD — Feature Design Document (`docs/FDD.md`)
Especificação técnica acionável para o desenvolvedor codificar. Deve conter:
1. Contexto e motivação técnica
2. Objetivos técnicos
3. Escopo e exclusões
4. **Fluxos detalhados:** criação do evento na outbox, processamento pelo worker, retry e DLQ
5. **Contratos públicos:** no mínimo **4 endpoints HTTP** completos com payloads de exemplo (request e response), headers, status codes e semântica
6. **Matriz de erros previstos:** códigos obrigatoriamente padronizados com o prefixo `WEBHOOK_*`
7. **Estratégias de resiliência:** timeouts, retries, backoff, fallback
8. **Observabilidade:** métricas, logs estruturados e tracing
9. Dependências e compatibilidade
10. Critérios de aceite técnicos
11. Riscos e mitigação técnica
12. **Seção obrigatória adicional — "Integração com o sistema existente":**
    - Deve nomear pelo menos **4 caminhos de arquivo reais** do código base (ex.: `src/modules/orders/order.service.ts`).
    - Descrever como o módulo de webhooks se integrará com cada um (ex.: extensão do método `changeStatus`, reuso de classes de erro existentes, transações do Prisma, etc.).

---

### 4.4 ADRs — Architecture Decision Records (`docs/adrs/ADR-NNN-*.md`)
- Quantidade: entre **5 e 8 ADRs** em arquivos separados no formato `ADR-NNN-titulo-em-kebab-case.md` (ex.: `ADR-001-outbox-no-mysql.md`).
- Formato: MADR (ou variante padrão) contendo:
  - **Status**
  - **Contexto**
  - **Decisão**
  - **Alternativas Consideradas** (pelo menos 1 alternativa real discutida ou plausível)
  - **Consequências** (positivas e negativas, com trade-offs explícitos)
- O conjunto deve cobrir no mínimo **5 das 6 decisões principais da reunião**:
  1. Padrão Outbox no MySQL
  2. Política de retry com backoff e DLQ
  3. Autenticação HMAC-SHA256 com secret por endpoint
  4. Garantia at-least-once com cabeçalho `X-Event-Id`
  5. Worker em processo separado operando via polling
  6. Reuso dos padrões existentes do projeto
- Pelo menos **1 ADR** deve referenciar explicitamente arquivos, módulos ou classes do código existente.

---

### 4.5 Tracker de Rastreabilidade (`docs/TRACKER.md`)
Referência cruzada essencial para validação e prevenção de alucinações da IA.
- **Formato da Tabela:**
  ```markdown
  | ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
  ```
- **Campos:**
  - `ID`: identificador único (ex: `PRD-FR-01`, `RFC-ALT-02`, `FDD-CONTRATO-03`, `ADR-002`)
  - `Documento`: arquivo onde o item aparece (ex: `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/adrs/ADR-002-....md`)
  - `Tipo`: Requisito Funcional, Requisito Não Funcional, Decisão, Restrição, Trade-off, etc.
  - `Conteúdo (resumo)`: descrição em uma linha do item
  - `Fonte`: `TRANSCRICAO` ou `CODIGO`
  - `Localização`:
    - Se `TRANSCRICAO`: timestamp + nome do falante (ex: `[09:17] Diego`)
    - Se `CODIGO`: caminho real do arquivo (ex: `src/modules/orders/order.service.ts`)
- **Regras de Cobertura do Tracker:**
  - Pelo menos **80% dos itens identificáveis** nos documentos devem estar mapeados.
  - Pelo menos **70% das linhas** devem ter `Fonte = TRANSCRICAO` com timestamp válido `[hh:mm] Nome`.
  - Pelo menos **5 linhas** devem ter `Fonte = CODIGO` com caminho de arquivo real existente no projeto.

---

### 4.6 README do Processo (`README.md`)
Substituir o README original da raiz para relatar o processo de desenvolvimento e uso da IA. Deve conter:
1. **Sobre o desafio:** 1 a 2 parágrafos sintetizando a tarefa em suas próprias palavras.
2. **Ferramentas de IA utilizadas:** lista das ferramentas com breve explicação do papel de cada uma.
3. **Workflow adotado:** ordem de produção dos documentos e dinâmica de interação com a IA.
4. **Prompts customizados:** pelo menos **2 prompts relevantes** utilizados/adaptados, apresentados em blocos de código.
5. **Iterações e ajustes:** relato dos momentos em que a IA gerou conteúdo incorreto, raso ou alucinado, detalhando as correções e quantas iterações foram necessárias.
6. **Como navegar a entrega:** caminhos dos arquivos e ordem sugerida de leitura.

---

## 5. Checklist Completo de Critérios de Aceite

### PRD (`docs/PRD.md`)
- [ ] Arquivo existe e está em Markdown
- [ ] Contém todas as seções obrigatórias especificadas
- [ ] Identifica no mínimo 8 requisitos funcionais discutidos na reunião
- [ ] Inclui pelo menos 1 objetivo com métrica e meta quantitativa
- [ ] Seção "Fora de escopo" lista pelo menos 2 itens explicitamente descartados ou adiados na reunião
- [ ] Seção "Riscos" inclui pelo menos 2 riscos com probabilidade, impacto e mitigação

### RFC (`docs/RFC.md`)
- [ ] Arquivo existe e está em Markdown
- [ ] Contém todas as seções obrigatórias especificadas
- [ ] Seção "Alternativas consideradas" lista pelo menos 2 alternativas descartadas na reunião com trade-offs
- [ ] Seção "Questões em aberto" lista pelo menos 2 pontos adiados ou não decididos na reunião
- [ ] Referencia, com link, pelo menos 2 ADRs do pacote

### FDD (`docs/FDD.md`)
- [ ] Arquivo existe e está em Markdown
- [ ] Contém todas as seções obrigatórias especificadas
- [ ] Seção "Contratos públicos" inclui pelo menos 4 endpoints HTTP com payload de exemplo (request/response) e status codes
- [ ] Matriz de erros usa códigos padronizados com o prefixo `WEBHOOK_*`
- [ ] Seção "Integração com o sistema existente" referencia pelo menos 4 caminhos de arquivo reais do código base
- [ ] Seção "Observabilidade" cita métricas, logs e tracing

### ADRs (`docs/adrs/ADR-NNN-*.md`)
- [ ] Pasta `docs/adrs/` contém entre 5 e 8 arquivos no formato `ADR-NNN-titulo-em-kebab-case.md`
- [ ] Cada ADR contém as seções Status, Contexto, Decisão, Alternativas Consideradas, Consequências
- [ ] O conjunto cobre pelo menos 5 das 6 decisões principais da reunião
- [ ] Pelo menos 1 ADR referencia explicitamente arquivos, módulos ou classes do código base

### Tracker (`docs/TRACKER.md`)
- [ ] Arquivo existe e segue o formato de tabela obrigatório
- [ ] Pelo menos 80% dos itens identificáveis dos documentos têm linha correspondente
- [ ] Pelo menos 70% das linhas têm `Fonte = TRANSCRICAO` com timestamp válido no formato `[hh:mm] Nome`
- [ ] Pelo menos 5 linhas têm `Fonte = CODIGO` com caminho de arquivo real

### README (`README.md`)
- [ ] Contém todas as seções obrigatórias especificadas
- [ ] Lista pelo menos 1 ferramenta de IA utilizada
- [ ] Mostra pelo menos 2 prompts customizados em blocos de código
- [ ] Descreve pelo menos 2 iterações ou ajustes concretos feitos durante a produção

### Consistência Geral
- [ ] Nenhum requisito, decisão ou restrição contradiz a transcrição ou o código existente
- [ ] Nenhum arquivo de código mencionado nos documentos é inexistente no repositório

---

## 6. Estrutura Obrigatória de Pastas e Arquivos

```text
.
├── README.md                 (substituído pelo aluno documentando o processo)
├── TRANSCRICAO.md            (não alterar)
├── REQUIREMENTS.md           (este guia consolidado de requisitos)
├── docs/
│   ├── PRD.md                (preenchido pelo aluno)
│   ├── RFC.md                (preenchido pelo aluno)
│   ├── FDD.md                (preenchido pelo aluno)
│   ├── TRACKER.md            (preenchido pelo aluno)
│   └── adrs/
│       ├── ADR-001-titulo-curto.md
│       ├── ADR-002-titulo-curto.md
│       ├── ADR-003-titulo-curto.md
│       ├── ADR-004-titulo-curto.md
│       ├── ADR-005-titulo-curto.md
│       └── ... (até 8 ADRs)
├── src/                      (não alterar)
├── prisma/                   (não alterar)
├── tests/                    (não alterar)
└── ... (demais arquivos do boilerplate)
```

---

## 7. Ordem de Execução Recomendada

1. **Contextualização e Leitura da Base:** Explorar o código existente (`src/`, `prisma/schema.prisma`) e ler criticamente `TRANSCRICAO.md`.
2. **ADRs primeiro:** Registrar as 5 a 8 decisões estruturais que formam o esqueleto arquitetural.
3. **RFC:** Consolidar a proposta técnica de alto nível, mapeando alternativas descartadas, questões em aberto e ligando aos ADRs.
4. **FDD:** Detalhar a engenharia da implementação (fluxos, endpoints, outbox, worker, erros `WEBHOOK_*`, arquivos do código base).
5. **PRD:** Consolidar visão de produto, requisitos funcionais, métricas quantitativas e itens fora de escopo.
6. **Tracker:** Mapear os identificadores com rastreabilidade detalhada para transcrição e arquivos de código.
7. **README do Processo:** Documentar o fluxo de engenharia de prompt, ferramentas e iterações.
8. **Checklist e Auditoria Final:** Validar item por item antes da entrega.
