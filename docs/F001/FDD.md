# FDD — Feature Design Document: Sistema de Webhooks de Notificação de Pedidos

**Versão:** 1.0  
**Data:** 2026-10-11  
**Status:** Pronto para Implementação  
**Autor Principal:** Larissa (Tech Lead)  
**Engenheiros Responsáveis:** Bruno (Time de Pedidos), Diego (Time de Plataforma), Sofia (Segurança)  
**Módulo Alvo:** `src/modules/webhooks/` e `src/worker.ts`  

---

## 1. Contexto e Motivação Técnica

O Order Management System (OMS) opera atualmente sob uma arquitetura de API síncrona desenvolvida em Node.js com TypeScript, persistência relacional no MySQL 8.0 gerenciada pelo Prisma ORM e roteamento via Express. Clientes B2B estratégicos (Atlas Comercial, MaxDistribuição e Nova Cargo) realizam consultas contínuas no endpoint `GET /orders` para identificar alterações nos pedidos, visando sincronizar seus sistemas legados de ERP e WMS.

Essa abordagem de consulta ativa (*polling massivo*) apresenta sérias limitações arquiteturais:
1. **Ineficiência de Recursos:** Mais de 80% das chamadas HTTP recebidas em `GET /orders` retornam conjuntos idênticos de dados sem alteração de estado, onerando o pool de conexões do MySQL, CPU e banda de rede.
2. **Latência de Integração e Risco Comercial:** Mudanças de estado demoram até o próximo ciclo de polling do parceiro para serem identificadas. A Atlas Comercial estabeleceu um SLA improrrogável de entrega de notificações em tempo real (< 10 segundos) até o fim de novembro sob risco iminente de cancelamento de contrato (*churn*).
3. **Inviabilidade do Disparo Síncrono:** O método `OrderService.changeStatus` executa uma transação crítica (`prisma.$transaction`) que atualiza `orders`, `order_status_history` e estoque. Disparar chamadas HTTP síncronas para servidores externos dentro dessa transação bloquearia threads da aplicação e conexões do banco diante de clientes lentos, além de causar rollbacks indevidos de pedidos legítimos caso o receptor do parceiro estivesse temporariamente indisponível.

A feature de Webhooks resolve essas deficiências ao implementar uma arquitetura orientada a eventos assíncrona fundamentada no padrão Transactional Outbox, desacoplando o commit local do despacho externo.

---

## 2. Objetivos Técnicos

| Objetivo Técnico | Métrica / Parâmetro | Meta / Critério de Sucesso |
| :--- | :--- | :--- |
| **Latência de Despacho** | Tempo entre o commit da transação do pedido e o início do disparo HTTP | **p95 < 10 segundos** (com ciclo de polling a cada 2s) |
| **Consistência Transacional** | Discrepância entre alteração de status comitada e registro na outbox | **0% de inconsistência** (atomicidade ACID estrita via `prisma.$transaction`) |
| **Isolamento de Processos** | Degradação de CPU/Memory da API REST causada por lentidão em webhooks | **0% de impacto** (execução em processo autônomo desacoplado `src/worker.ts`) |
| **Tolerância a Indisponibilidade** | Janela de retenção e retentativas diante de quedas dos clientes | **~15 horas** (5 retentativas com backoff progressivo antes da DLQ) |
| **Proteção de Transporte & Payload** | Segurança criptográfica de payloads contra adulteração e replay | **100% dos disparos** assinados com HMAC-SHA256, HTTPS obrigatório e teto de 64 KB |
| **Taxa de Sucesso Final** | Eventos entregues com sucesso aos endpoints ativos | **≥ 99%** de entregas confirmadas |

---

## 3. Escopo e Exclusões

### 3.1 Escopo Técnico de Implementação (In-Scope)
- **Módulo de Configuração de Webhooks (`src/modules/webhooks/`):** CRUD completo para cadastro, edição, listagem e remoção de endpoints de clientes autenticados via JWT.
- **Filtro Seletivo de Eventos:** Suporte à inscrição em subconjuntos específicos de `OrderStatus` (`PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`).
- **Publicação Atômica no Outbox:** Função pura `publishWebhookEvent(tx, order, from, to)` integrada à transação do `OrderService.changeStatus`.
- **Snapshot Imutável:** Serialização dos atributos históricos do pedido no campo `payload` no momento da inserção.
- **Processo Worker Autônomo (`src/worker.ts`):** Loop contínuo com ciclo de polling a cada 2s, lotes de 50 eventos e timeout HTTP de 10s.
- **Assinatura Digital HMAC-SHA256:** Geração de assinatura hexadecimal no cabeçalho `X-Signature` com chave secreta única e inclusão de `X-Timestamp`, `X-Event-Id` e `X-Webhook-Id`.
- **Rotação de Secrets com Convivência:** Endpoint para geração de nova chave mantendo a anterior válida por 24 horas (`previousSecret`).
- **Retentativas com Backoff Exponencial e DLQ:** Escalonamento em 1m, 5m, 30m, 2h e 12h, seguido de migração definitiva para tabela `webhook_dead_letter`.
- **Histórico e Replay Administrativo:** Endpoint `GET /webhooks/:id/deliveries` (últimas 100 execuções) e `POST /admin/webhooks/dead-letter/:id/replay` protegido por `requireRole('ADMIN')`.

