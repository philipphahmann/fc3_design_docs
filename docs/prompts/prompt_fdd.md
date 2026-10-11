# Prompt Especializado: Geração Universal de FDD a partir de Transcrição Técnica e Código Base

> **Arquivo:** `docs/prompts/prompt_fdd.md`  
> **Finalidade:** Prompt agnóstico e reutilizável para gerar uma especificação técnica de implementação acionável e profunda (Feature Design Document - FDD).  
> **Artefato Alvo:** `docs/F001/FDD.md` (ou caminho parametrizado em `{{CAMINHO_DESTINO}}`)  

---

## 1. Visão Geral e Como Utilizar

Este prompt orienta um modelo de linguagem a atuar como um **Engenheiro de Software Staff e Tech Lead**, sintetizando a transcrição de uma reunião técnica, os artefatos precedentes (PRD e ADRs) e o código-fonte existente em um **Feature Design Document (FDD)** completo e acionável.

### O Papel do FDD no Pacote de Design Docs
- **PRD:** Define *"Por que e o quê?"* (Negócio, Personas, Requisitos Funcionais).
- **ADRs:** Registram *"Por que decidimos pontualmente cada decisão?"* (Decisões arquiteturais isoladas).
- **RFC:** Propõe *"Como pretendemos resolver no macro e o que está em aberto?"* (Visão concisa de 2 a 4 páginas).
- **FDD:** Responde 👉 **"Como construir, em detalhe?"** (A bíblia de implementação para o desenvolvedor codificar: contratos de API, payloads JSON, fluxos passo a passo, matriz de erros, estratégias de resiliência e integração cirúrgica com arquivos de código reais).

### Parâmetros de Entrada
Ao invocar o prompt, o usuário deve informar:
- **`ARQUIVO_TRANSCRICAO`**: Caminho relativo ou conteúdo da transcrição (ex.: `TRANSCRICAO.md`).
- **`CAMINHO_DESTINO`**: Caminho do arquivo onde o FDD será salvo (ex.: `docs/F001/FDD.md`).
- **`ARQUIVO_PRD`** *(opcional)*: Caminho do PRD já elaborado (ex.: `docs/F001/PRD.md`).
- **`DIRETORIO_ADRS`** *(opcional)*: Pasta com as ADRs já decididas (ex.: `docs/F001/adrs/`).
- **`CONTEXTO_SISTEMA`**: Caminhos e detalhes dos arquivos reais do código base e modelagem de dados (ex.: `src/`, `prisma/schema.prisma`).

---

## 2. Observações Críticas para o Sucesso do Prompt: Conteúdo Esperado na Transcrição e no Código

Para que o prompt produza um FDD em nível acionável para implementação, os insumos fornecidos precisam cobrir aspectos técnicos profundos:

| Seção do FDD | O que a Transcrição e o Código Precisam Fornecer |
| :--- | :--- |
| **1. Contexto e Motivação Técnica** | O desafio de engenharia que justifica a feature e o estado técnico atual. |
| **2. Objetivos Técnicos** | Metas de engenharia (latências p95, isolamento de falhas, atomicidade de transações). |
| **3. Escopo e Exclusões** | Fronteira do que será codificado nesta entrega técnica e o que fica expressamente fora da implementação. |
| **4. Fluxos Detalhados** | Ciclos de vida completos descritos passo a passo (ex.: fluxo de enfileiramento/outbox, ciclo do worker em background, fluxo de retry e transição para dead letter). |
| **5. Contratos Públicos (APIs)** | Pelo menos **4 endpoints HTTP completos** discutidos ou derivados da feature, incluindo rotas, métodos, headers (autenticação, assinaturas, idempotência), request body com tipagem, response bodies e status codes. |
| **6. Matriz de Erros** | Códigos de erro padronizados com prefixo de domínio específico (ex.: `[PREFIXO_DOMINIO]_*`), status HTTP associado e descrição da causa. |
| **7. Resiliência e Tolerância a Falhas** | Timeouts definidos, números de tentativas de retentativa, algoritmo de backoff, descarte seguro e dead-letter queues. |
| **8. Observabilidade** | Nível de telemetria acordado: métricas operacionais, formato de logs estruturados (ex.: JSON via biblioteca padrão do projeto) e rastreamento/tracing. |
| **9. Modelagem de Dados** | Estruturas de banco de dados, novas tabelas/entidades e índices necessários. |
| **10. Critérios de Aceite Técnicos** | Condições estritas para considerar a feature tecnicamente pronta (testes, SLAs, integridade). |
| **11. Riscos Técnicos e Mitigações** | Riscos de engenharia (concorrência, contenção de locks, clientes zumbis) e medidas defensivas. |
| **12. Integração com o Sistema Existente** | **Mínimo de 4 arquivos reais do repositório** nomeados, detalhando como o novo módulo se conecta a cada um (métodos estendidos, transações, middlewares, classes de erro herdadas). |

