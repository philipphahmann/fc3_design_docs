# ADR-006: Reuso dos Padrões Arquiteturais Existentes (Módulos, AppError, Pino, Middleware de Erro e Prisma ORM)

## Status
Aceito

## Contexto
O Order Management System (OMS) foi desenvolvido em Node.js com TypeScript e adota padrões arquiteturais consolidados para organização de módulos, tratamento de erros, observabilidade e persistência de dados.

A estrutura atual conta com:
- **Organização Modular por Domínio:** Diretório `src/modules/<dominio>/` (ex.: `src/modules/orders`, `src/modules/auth`), contendo controller, service, repository, routes e schemas Zod.
- **Tratamento Centralizado de Erros:** Classe base abstrata `AppError` em `src/shared/errors/app-error.ts` e classes especializadas (`NotFoundError`, `ConflictError`, `ValidationError`, `ForbiddenError`), com códigos de erro em caixa alta.
- **Middleware de Erros Unificado:** `src/middlewares/error.middleware.ts`, que captura instâncias de `AppError`, erros de validação Zod e falhas do Prisma ORM, normalizando as respostas HTTP em JSON estruturado com status code adequado.
- **Observabilidade Estruturada:** Biblioteca **Pino** configurada em `src/config/logger.ts`, gerando logs em formato JSON.
- **Segurança e Controle de Acesso (RBAC):** Middleware `authenticate` e função utilitária `requireRole` em `src/middlewares/auth.middleware.ts`.
- **Acesso a Dados:** Prisma ORM com tipagem estrita gerada a partir de `prisma/schema.prisma`.

Ao projetar a nova funcionalidade de webhooks, a equipe precisou decidir entre manter rigorosa conformidade com esses padrões corporativos ou introduzir novas bibliotecas, camadas ad-hoc ou microsserviços separados.

## Decisão
Reutilizar integralmente os padrões arquiteturais e bibliotecas existentes na base de código, integrando a feature de webhooks como um cidadão de primeira classe do ecossistema do OMS:

1. **Estrutura Modular:**
   - Criação do módulo `src/modules/webhooks/`, contendo:
     - `webhook.controller.ts`: Manipulação de requisições HTTP do CRUD de endpoints e consulta de histórico.
     - `webhook.service.ts`: Regras de negócio de cadastro, rotação de secrets, filtros de eventos e auditoria.
     - `webhook.repository.ts`: Abstração de persistência via PrismaClient.
     - `webhook.routes.ts`: Definição das rotas Express com injeção de middlewares.
     - `webhook.schemas.ts`: Schemas de validação de entrada via Zod.
     - `webhook.worker.ts`: Processamento e despacho de lotes da outbox.

2. **Hierarquia de Erros e Prefixo Padronizado:**
   - Criação de classes de erro de domínio estendendo `AppError`, padronizando todos os códigos de erro sob o prefixo **`WEBHOOK_*`**:
     - `WEBHOOK_NOT_FOUND` (404)
     - `WEBHOOK_INVALID_URL` (400)
     - `WEBHOOK_INVALID_EVENTS` (400)
     - `WEBHOOK_SECRET_REQUIRED` (400)
     - `WEBHOOK_SECRET_ROTATION_EXPIRED` (401/403)
     - `WEBHOOK_MAX_RETRIES_EXCEEDED` (500)
     - `WEBHOOK_TIMEOUT` (504)
     - `WEBHOOK_DLQ_NOT_FOUND` (404)
     - `WEBHOOK_FORBIDDEN` (403)
   - Reuso imediato de `src/middlewares/error.middleware.ts` sem alteração no interceptor global.

3. **Telemetria com Pino Logger:**
   - Uso exclusivo da instância de `logger` de `src/config/logger.ts` para registrar logs estruturados em JSON no worker e na API, enriquecidos com os atributos `event_id`, `webhook_id`, `status_code`, `attempt` e `duration_ms`.

4. **Controle de Acesso RBAC:**
   - Aplicação do middleware `authenticate` em todas as rotas de webhooks e `requireRole('ADMIN')` especificamente na rota de replay de DLQ (`POST /admin/webhooks/dead-letter/:id/replay`).