### 3.2 Exclusões Técnicas (Out-of-Scope)
- **Notificações de Alerta por E-mail:** Descartado na v1; não haverá envio de e-mails em caso de falhas consecutivas de webhook.
- **Dashboard Visual / Frontend:** O escopo abrange exclusivamente contratos de backend, endpoints REST e worker de linha de comando.
- **Rate Limiting de Saída (*Outbound Throttling*):** O controle de vazão de disparos por cliente não será implementado na v1, permanecendo sob observação em produção.
- **Brokers de Mensageria Externa:** Nenhum componente externo (Kafka, Redis Streams, RabbitMQ) será adicionado à infraestrutura.
- **Garantia de Ordenação Global Multi-Worker:** A ordenação é assegurada estritamente por pedido (`order_id`) enquanto vigorar a topologia de worker único.
- **Webhooks de Entrada (*Inbound Webhooks*):** O sistema não receberá requisições de eventos vindas de parceiros; o fluxo é estritamente de emissão (*outbound*).

---

## 4. Arquitetura e Modelagem de Dados

### 4.1 Diagrama Macro de Componentes

```mermaid
flowchart TD
    subgraph API_Process["Processo 1: API HTTP Express (src/server.ts)"]
        OrderCtrl["OrderController"]
        OrderSvc["OrderService"]
        WebhookCtrl["WebhookController"]
        AdminWebhookCtrl["AdminWebhookController"]
        AuthMid["authenticate & requireRole"]
        ErrorMid["error.middleware.ts (AppError)"]
    end

    subgraph MySQL_DB["Banco de Dados: MySQL 8.0 (Prisma ORM)"]
        TabOrders[("orders")]
        TabHistory[("order_status_history")]
        TabEndpoints[("webhook_endpoints")]
        TabOutbox[("webhook_outbox")]
        TabDLQ[("webhook_dead_letter")]
        TabDeliveries[("webhook_deliveries")]
    end

    subgraph Worker_Process["Processo 2: Background Worker (src/worker.ts)"]
        WorkerLoop["Polling Loop (2s)"]
        Signer["HMAC-SHA256 Signer"]
        Dispatcher["HTTP Dispatcher (Timeout 10s)"]
        RetryEngine["Backoff & DLQ Engine"]
    end

    subgraph External_Partners["Clientes B2B (HTTPS)"]
        PartnerHost["Endpoint Receptor Externo"]
    end

    OrderCtrl --> OrderSvc
    OrderSvc -->|prisma.$transaction ACID| TabOrders
    OrderSvc -->|prisma.$transaction ACID| TabHistory
    OrderSvc -->|publishWebhookEvent| TabOutbox
    OrderSvc -.->|Consulta endpoints ativos| TabEndpoints

    WebhookCtrl --> TabEndpoints
    WebhookCtrl --> TabDeliveries
    AdminWebhookCtrl --> AuthMid
    AdminWebhookCtrl -->|Replay reverte p/ PENDING| TabOutbox
    AdminWebhookCtrl -.->|Lê evento falho| TabDLQ

    WorkerLoop -->|SELECT status = PENDING LIMIT 50| TabOutbox
    WorkerLoop --> Signer
    Signer --> Dispatcher
    Dispatcher -->|POST HTTPS (Headers + JSON)| PartnerHost
    Dispatcher -->|Sucesso 2xx| TabDeliveries
    Dispatcher -->|Atualiza DELIVERED| TabOutbox
    Dispatcher -->|Falha / Timeout| RetryEngine
    RetryEngine -->|Atualiza next_retry_at| TabOutbox
    RetryEngine -->|Exaustão 5 tentativas| TabDLQ
```

### 4.2 Modelagem de Dados (Schema Prisma)

As seguintes definições de modelo serão incorporadas ao arquivo `prisma/schema.prisma`:

```prisma
enum WebhookStatus {
  PENDING
  PROCESSING
  DELIVERED
  FAILED
}

model WebhookEndpoint {
  id               String            @id @default(uuid())
  customerId       String            @map("customer_id")
  url              String            @db.VarChar(500)
  secret           String            @db.VarChar(128)
  previousSecret   String?           @map("previous_secret") @db.VarChar(128)
  secretRotatedAt  DateTime?         @map("secret_rotated_at")
  events           Json              // Array de OrderStatus inscritos, ex: ["PAID", "SHIPPED"]
  isActive         Boolean           @default(true) @map("is_active")
  createdAt        DateTime          @default(now()) @map("created_at")
  updatedAt        DateTime          @updatedAt @map("updated_at")

  outboxEvents     WebhookOutbox[]
  deliveries       WebhookDelivery[]

  @@index([customerId, isActive], name: "idx_webhook_endpoints_customer_active")
  @@map("webhook_endpoints")
}

model WebhookOutbox {
  id             String          @id @default(uuid())
  eventId        String          @unique @map("event_id") // UUID v4 único para idempotência (X-Event-Id)
  webhookId      String          @map("webhook_id")
  orderId        String          @map("order_id")
  eventType      String          @map("event_type") @db.VarChar(100) // "order.status_changed"
  payload        Json            // Snapshot imutável do pedido no momento da transição
  status         WebhookStatus   @default(PENDING)
  attemptCount   Int             @default(0) @map("attempt_count")
  nextRetryAt    DateTime        @default(now()) @map("next_retry_at")
  lastError      String?         @map("last_error") @db.Text
  createdAt      DateTime        @default(now()) @map("created_at")
  updatedAt      DateTime        @updatedAt @map("updated_at")

  endpoint       WebhookEndpoint @relation(fields: [webhookId], references: [id], onDelete: Cascade)

  @@index([status, nextRetryAt, createdAt], name: "idx_webhook_outbox_polling")
  @@index([orderId], name: "idx_webhook_outbox_order")
  @@map("webhook_outbox")
}

model WebhookDeadLetter {
  id             String          @id @default(uuid())
  originalEventId String         @unique @map("original_event_id")
  webhookId      String          @map("webhook_id")
  orderId        String          @map("order_id")
  eventType      String          @map("event_type") @db.VarChar(100)
  payload        Json            // Snapshot preservado do evento
  totalAttempts  Int             @map("total_attempts")
  lastStatusCode Int?            @map("last_status_code")
  errorMessage   String          @map("error_message") @db.Text
  failedAt       DateTime        @default(now()) @map("failed_at")
  replayedAt     DateTime?       @map("replayed_at")
  replayedBy     String?         @map("replayed_by") // ID do usuário ADMIN que executou o replay

  @@index([webhookId], name: "idx_webhook_dlq_webhook")
  @@index([orderId], name: "idx_webhook_dlq_order")
  @@map("webhook_dead_letter")
}

model WebhookDelivery {
  id             String          @id @default(uuid())
  webhookId      String          @map("webhook_id")
  eventId        String          @map("event_id")
  statusCode     Int?            @map("status_code")
  durationMs     Int             @map("duration_ms")
  success        Boolean
  errorMessage   String?         @map("error_message") @db.Text
  createdAt      DateTime        @default(now()) @map("created_at")

  endpoint       WebhookEndpoint @relation(fields: [webhookId], references: [id], onDelete: Cascade)

  @@index([webhookId, createdAt], name: "idx_webhook_deliveries_recent")
  @@map("webhook_deliveries")
}
```