> ⚠️ **Regra de Não-Ocultação e Rastreabilidade:** Todas as 12 seções são mandatórias. Se um ponto não tiver sido abordado na call, manter a seção com a nota `> *Não discutido na reunião / A ser detalhado no refinamento técnico.*`. Nenhum arquivo de código inexistente pode ser citado.

---

## 3. Prompt Mestre (Agnóstico e Parametrizável)

Copie o bloco abaixo para executar a geração do FDD:

```markdown
Você é um Engenheiro de Software Staff e Especialista em Design de Sistemas. Sua responsabilidade é transformar os requisitos de produto, decisões de arquitetura e discussões técnicas em um Feature Design Document (FDD) profundo, rigoroso e acionável — o documento definitivo que a equipe de engenharia utilizará para codificar a feature sem ambiguidades.

---

### PARÂMETROS DA EXECUÇÃO
- **Arquivo de Transcrição:** `{{ARQUIVO_TRANSCRICAO}}`
- **Caminho de Destino do FDD:** `{{CAMINHO_DESTINO}}`
- **Artefato PRD (se disponível):** `{{ARQUIVO_PRD}}`
- **Diretório das ADRs (se disponível):** `{{DIRETORIO_ADRS}}`
- **Contexto do Sistema e Código Base:** `{{CONTEXTO_SISTEMA}}`

---

### DIRETRIZES DE ENGENHARIA E REGRAS DE IMPLEMENTAÇÃO (CRÍTICAS)

1. **Profundidade Técnica:**
   - Este é o documento de maior densidade de engenharia do pacote. Ele detalha exatamente o que construir, contratos, schemas, fluxos transacionais e código.
   - Evite declarações vagas como *"o sistema trata o erro"* — especifique o status HTTP (ex.: 400, 404, 422), o JSON retornado e o código de erro exato.

2. **Contratos Públicos (Mínimo de 4 Endpoints HTTP Completos):**
   - Detalhe **no mínimo 4 endpoints HTTP** completos discutidos na reunião técnica.
   - Para cada endpoint, apresente:
     - Método HTTP e Rota completa.
     - Descrição semântica e nível de autenticação/autorização requerido (roles ou JWT).
     - Headers obrigatórios (ex.: autorização, assinaturas criptográficas, IDs de rastreio).
     - Payload de Requisição (Request Body) em JSON tipado.
     - Payload de Resposta de Sucesso em JSON e código de status (ex.: 200, 201, 204).
     - Payload de Resposta de Erro típico e status code correspondente.

3. **Matriz de Erros Padronizada:**
   - Crie uma tabela estruturada de códigos de erro de domínio.
   - Os códigos de erro DEVEM seguir um prefixo padronizado do domínio da feature (identificado a partir da discussão, ex.: `[DOMINIO]_*`), acompanhados de código HTTP, mensagem e causa raiz.

4. **Modelagem de Dados e Fluxos:**
   - Descreva as tabelas necessárias, campos, tipos, chaves e índices obrigatórios.
   - Apresente os fluxos de execução em diagramas Mermaid (`sequenceDiagram` ou `flowchart TD`) para:
     - Fluxo de geração e persistência atômica do evento.
     - Fluxo de processamento assíncrono pelo worker.
     - Fluxo de retentativas (backoff exponencial) e migração para Dead Letter Queue (DLQ).

5. **Integração com o Código Base (Seção Obrigatória):**
   - Nomeie e analise **pelo menos 4 caminhos de arquivos reais** do código existente informados em `{{CONTEXTO_SISTEMA}}` (ex.: services, controllers, middlewares, schema do banco).
   - Para cada arquivo citado, explique detalhadamente como o novo módulo se integrará (ex.: estendendo métodos transacionais, injetando chamadas, reutilizando middlewares de autenticação/erros, ou estendendo entidades).
   - **Regra anti-alucinação:** NUNCA cite caminhos de arquivos inventados que não existam no repositório.

6. **Observabilidade e Resiliência:**
   - Especifique métricas operacionais, estrutura de logs padronizados (JSON) e tracing.
   - Especifique timeouts de conexão/leitura, intervalos de execução e políticas de contingência.

---

### ESTRUTURA MANDATÓRIA DO FDD (12 SEÇÕES)

O FDD gerado deve conter EXATAMENTE as seguintes 12 seções:

# FDD — Feature Design Document: [Nome da Feature Extraído da Transcrição]

## 1. Contexto e Motivação Técnica
- Contexto de engenharia, limitações da arquitetura atual e motivação técnica para a construção da feature.

---

## 2. Objetivos Técnicos
- Metas de engenharia mensuráveis (latência de processamento, isolamento de falhas, tolerância a indisponibilidade de terceiros, integridade transacional).

---

## 3. Escopo e Exclusões
- **Escopo Técnico de Implementação:** Módulos, workers, endpoints e tabelas a serem implementados.
- **Exclusões Técnicas:** Abordagens, integrações ou componentes explicitamente fora do escopo desta implementação técnica.

---

## 4. Arquitetura e Modelagem de Dados
- **Diagrama Macro dos Componentes:** Diagrama Mermaid de arquitetura técnica da feature.
- **Modelagem de Dados (Schema/Tabelas):** Detalhamento das novas entidades necessárias (campos, tipos, chaves primárias UUID/auto-increment, índices para polling/status).
- **Garantias Transacionais:** Detalhamento do mecanismo de atomicidade (ex.: transações de banco com a entidade principal).

---

## 5. Fluxos Detalhados de Execução
*(Diagramas Mermaid e passo a passo detalhado)*

### 5.1 Fluxo de Produção do Evento (Inserção Transacional)
- Sequência ordenada desde a ação de negócio até o registro persistido.
### 5.2 Fluxo de Consumo e Despacho Assíncrono (Worker)
- Ciclo de vida do worker: polling/leitura em lote, envio HTTP, timeout e atualização de status.
### 5.3 Fluxo de Resiliência, Retentativas e DLQ
- Algoritmo de backoff exponencial, contagem de tentativas e movimentação segura para Dead Letter Queue.

---

## 6. Contratos Públicos (APIs e Endpoints)
*(Mínimo de 4 endpoints HTTP completos detalhados)*

### 6.1 [Método] /caminho-endpoint-1
- **Descrição & Semântica:** ...
- **Autenticação & Autorização:** [Roles / Headers necessários]
- **Headers da Requisição:** ...
- **Request Body (JSON):**
```json
{ ... }
```
- **Response Sucesso (Status HTTP):**
```json
{ ... }
```
- **Response Erro (Status HTTP):**
```json
{ ... }
```

### 6.2 [Método] /caminho-endpoint-2
[Mesma estrutura completa]

### 6.3 [Método] /caminho-endpoint-3
[Mesma estrutura completa]

### 6.4 [Método] /caminho-endpoint-4
[Mesma estrutura completa]

---

## 7. Matriz de Erros Padronizada
Tabela contendo os códigos de erro padronizados do módulo:
| Código de Erro | Status HTTP | Mensagem Padronizada | Causa / Cenário | Ação do Cliente/Sistema |
*(Todos os códigos com prefixo de domínio específico, ex.: `[DOMINIO]_*`)*

---

## 8. Estratégias de Resiliência e Segurança
- **Timeouts e Limites:** Timeouts HTTP, tamanho máximo de payload (KB).
- **Política de Retentativas:** Tabela de intervalos e teto de tentativas.
- **Criptografia e Assinatura:** Algoritmo de assinatura (ex.: HMAC), tratamento de chaves/secrets, headers de segurança e rotação de credenciais.
- **Garantia de Entrega e Idempotência:** Modelo de entrega (*at-least-once*) e identificadores de desduplicação (`X-Event-Id`).

---

## 9. Observabilidade e Telemetria
- **Métricas:** Lista de métricas expostas (contadores de sucesso/falha, latência de despacho, eventos pendentes em fila/DLQ).
- **Logs Estruturados:** Formato dos logs JSON (campos obrigatórios: level, event_id, timestamp, status_code, duration_ms).
- **Tracing & Correlação:** Propagação de IDs para rastreabilidade de ponta a ponta.

---

## 10. Integração com o Sistema Existente
*(Seção obrigatória: mínimo de 4 arquivos reais do código base existentes)*

### 10.1 `caminho/do/arquivo/real-1.ts`
- **Papel Atual:** [O que o arquivo faz hoje]
- **Ponto de Integração:** [Qual método, transação ou classe será estendida]
- **Impacto Técnico:** [Como a nova feature se acopla sem quebrar o comportamento existente]

### 10.2 `caminho/do/arquivo/real-2.ts`
[Mesma estrutura]

### 10.3 `caminho/do/arquivo/real-3.ts`
[Mesma estrutura]

### 10.4 `caminho/do/arquivo/real-4.ts`
[Mesma estrutura]

---

## 11. Riscos de Engenharia e Mitigação
| Risco Técnico | Probabilidade | Impacto | Mitigação de Engenharia | Plano de Contingência |

---

## 12. Critérios de Aceite Técnicos e Testes
- **Critérios de Aceite de Engenharia:** Checklist técnico de validação da feature.
- **Estratégia de Testes Automatizados:** Testes unitários, testes de integração com banco/worker e cenários de fault injection.

---

### INSTRUÇÕES FINAIS DE FORMATAÇÃO E SAÍDA:
1. Gere o documento no idioma Português (Brasil).
2. Escreva o conteúdo em Markdown puro, iniciando diretamente pelo título `# FDD — Feature Design Document: ...`.
3. Nunca oculte nenhuma das 12 seções; caso um ponto não tenha sido discutido na call, mantenha o cabeçalho e registre `> *Não discutido na reunião / A ser definido no refinamento técnico.*`.
4. Não inclua comentários introdutórios antes do Markdown nem notas posteriores. O texto deve ser estritamente o conteúdo a ser gravado em `{{CAMINHO_DESTINO}}`.
```

---

## 4. Guia de Aplicação com Exemplo Prático

Para aplicar este prompt na transcrição e no código do projeto atual:

```bash
# Exemplo de parametrização:
ARQUIVO_TRANSCRICAO: "TRANSCRICAO.md"
CAMINHO_DESTINO: "docs/F001/FDD.md"
ARQUIVO_PRD: "docs/F001/PRD.md"
DIRETORIO_ADRS: "docs/F001/adrs"
CONTEXTO_SISTEMA: "Código Node.js/TypeScript em src/ (ex: src/modules/orders/order.service.ts, src/middlewares/error.middleware.ts, src/shared/errors/app-error.ts, src/config/logger.ts, src/server.ts) e prisma/schema.prisma"
```
