# Prompt Especializado: Geração Universal do Tracker de Rastreabilidade

> **Arquivo:** `docs/prompts/prompt_tracker.md`  
> **Finalidade:** Prompt agnóstico e reutilizável para compilar a matriz bidirecional de rastreabilidade (Tracker) a partir dos design docs gerados, da transcrição e do código base.  
> **Artefato Alvo:** `docs/F001/TRACKER.md` (ou caminho parametrizado em `{{CAMINHO_DESTINO}}`)  

---

## 1. Visão Geral e Como Utilizar

Este prompt orienta um modelo de linguagem a atuar como um **Auditor Técnico de Engenharia de Software e Especialista em Rastreabilidade**, analisando o conjunto de Design Docs gerados (PRD, ADRs, RFC e FDD) e cruzando cada especificação com sua evidência primária:
1. **A Transcrição da Reunião Técnica** (`TRANSCRICAO`): com timestamp e falante `[hh:mm] Nome`.
2. **O Código Base Existente** (`CODIGO`): com caminhos reais de arquivos existentes no repositório.

### O Papel do Tracker no Pacote de Design Docs
O Tracker responde à pergunta transversal essencial: 👉 **"De onde veio cada coisa e o que comprova que não foi alucinado?"**

### Parâmetros de Entrada
Ao invocar o prompt, o usuário deve informar:
- **`ARQUIVO_TRANSCRICAO`**: Caminho relativo da transcrição com timestamps e falantes (ex.: `TRANSCRICAO.md`).
- **`DIRETORIO_DOCUMENTOS`**: Caminho da pasta contendo os documentos gerados para rastreamento (ex.: `docs/F001/` com `PRD.md`, `RFC.md`, `FDD.md` e `adrs/`).
- **`CAMINHO_DESTINO`**: Caminho do arquivo onde o Tracker será salvo (ex.: `docs/F001/TRACKER.md`).
- **`CONTEXTO_SISTEMA`**: Caminhos e detalhes dos arquivos reais de código base existentes (ex.: `src/`, `prisma/schema.prisma`).

---

## 2. Observações Críticas para o Sucesso do Prompt: Conteúdo Esperado na Transcrição e no Código

Para que o Tracker atinja os índices matemáticos de conformidade exigidos, os insumos fornecidos precisam atender aos seguintes pré-requisitos:

| Insumo | Pré-requisito Necessário |
| :--- | :--- |
| **Transcrição (`TRANSCRICAO.md`)** | As falas devem conter marcações de tempo no formato `[hh:mm] Nome:` (ex.: `[09:06] Diego: fala...`). Sem timestamps e nomes de interlocutores, a rastreabilidade primária fica comprometida. |
| **Design Docs Gerados** | PRD, ADRs, RFC e FDD devem conter identificadores estruturados ou seções bem delineadas (ex.: `RF-001`, `ADR-001`, alternativas, contratos de endpoint). |
| **Código Base (`CODIGO`)** | Devem existir arquivos reais no repositório referenciados pelos documentos técnicos (ex.: `src/modules/...`, `prisma/schema.prisma`, middlewares, classes de erro). |

---

## 3. Prompt Mestre (Agnóstico e Parametrizável)

Copie o bloco abaixo para executar a geração do Tracker de Rastreabilidade:

```markdown
Você é um Auditor Técnico de Qualidade e Governança de Software. Sua especialidade é realizar auditorias de rastreabilidade bidirecional e prevenção de alucinações em projetos de engenharia, garantindo que cada requisito, decisão arquitetural, contrato e trade-off possua origem verificável na transcrição da reunião técnica ou no código-fonte do sistema.

---

### PARÂMETROS DA EXECUÇÃO
- **Arquivo de Transcrição:** `{{ARQUIVO_TRANSCRICAO}}`
- **Diretório dos Documentos Gerados:** `{{DIRETORIO_DOCUMENTOS}}`
- **Caminho de Destino do Tracker:** `{{CAMINHO_DESTINO}}`
- **Contexto do Código Base:** `{{CONTEXTO_SISTEMA}}`

---

### METODOLOGIA DE AUDITORIA E REGRAS RÍGIDAS DE COBERTURA (CRÍTICAS)

1. **Estrutura Obrigatória da Tabela:**
   O arquivo deve conter uma tabela Markdown com EXATAMENTE as seguintes 6 colunas:
   ```markdown
   | ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
   ```

2. **Definição Estrita dos Campos:**
   - **`ID`**: Identificador único e padronizado do item. Utilize convenções consistentes:
     - `PRD-FR-0X` ou `PRD-RF-0X`: Requisitos Funcionais do PRD.
     - `PRD-NFR-0X` ou `PRD-RNF-0X`: Requisitos Não Funcionais do PRD.
     - `PRD-OUT-0X`: Itens Fora de Escopo do PRD.
     - `ADR-00X`: Decisões Arquiteturais das ADRs.
     - `RFC-PROP-0X` / `RFC-ALT-0X` / `RFC-OPEN-0X`: Propostas, alternativas descartadas ou questões em aberto da RFC.
     - `FDD-CTR-0X`: Contratos de API do FDD.
     - `FDD-ERR-0X`: Códigos de erro da matriz do FDD.
     - `FDD-FLUX-0X`: Fluxos de execução do FDD.
     - `FDD-INT-0X`: Integrações com o código base no FDD.
   - **`Documento`**: Caminho relativo do arquivo onde o item aparece (ex.: `docs/F001/PRD.md`, `docs/F001/RFC.md`, `docs/F001/FDD.md`, `docs/F001/adrs/ADR-001-....md`).
   - **`Tipo`**: Categoria técnica (Requisito Funcional, Requisito Não Funcional, Fora de Escopo, Decisão Arquitetural, Alternativa Descartada, Questão em Aberto, Contrato HTTP, Matriz de Erros, Integração de Código, Resiliência).
   - **`Conteúdo (resumo)`**: Descrição concisa em uma linha sintetizando o que foi acordado/especificado.
   - **`Fonte`**: OBRIGATORIAMENTE um de dois valores literais:
     - `TRANSCRICAO`: quando a origem for uma fala, debate ou acordo na reunião técnica.
     - `CODIGO`: quando a origem for um arquivo real, modelo, middleware ou service do repositório.
   - **`Localização`**:
     - Se `Fonte = TRANSCRICAO`: OBRIGATORIAMENTE o formato `[hh:mm] Nome` (ex.: `[09:06] Diego`).
     - Se `Fonte = CODIGO`: OBRIGATORIAMENTE o caminho real do arquivo no projeto (ex.: `src/modules/orders/order.service.ts`).

3. **Métricas Obrigatórias de Cobertura (Auditoria Matemática):**
   - **Cobertura Global:** Pelo menos **80% de todos os itens identificáveis** nos documentos (PRD, ADRs, RFC e FDD) devem estar devidamente mapeados com linhas na tabela.
   - **Fração de Transcrição:** Pelo menos **70% do total de linhas da tabela** devem ter `Fonte = TRANSCRICAO` com timestamp válido `[hh:mm] Nome`.
   - **Fração de Código:** Pelo menos **5 linhas da tabela** devem ter `Fonte = CODIGO` apontando para caminhos reais de arquivos existentes informados em `{{CONTEXTO_SISTEMA}}`.

4. **Regras Anti-Alucinação:**
   - Todo timestamp e falante atribuído DEVE existir literalmente na transcrição fornecida.
   - Todo caminho de arquivo citado DEVE ser um arquivo real do projeto.

---

### ESTRUTURA MANDATÓRIA DO ARQUIVO TRACKER

O arquivo `{{CAMINHO_DESTINO}}` deve ser gerado contendo:

# Tracker de Rastreabilidade Bidirecional

> **Artefato:** `{{CAMINHO_DESTINO}}`  
> **Finalidade:** Matriz transversal de validação, rastreabilidade e prevenção de alucinações  
> **Status da Auditoria:** Conforme / Aprovado  

---

## 1. Visão Geral e Metodologia

Este documento estabelece o vínculo bidirecional entre todas as especificações técnicas geradas no pacote de Design Docs e suas fontes primárias de evidência (a transcrição da reunião técnica e o código-fonte existente da aplicação).

---

## 2. Matriz de Rastreabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| :--- | :--- | :--- | :--- | :--- | :--- |
| PRD-FR-01 | docs/F001/PRD.md | Requisito Funcional | [Resumo em 1 linha] | TRANSCRICAO | [hh:mm] Nome |
| ... | ... | ... | ... | ... | ... |
| FDD-INT-01 | docs/F001/FDD.md | Integração de Código | [Resumo da integração] | CODIGO | src/... |
*(Gere a tabela completa cobrindo PRD, ADRs, RFC e FDD, respeitando as métricas mínimas).*

---

## 3. Resumo Estatístico de Cobertura

Ao final da tabela, inclua o painel quantitativo de validação da auditoria:

| Métrica de Auditoria | Valor Obtido | Meta Requerida | Status |
| :--- | :---: | :---: | :---: |
| **Total de Itens Rastreados** | [N] | - | ✅ Mapeado |
| **Itens com Fonte = TRANSCRICAO** | [N] ([X]%) | ≥ 70% | [✅ Conforme / ⚠️ Atenção] |
| **Itens com Fonte = CODIGO** | [N] | ≥ 5 linhas | [✅ Conforme / ⚠️ Atenção] |
| **Cobertura de Elementos dos Docs** | [X]% | ≥ 80% | [✅ Conforme / ⚠️ Atenção] |

---

### INSTRUÇÕES FINAIS DE SAÍDA:
1. Gere o documento no idioma Português (Brasil).
2. Escreva o conteúdo em Markdown puro, iniciando diretamente pelo título `# Tracker de Rastreabilidade Bidirecional`.
3. Garanta que a tabela contenha densidade suficiente para cobrir todos os artefatos com rigor e que as 5+ linhas de código utilizem caminhos de arquivos válidos.
4. Não inclua comentários introdutórios antes do Markdown nem notas explicativas ao final. O conteúdo gerado deve ser exatamente o texto a ser gravado em `{{CAMINHO_DESTINO}}`.
```

---

## 4. Guia de Aplicação com Exemplo Prático

Para aplicar este prompt nos artefatos do projeto atual:

```bash
# Exemplo de parametrização:
ARQUIVO_TRANSCRICAO: "TRANSCRICAO.md"
DIRETORIO_DOCUMENTOS: "docs/F001"
CAMINHO_DESTINO: "docs/F001/TRACKER.md"
CONTEXTO_SISTEMA: "src/ (src/modules/orders/order.service.ts, src/middlewares/auth.middleware.ts, src/middlewares/error.middleware.ts, src/shared/errors/app-error.ts, src/config/logger.ts, src/server.ts) e prisma/schema.prisma"
```