### 4.3 Garantias Transacionais
A gravação da intenção de notificação é executada de forma **síncrona e atômica** dentro da mesma unidade de trabalho do Prisma (`prisma.$transaction`). A função auxiliar `publishWebhookEvent` recebe a instância da transação ativa `tx: Prisma.TransactionClient`:
- Caso qualquer validação de pedido falhe ou o banco lance exceção, toda a transação sofre rollback;
- Nenhuma linha é inserida na outbox se o status do pedido não persistir;
- Nenhuma notificação é enviada se a transação falhar.

---

## 5. Fluxos Detalhados de Execução

### 5.1 Fluxo de Produção do Evento (Inserção Transacional)

```mermaid
sequenceDiagram
    autonumber
    actor Cliente as Operador / API Client
    participant OrderSvc as OrderService.changeStatus()
    participant Tx as Prisma Transaction Client (tx)
    participant OutboxHelper as publishWebhookEvent()
    participant DB as MySQL Database

    Cliente->>OrderSvc: changeStatus(orderId, { toStatus: "SHIPPED" }, userId)
    OrderSvc->>Tx: Abre prisma.$transaction(async (tx) => { ... })
    Tx->>DB: Busca pedido atual e valida transição permitida
    Tx->>DB: Atualiza orders SET status = "SHIPPED"
    Tx->>DB: Cria registro em order_status_history
    Tx->>DB: Atualiza estoque (se aplicável)
    
    OrderSvc->>OutboxHelper: publishWebhookEvent(tx, order, fromStatus, toStatus)
    OutboxHelper->>DB: SELECT id, events FROM webhook_endpoints WHERE customer_id = order.customerId AND is_active = true
    
    loop Para cada endpoint ativo
        alt Novo status consta na lista de eventos do endpoint
            OutboxHelper->>OutboxHelper: Gera eventId = randomUUID()
            OutboxHelper->>OutboxHelper: Renderiza Snapshot JSON (id, order_number, total_cents, etc.)
            OutboxHelper->>DB: INSERT INTO webhook_outbox (eventId, webhookId, payload, status = PENDING)
        else Status ignorado pelo filtro
            Note over OutboxHelper: Descarte silencioso (nenhuma linha gerada)
        end
    end

    Tx-->>OrderSvc: Commit confirmado
    OrderSvc-->>Cliente: Retorna OrderWithRelations atualizado
```

### 5.2 Fluxo de Consumo e Despacho Assíncrono (Worker)

```mermaid
sequenceDiagram
    autonumber
    participant Worker as Background Worker (src/worker.ts)
    participant DB as MySQL Database
    participant Partner as Receptor do Parceiro (HTTPS)

    loop A cada 2 segundos (Polling Loop)
        Worker->>DB: SELECT * FROM webhook_outbox WHERE status = 'PENDING' AND next_retry_at <= NOW() ORDER BY created_at ASC LIMIT 50
        DB-->>Worker: Retorna lote de eventos (0 a 50)
        
        loop Para cada evento do lote
            Worker->>DB: UPDATE webhook_outbox SET status = 'PROCESSING' WHERE id = evento.id
            Worker->>DB: Busca secret ativa do endpoint correspondente
            Worker->>Worker: Computa HMAC-SHA256 sobre rawBody JSON com a secret
            Worker->>Worker: Injeta Headers (X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id)
            
            Worker->>Partner: POST https://partner.com/webhook (Timeout: 10s)
            
            alt Resposta HTTP 2xx recebida em < 10s
                Partner-->>Worker: HTTP 200/204 OK
                Worker->>DB: UPDATE webhook_outbox SET status = 'DELIVERED' WHERE id = evento.id
                Worker->>DB: INSERT INTO webhook_deliveries (success = true, duration_ms, statusCode = 200)
            else Resposta HTTP 4xx / 5xx ou Timeout (>= 10s)
                Partner--xWorker: Erro ou Socket Timeout
                Worker->>Worker: Dispara Tratamento de Retentativa (Seção 5.3)
            end
        end
    end
```

### 5.3 Fluxo de Resiliência, Retentativas e DLQ