5. **Modelagem de Dados Unificada:**
   - Adição das tabelas de webhooks (`WebhookEndpoint`, `WebhookOutbox`, `WebhookDeadLetter`, `WebhookDelivery`) diretamente no arquivo `prisma/schema.prisma`.

## Alternativas Consideradas

### Criação de Novo Microsserviço Independente com Stack Própria
- **Descrição:** Extrair a funcionalidade de webhooks para um microsserviço separado em outro repositório, com sua própria stack, banco de dados e pipeline de deploy.
- **Motivo do Descarte:** Descartado na reunião técnica ([09:27] Bruno e Larissa). A empresa possui um time enxuto; a divisão prematura em múltiplos repositórios e serviços introduziria custos desnecessários de governança, pipelines de CI/CD duplicados e complexidade de orquestração. A modularização limpa dentro do monólito modular existente atende perfeitamente os requisitos com baixo atrito.

### Adoção de Nova Biblioteca de Logger (Winston) ou Framework de Filas em Memória (BullMQ)
- **Descrição:** Adicionar bibliotecas paralelas como Winston para logs ou BullMQ/Agenda com Redis para gerenciar o agendamento de tarefas do worker.
- **Motivo do Descarte:** Descartado na reunião técnica ([09:29] Bruno e Larissa). O Pino já está integrado e configurado em todo o projeto com alta performance; introduzir novos loggers fragmentaria a telemetria corporativa. A adição de BullMQ exigiria dependência de Redis, violando a decisão de infraestrutura enxuta estabelecida na ADR-001.

### Driver SQL Puro / Knex em Substituição ao Prisma ORM
- **Descrição:** Executar queries SQL manuais via driver MySQL para o processamento de outbox no worker em vez de utilizar o Prisma.
- **Motivo do Descarte:** Descartado na reunião técnica ([09:11] Bruno). O Prisma ORM é o padrão unificado da empresa, fornecendo segurança estática de tipos TypeScript, controle unificado de migrações e capacidade de compartilhar o objeto de transação `tx` (`Prisma.TransactionClient`) entre módulos.

## Consequências

### Positivas
- **Velocidade de Entrega e Baixa Curva de Aprendizado:** Qualquer engenheiro da equipe pode navegar, testar e manter o módulo de webhooks imediatamente, reconhecendo a exata convenção de pastas, injeção de dependências e tratamento de exceções.
- **Zero Atrito de Integração:** O middleware centralizado de erros captura automaticamente todas as novas exceções `WEBHOOK_*`, garantindo respostas JSON uniformes aos consumidores da API.
- **Manutenibilidade e Governança:** O esquema de dados relacional e as migrações continuam centralizados no Prisma ORM.

### Negativas e Trade-offs
- **Acoplamento à Stack Tecnológica:** O módulo assume as restrições e o ciclo de vida do Prisma ORM e do Express, demandando refatoração caso o framework web central seja alterado no futuro.

## Integração com o Código Base / Rastreabilidade
- **Arquivos Reais Integrados e Reutilizados:**
  - `src/modules/orders/order.service.ts`: Inserção da publicação de webhook na transação atômica.
  - `src/middlewares/auth.middleware.ts`: Reuso de `authenticate` e `requireRole('ADMIN')`.
  - `src/middlewares/error.middleware.ts`: Interceptação automática de `AppError` e exceções Zod.
  - `src/shared/errors/app-error.ts`: Classe base herdada pelas exceções do domínio de webhooks.
  - `src/config/logger.ts`: Logger corporativo Pino instanciado e reutilizado sem modificações.
  - `src/server.ts`: Registro do roteador `webhookRoutes` no pipeline do Express.
  - `prisma/schema.prisma`: Extensão dos modelos relacionais.
- **Rastreabilidade na Transcrição:** Deliberado e formalizado por Bruno, Diego, Sofia e Larissa em `[09:27] Bruno`, `[09:28] Bruno e Diego`, `[09:29] Bruno e Diego`, `[09:30] Bruno e Larissa`, `[09:35] Sofia e Larissa` e `[09:48] Larissa`.