```mermaid
sequenceDiagram
    autonumber
    participant Worker as Background Worker
    participant DB as MySQL Database
    participant DLQ as Tabela webhook_dead_letter

    Note over Worker: Falha detectada na entrega do evento
    Worker->>Worker: Incrementa attemptCount = attemptCount + 1
    
    alt attemptCount < 5 (Retentativa válida)
        Worker->>Worker: Calcula intervalo: [1m, 5m, 30m, 2h, 12h][attemptCount - 1]
        Worker->>Worker: nextRetryAt = NOW() + intervalo
        Worker->>DB: UPDATE webhook_outbox SET status = 'PENDING', attempt_count = attemptCount, next_retry_at = nextRetryAt, last_error = errorMsg
        Worker->>DB: INSERT INTO webhook_deliveries (success = false, statusCode, errorMessage)
    else attemptCount >= 5 (Exaustão definitiva de tentativas)
        Worker->>DB: INSERT INTO webhook_dead_letter (originalEventId, webhookId, orderId, payload, totalAttempts = 5, errorMessage)
        Worker->>DB: DELETE FROM webhook_outbox WHERE id = evento.id
        Worker->>DB: INSERT INTO webhook_deliveries (success = false, errorMessage = "MAX_RETRIES_EXCEEDED -> MOVED_TO_DLQ")
        Note over Worker,DLQ: Evento isolado permanentemente na DLQ
    end
```

---

## 6. Contratos Públicos (APIs e Endpoints)

### 6.1 `POST /webhooks` — Cadastro de Endpoint
Permite a um cliente parceiro cadastrar um novo endpoint receptor no OMS.

- **Autenticação:** Obrigatória via cabeçalho `Authorization: Bearer <jwt>`.
- **Regra de Validação:** A URL deve utilizar obrigatoriamente protocolo HTTPS (`https://`). O payload exige pelo menos 1 evento válido.

#### Request Headers
```http
POST /webhooks HTTP/1.1
Host: api.oms.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json
```

#### Request Body
```json
{
  "customerId": "c1f7b029-d5c2-4a57-8bfa-1014a08e1001",
  "url": "https://erp.atlascomercial.com.br/webhooks/orders",
  "events": ["PAID", "PROCESSING", "SHIPPED", "DELIVERED"]
}
```

#### Response Sucesso (`201 Created`)
> *Nota de Segurança:* O campo `secret` é retornado **exclusivamente** na criação. Não será retornado em requisições de consulta posteriores.
```json
{
  "id": "wh_8a91b2c3-4d5e-6f7a-8b9c-0d1e2f3a4b5c",
  "customerId": "c1f7b029-d5c2-4a57-8bfa-1014a08e1001",
  "url": "https://erp.atlascomercial.com.br/webhooks/orders",
  "events": ["PAID", "PROCESSING", "SHIPPED", "DELIVERED"],
  "isActive": true,
  "secret": "whsec_7f9a1c84b2e64d3091f8a5c2d3e4b5a67c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f",
  "createdAt": "2026-10-11T14:30:00.000Z",
  "updatedAt": "2026-10-11T14:30:00.000Z"
}
```

#### Response Erro (`400 Bad Request`)
```json
{
  "error": {
    "code": "WEBHOOK_INVALID_URL",
    "message": "Webhook URL must strictly use HTTPS protocol",
    "details": {
      "providedUrl": "http://erp.atlascomercial.com.br/webhooks/orders"
    }
  }
}
```

---

### 6.2 `GET /webhooks` — Listagem de Endpoints
Retorna a listagem dos webhooks cadastrados para a conta do cliente.

- **Autenticação:** Obrigatória via cabeçalho `Authorization: Bearer <jwt>`.
- **Query Parameters:** `customerId` (UUID, obrigatório).

#### Request Headers
```http
GET /webhooks?customerId=c1f7b029-d5c2-4a57-8bfa-1014a08e1001 HTTP/1.1
Host: api.oms.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

#### Response Sucesso (`200 OK`)
```json
{
  "items": [
    {
      "id": "wh_8a91b2c3-4d5e-6f7a-8b9c-0d1e2f3a4b5c",
      "customerId": "c1f7b029-d5c2-4a57-8bfa-1014a08e1001",
      "url": "https://erp.atlascomercial.com.br/webhooks/orders",
      "events": ["PAID", "PROCESSING", "SHIPPED", "DELIVERED"],
      "isActive": true,
      "createdAt": "2026-10-11T14:30:00.000Z",
      "updatedAt": "2026-10-11T14:30:00.000Z"
    }
  ],
  "total": 1
}
```

---

### 6.3 `PATCH /webhooks/:id` — Atualização Parcial
Permite alterar a URL, a lista de eventos monitorados ou o status de ativação.

- **Autenticação:** Obrigatória via cabeçalho `Authorization: Bearer <jwt>`.

#### Request Headers
```http
PATCH /webhooks/wh_8a91b2c3-4d5e-6f7a-8b9c-0d1e2f3a4b5c HTTP/1.1
Host: api.oms.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json
```

#### Request Body
```json
{
  "url": "https://erp.atlascomercial.com.br/v2/webhooks/orders",
  "events": ["SHIPPED", "DELIVERED"],
  "isActive": true
}
```

#### Response Sucesso (`200 OK`)
```json
{
  "id": "wh_8a91b2c3-4d5e-6f7a-8b9c-0d1e2f3a4b5c",
  "customerId": "c1f7b029-d5c2-4a57-8bfa-1014a08e1001",
  "url": "https://erp.atlascomercial.com.br/v2/webhooks/orders",
  "events": ["SHIPPED", "DELIVERED"],
  "isActive": true,
  "createdAt": "2026-10-11T14:30:00.000Z",
  "updatedAt": "2026-10-11T16:45:00.000Z"
}
```

---

### 6.4 `DELETE /webhooks/:id` — Remoção / Desativação
Desativa ou remove o registro de webhook, interrompendo enfileiramentos futuros.

- **Autenticação:** Obrigatória via cabeçalho `Authorization: Bearer <jwt>`.

#### Request Headers
```http
DELETE /webhooks/wh_8a91b2c3-4d5e-6f7a-8b9c-0d1e2f3a4b5c HTTP/1.1
Host: api.oms.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

#### Response Sucesso (`204 No Content`)
Corpo vazio.

---

### 6.5 `POST /webhooks/:id/rotate-secret` — Rotação de Segredo
Gera uma nova chave secreta mantendo a chave anterior válida por 24 horas (*grace period*).

- **Autenticação:** Obrigatória via cabeçalho `Authorization: Bearer <jwt>`.

#### Request Headers
```http
POST /webhooks/wh_8a91b2c3-4d5e-6f7a-8b9c-0d1e2f3a4b5c/rotate-secret HTTP/1.1
Host: api.oms.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

#### Response Sucesso (`200 OK`)
```json
{
  "id": "wh_8a91b2c3-4d5e-6f7a-8b9c-0d1e2f3a4b5c",
  "newSecret": "whsec_99a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4d3e2f1a0b9c8d7e6f5a4b3c2d1e0f9a8",
  "gracePeriodHours": 24,
  "previousSecretExpiresAt": "2026-10-12T17:00:00.000Z",
  "message": "Both previous and new secrets will be accepted for verification during the 24-hour grace period."
}
```

---

### 6.6 `GET /webhooks/:id/deliveries` — Histórico de Entregas
Consulta os últimos 100 disparos executados contra aquele endpoint para auditoria e conferência.

- **Autenticação:** Obrigatória via cabeçalho `Authorization: Bearer <jwt>`.

#### Request Headers
```http
GET /webhooks/wh_8a91b2c3-4d5e-6f7a-8b9c-0d1e2f3a4b5c/deliveries HTTP/1.1
Host: api.oms.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

#### Response Sucesso (`200 OK`)
```json
{
  "items": [
    {
      "id": "del_11a22b33-44c5-66d7-88e9-00f11a22b33c",
      "eventId": "evt_9f8e7d6c-5b4a-3f2e-1d0c-9b8a7f6e5d4c",
      "statusCode": 200,
      "durationMs": 142,
      "success": true,
      "errorMessage": null,
      "createdAt": "2026-10-11T16:46:12.000Z"
    },
    {
      "id": "del_00f11a22-33b4-55c6-77d8-99e00f11a22b",
      "eventId": "evt_8e7d6c5b-4a3f-2e1d-0c9b-8a7f6e5d4c3b",
      "statusCode": 504,
      "durationMs": 10002,
      "success": false,
      "errorMessage": "HTTP request timed out after 10000ms",
      "createdAt": "2026-10-11T15:20:00.000Z"
    }
  ],
  "total": 2
}
```

---

### 6.7 `POST /admin/webhooks/dead-letter/:id/replay` — Replay Manual de DLQ
Permite a um operador com permissão de administrador reprocessar uma mensagem morta.

- **Autenticação & Autorização:** Exige obrigatoriamente perfil `ADMIN` (`requireRole('ADMIN')`).

#### Request Headers
```http
POST /admin/webhooks/dead-letter/dlq_5a6b7c8d-9e0f-1a2b-3c4d-5e6f7a8b9c0d/replay HTTP/1.1
Host: api.oms.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9... (Token com role: ADMIN)
```

#### Response Sucesso (`200 OK`)
```json
{
  "success": true,
  "deadLetterId": "dlq_5a6b7c8d-9e0f-1a2b-3c4d-5e6f7a8b9c0d",
  "requeuedOutboxId": "whout_77a88b99-00c1-22d3-44e5-66f77a88b99c",
  "status": "PENDING",
  "message": "Event successfully re-enqueued into outbox with attempt count reset to 0."
}
```

#### Response Erro de Permissão (`403 Forbidden`)
```json
{
  "error": {
    "code": "WEBHOOK_FORBIDDEN",
    "message": "Only users with ADMIN role are authorized to trigger dead letter replay actions"
  }
}
```

---

### 6.8 Contrato de Notificação de Saída (Disparo HTTP do Worker para o Cliente)
Especificação da requisição HTTP POST transmitida pelo worker em direção ao servidor parceiro.

#### Request HTTP Disparado
```http
POST /webhooks/orders HTTP/1.1
Host: erp.atlascomercial.com.br
User-Agent: OMS-Webhook-Worker/1.0
Content-Type: application/json
X-Event-Id: 9f8e7d6c-5b4a-3f2e-1d0c-9b8a7f6e5d4c
X-Webhook-Id: wh_8a91b2c3-4d5e-6f7a-8b9c-0d1e2f3a4b5c
X-Timestamp: 2026-10-11T16:46:10.512Z
X-Signature: c8b92b6727dc02d5b6a74b059737e54f9a0c793132ebdf02b851b3d6a9a7b931
Content-Length: 328

{
  "event_id": "9f8e7d6c-5b4a-3f2e-1d0c-9b8a7f6e5d4c",
  "event_type": "order.status_changed",
  "timestamp": "2026-10-11T16:46:10.000Z",
  "order_id": "ord_10293847-5647-3829-1029-384756102938",
  "order_number": "ORD-2026-08492",
  "customer_id": "c1f7b029-d5c2-4a57-8bfa-1014a08e1001",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "total_cents": 154900
}
```

---

## 7. Matriz de Erros Padronizada

Todas as exceções do módulo adotam obrigatoriamente o prefixo de domínio **`WEBHOOK_*`**, estendendo a classe base corporativa `AppError`:

| Código de Erro | Status HTTP | Mensagem Padronizada | Causa / Cenário | Ação do Cliente / Sistema |
| :--- | :--- | :--- | :--- | :--- |
| `WEBHOOK_NOT_FOUND` | 404 Not Found | `Webhook endpoint not found` | ID informado não existe ou pertence a outro cliente | Verificar ID informado e permissões |
| `WEBHOOK_INVALID_URL` | 400 Bad Request | `Webhook URL must strictly use HTTPS protocol` | Cadastro ou atualização com URL `http://` | Alterar URL para `https://` com TLS válido |
| `WEBHOOK_INVALID_EVENTS` | 400 Bad Request | `At least one valid order status event must be selected` | Lista de eventos vazia ou contendo status inexistente | Fornecer array de status válidos (`OrderStatus`) |
| `WEBHOOK_SECRET_REQUIRED` | 400 Bad Request | `Secret is required for signature verification` | Tentativa de assinatura sem chave carregada | Verificar integridade cadastral do endpoint |
| `WEBHOOK_SECRET_ROTATION_EXPIRED` | 401 Unauthorized | `Previous secret has expired after 24h grace period` | Cliente tentando validar com segredo anterior após 24h | Atualizar receptor com o segredo ativo |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 413 Payload Too Large | `Event payload exceeds maximum limit of 64 KB` | Payload gerado com tamanho superior a 65.536 bytes | Omitir campos excessivos e consultar via API |
| `WEBHOOK_TIMEOUT` | 504 Gateway Timeout | `Webhook endpoint failed to respond within 10000ms` | Servidor do parceiro demorou ≥ 10s para responder | Worker agenda retentativa automática via backoff |
| `WEBHOOK_MAX_RETRIES_EXCEEDED` | 500 Internal Error | `Delivery failed after 5 attempts; transferred to DLQ` | Exaustão das 5 tentativas de retentativa | Mensagem movida para `webhook_dead_letter` |
| `WEBHOOK_DLQ_NOT_FOUND` | 404 Not Found | `Dead letter record not found for the specified ID` | ID de mensagem morta inexistente para replay | Verificar ID na tabela de DLQ |
| `WEBHOOK_FORBIDDEN` | 403 Forbidden | `Insufficient permissions; ADMIN role required` | Usuário sem perfil `ADMIN` acionando rota de replay | Requisitar privilégio de administrador |

---

## 8. Estratégias de Resiliência e Segurança

### 8.1 Timeouts e Limite de Payload
- **Timeout Rígido de Despacho:** Toda requisição HTTP executada pelo worker possui limite estrito de **10 segundos** (`timeout: 10000`). Conexões lentas são canceladas para evitar esgotamento de descritores de arquivo e travamento de threads.
- **Teto Máximo de Payload (64 KB):** O tamanho máximo permitido para serialização de um evento é **64 KB**. O snapshot omite deliberadamente `OrderItem[]` para garantir payloads enxutos (~1 a 2 KB).

### 8.2 Política de Retentativas e Backoff Exponencial
O agendamento de retentativas segue progressão matemática determinística:

| Tentativa | Intervalo Após Falha Anterior | Janela Cumulativa Decorrida | Destino em Caso de Falha |
| :---: | :---: | :---: | :--- |
| **1ª (Inicial)** | Imediata no próximo polling (≤ 2s) | 0 a 2s | Agenda Retentativa 1 (1 min) |
| **Retentativa 1** | **1 minuto** | ~1 minuto | Agenda Retentativa 2 (5 min) |
| **Retentativa 2** | **5 minutos** | ~6 minutos | Agenda Retentativa 3 (30 min) |
| **Retentativa 3** | **30 minutos** | ~36 minutos | Agenda Retentativa 4 (2 horas) |
| **Retentativa 4** | **2 horas** | ~2 horas e 36 minutos | Agenda Retentativa 5 (12 horas) |
| **Retentativa 5** | **12 horas** | **~14 horas e 36 minutos** | **Migração para DLQ (`webhook_dead_letter`)** |

### 8.3 Criptografia, HMAC-SHA256 e Rotação
- **Assinatura Digital:** Computada exclusivamente com o módulo nativo `crypto` do Node.js:
  ```typescript
  const signature = crypto
    .createHmac('sha256', secret)
    .update(rawBodyString)
    .digest('hex');
  ```
- **Proteção contra Replay Attacks:** Injeção do cabeçalho `X-Timestamp`. Receptores de clientes devem verificar se o timestamp do request difere em mais de 5 minutos do relógio local.
- **Rotação Suave de 24 Horas:** Chaves antigas permanecem salvas em `previousSecret` e são aceitas até `secretRotatedAt + 24h`.

### 8.4 Idempotência (At-Least-Once Delivery)
- O sistema opera sob garantia *at-least-once*.
- Todo envio propaga o cabeçalho `X-Event-Id` preenchido com UUID v4 fixado na inserção da outbox. Clientes devem persistir o `X-Event-Id` e responder imediatamente 200 OK caso recebam requisição com identificador já processado.

---

## 9. Observabilidade e Telemetria

### 9.1 Métricas Operacionais
O módulo expõe métricas para monitoramento corporativo:
- `webhook_events_enqueued_total`: Contador de eventos gravados na outbox.
- `webhook_delivery_attempts_total{status="success|failure"}`: Contador de tentativas de despacho HTTP.
- `webhook_delivery_duration_seconds{quantile="p50|p95|p99"}`: Histograma de latência de resposta dos receptores externos.
- `webhook_outbox_pending_gauge`: Medidor de eventos represados na fila aguardando processamento.
- `webhook_dead_letter_total`: Contador cumulativo de eventos transferidos para a DLQ.

### 9.2 Logs Estruturados em JSON (Pino Logger)
Todos os logs utilizam a instância compartilhada `src/config/logger.ts`, gerando saídas estruturadas em JSON:

```json
{
  "level": "info",
  "time": 1791737170512,
  "pid": 4821,
  "hostname": "oms-worker-01",
  "context": "WebhookWorker",
  "event_id": "9f8e7d6c-5b4a-3f2e-1d0c-9b8a7f6e5d4c",
  "webhook_id": "wh_8a91b2c3-4d5e-6f7a-8b9c-0d1e2f3a4b5c",
  "attempt": 1,
  "status_code": 200,
  "duration_ms": 142,
  "msg": "Webhook delivery succeeded"
}
```

Em cenários de falha ou timeout:
```json
{
  "level": "warn",
  "time": 1791737180514,
  "pid": 4821,
  "hostname": "oms-worker-01",
  "context": "WebhookWorker",
  "event_id": "8e7d6c5b-4a3f-2e1d-0c9b-8a7f6e5d4c3b",
  "webhook_id": "wh_8a91b2c3-4d5e-6f7a-8b9c-0d1e2f3a4b5c",
  "attempt": 2,
  "status_code": 504,
  "duration_ms": 10002,
  "error": "HTTP request timed out after 10000ms",
  "next_retry_at": "2026-10-11T16:51:10.000Z",
  "msg": "Webhook delivery failed, retry scheduled"
}
```

---

## 10. Integração com o Sistema Existente

Esta seção detalha o acoplamento da nova feature com **arquivos reais existentes na codebase**:

### 10.1 `src/modules/orders/order.service.ts`
- **Papel Atual:** Orquestra a criação, listagem e alteração de pedidos. O método `changeStatus(id, input, userId)` executa `this.prisma.$transaction(async (tx) => { ... })` atualizando `orders`, inserindo auditoria em `orderStatusHistory` e manipulando estoque (`debitStock`/`replenishStock`).
- **Ponto de Integração:** Dentro da closure transacional `this.prisma.$transaction` (logo após a linha 165), invocar a função auxiliar `publishWebhookEvent(tx, order, from, to)`.
- **Impacto Técnico:** A inserção na tabela `webhook_outbox` compartilha o mesmo client de transação `tx` (`Prisma.TransactionClient`). Se a gravação na outbox falhar, a transação inteira sofre rollback, mantendo 100% de consistência ACID sem risco de descompasso entre pedidos e eventos.

### 10.2 `src/middlewares/auth.middleware.ts`
- **Papel Atual:** Implementa autenticação JWT (`authenticate`) injetando o usuário autenticado em `req.user` (`AuthUser = { id, email, role: 'ADMIN' | 'OPERATOR' }`) e autorização baseada em papéis via `requireRole(...roles)`.
- **Ponto de Integração:** 
  1. O middleware `authenticate` será aplicado em todas as rotas do módulo de webhooks (`src/modules/webhooks/webhook.routes.ts`).
  2. A função `requireRole('ADMIN')` protegerá a rota de reprocessamento manual de mensagens mortas (`POST /admin/webhooks/dead-letter/:id/replay`).
- **Impacto Técnico:** Reuso direto sem modificação nos middlewares de segurança, garantindo conformidade com a política de governança RBAC da aplicação.

### 10.3 `src/middlewares/error.middleware.ts`
- **Papel Atual:** Middleware centralizado de tratamento de erros Express (`errorHandler`). Captura instâncias de `AppError` formatando respostas `{ error: { code, message, details } }`, além de tratar `ZodError` e exceções do Prisma.
- **Ponto de Integração:** O novo módulo definirá exceções que estendem `AppError` com códigos prefixados em `WEBHOOK_*`.
- **Impacto Técnico:** O middleware existente captura automaticamente todas as exceções de webhooks sem necessidade de adicionar qualquer linha de código no interceptor global.

### 10.4 `src/shared/errors/app-error.ts` (e `src/shared/errors/index.ts`)
- **Papel Atual:** Fornece a classe base abstrata `AppError` e especializações padrão (`NotFoundError`, `ConflictError`, `ValidationError`, `ForbiddenError`, `UnauthorizedError`).
- **Ponto de Integração:** O arquivo `src/modules/webhooks/webhook.errors.ts` importará `AppError` e exportará as classes especializadas do domínio (`WebhookNotFoundError`, `WebhookInvalidUrlError`, `WebhookSecretRotationExpiredError`, `WebhookTimeoutError`, `WebhookDlqNotFoundError`).
- **Impacto Técnico:** Preserva a arquitetura limpa e a uniformidade de tratamento de erros em toda a aplicação.

### 10.5 `src/config/logger.ts`
- **Papel Atual:** Exporta a instância corporativa do logger Pino configurada com serialização em JSON.
- **Ponto de Integração:** O processador do worker (`src/worker.ts` e `src/modules/webhooks/webhook.worker.ts`) consumirá a instância `logger`, enriquecendo os metadados com chaves `event_id`, `webhook_id`, `attempt`, `status_code` e `duration_ms`.
- **Impacto Técnico:** Telemetria estruturada enviada imediatamente aos agregadores de logs sem criar bibliotecas redundantes.

### 10.6 `src/server.ts`
- **Papel Atual:** Ponto de entrada HTTP do OMS, configurando middlewares globais e montando roteadores sob `/orders`, `/auth`, etc.
- **Ponto de Integração:** Importação e registro dos roteadores de webhooks:
  ```typescript
  app.use('/webhooks', webhookRoutes);
  app.use('/admin/webhooks', adminWebhookRoutes);
  ```
- **Impacto Técnico:** Expõe os novos endpoints REST de autoatendimento e administração sem alterar as rotas existentes.

### 10.7 `prisma/schema.prisma`
- **Papel Atual:** Define os modelos relacionais (`User`, `Customer`, `Order`, `OrderItem`, `Product`, `OrderStatusHistory`) para MySQL.
- **Ponto de Integração:** Inclusão dos modelos `WebhookEndpoint`, `WebhookOutbox`, `WebhookDeadLetter` e `WebhookDelivery` e do enum `WebhookStatus`.
- **Impacto Técnico:** Gera tipos estáticos rigorosos do Prisma Client consumidos tanto pela API quanto pelo worker assíncrono.

---

## 11. Riscos de Engenharia e Mitigação

| Risco Técnico | Probabilidade | Impacto | Mitigação de Engenharia | Plano de Contingência |
| :--- | :---: | :---: | :--- | :--- |
| **Contenção e Lock de Tabela no MySQL** | Baixa | Médio | Índice composto cobrindo `(status, next_retry_at, created_at)`. Polling executando `LIMIT 50`. | Implementar rotina de expurgo/arquivamento de eventos com mais de 30 dias para banco histórico. |
| **Starvation por Clientes Lentos / Zumbis** | Média | Alto | Timeout estrito de 10 segundos configurado no client HTTP com abort controller. | Descarte e migração rápida para retentativa sem bloquear eventos subsequentes. |
| **Vazamento Acidental de Segredos** | Média | Alto | Chave nunca retornada em requisições de consulta (`GET /webhooks`). Segredos exclusivos por endpoint. Rotação com grace period de 24h. | Inativação imediata do endpoint via `DELETE /webhooks/:id` ou rotação forçada pela engenharia. |
| **Loop de Falhas e Esgotamento de Disco** | Baixa | Médio | Teto de 5 retentativas com migração irreversível para tabela isolada de DLQ. | Monitoramento de métrica `webhook_dead_letter_total` com alertas para a equipe de suporte. |
| **Falha de Crash no Processo do Worker** | Baixa | Alto | Handlers para `uncaughtException` e `unhandledRejection` com shutdown gracioso (`SIGTERM`/`SIGINT`). | Orquestrador de processos (Docker/Kubernetes/PM2) reinicia o processo automaticamente; eventos permanecem seguros no MySQL. |

---

## 12. Critérios de Aceite Técnicos e Testes

### 12.1 Checklist de Critérios de Aceite Técnicos
- [ ] O modelo relacional no Prisma comita 100% das migrações sem quebrar as tabelas existentes (`orders`, `customers`).
- [ ] O método `OrderService.changeStatus` grava o evento na outbox de forma atômica; qualquer falha na outbox cancela a alteração de status do pedido.
- [ ] Pedidos cujo status não conste nos eventos inscritos do cliente não geram registros na tabela outbox.
- [ ] O processo `src/worker.ts` inicia de forma autônoma via `npm run worker` utilizando instância dedicada de `PrismaClient`.
- [ ] O worker executa polling a cada 2 segundos e dispara requisições HTTP POST aplicando timeout rígido de 10 segundos.
- [ ] As requisições disparadas pelo worker contêm os cabeçalhos obrigatórios `Content-Type`, `X-Event-Id`, `X-Webhook-Id`, `X-Timestamp` e `X-Signature`.
- [ ] A assinatura HMAC-SHA256 é calculada com exatidão sobre o corpo cru do request utilizando a chave secreta exclusiva do endpoint.
- [ ] Falhas temporárias (timeout ou HTTP != 2xx) escalonam retentativas nos intervalos de 1m, 5m, 30m, 2h e 12h.
- [ ] Eventos que ultrapassam a 5ª retentativa são transferidos para `webhook_dead_letter` e excluídos da fila ativa.
- [ ] O endpoint `POST /admin/webhooks/dead-letter/:id/replay` é protegido pelo middleware `requireRole('ADMIN')` e reinsere eventos como `PENDING`.
- [ ] A rotação de chave secreta viabiliza período de convivência de 24 horas mantendo a chave antiga em `previousSecret`.
- [ ] A latência p95 entre o commit do pedido e o envio do pacote HTTP é inferior a 10 segundos.

### 12.2 Estratégia de Testes Automatizados
- **Testes Unitários:**
  - Validação de schemas Zod (rejeição de URLs `http://`, arrays de eventos vazios ou tipos inválidos).
  - Cálculo determinístico de hash HMAC-SHA256 contra vetores de teste conhecidos.
  - Função de cálculo de backoff exponencial garantindo os intervalos corretos por tentativa.
  - Serialização de payload comprovando que o snapshot omite `OrderItem[]` e respeita o teto de 64 KB.
- **Testes de Integração:**
  - Execução transacional de `OrderService.changeStatus` em banco de testes MySQL, validando commit simultâneo em `orders`, `order_status_history` e `webhook_outbox`.
  - Simulação de erro na inserção da outbox, garantindo rollback completo da transação do pedido.
  - Ciclo de leitura do worker em lote consumindo eventos `PENDING` e atualizando para `DELIVERED` contra um mock server HTTP local.
- **Testes de Resiliência e Falhas:**
  - Mock server simulando latência de 12 segundos, comprovando timeout de 10s do worker e agendamento da 1ª retentativa para +1 minuto.
  - Simulação de 5 falhas consecutivas, comprovando transferência íntegra do evento para `webhook_dead_letter`.
  - Execução de replay de DLQ com usuário `ADMIN` (sucesso 200) e usuário `OPERATOR` (rejeição 403 Forbidden).
