# ADR-001: Padrão Transactional Outbox Persistido no MySQL Existente

## Status
Aceito

## Contexto
O Order Management System (OMS) precisa emitir notificações assíncronas (webhooks) para sistemas corporativos parceiros (Atlas Comercial, MaxDistribuição e Nova Cargo) sempre que o status de um pedido for alterado. 

Atualmente, o método `changeStatus` da classe `OrderService` executa uma transação crítica no banco de dados relacional MySQL via Prisma ORM (`prisma.$transaction`). Essa transação atualiza a tabela `orders`, registra auditoria em `order_status_history` e ajusta os estoques de produtos. 

A introdução do disparo de webhooks impõe um desafio de consistência e disponibilidade: se a chamada HTTP for executada sincronamente dentro da transação de pedidos, qualquer lentidão, timeout de rede ou indisponibilidade do cliente receptor travará a transação SQL, retendo conexões do pool e degradando o throughput do sistema principal. Por outro lado, disparar a requisição de forma assíncrona após o commit sem garantia de persistência geraria perda irrecuperável de eventos em caso de falha de processo ou reinicialização da aplicação. Além disso, a equipe de engenharia é enxuta e busca evitar a introdução de novos componentes complexos de infraestrutura (como clusters de mensageria dedicada).

## Decisão
Adotar o padrão arquitetural **Transactional Outbox** utilizando o próprio banco de dados relacional **MySQL** já em produção.

A gravação da intenção de notificação é acoplada diretamente à transação do pedido: dentro da mesma chamada `prisma.$transaction` gerenciada pelo `OrderService.changeStatus`, será realizada a inserção de um registro na tabela `webhook_outbox`. Caso a transação do pedido falhe ou sofra rollback, o evento de webhook é cancelado automaticamente pela atomicidade ACID do banco relacional. Caso a transação comite com sucesso, a persistência do evento estará matematicamente garantida. Um worker desacoplado será responsável por ler os registros pendentes e realizar os disparos HTTP de forma autônoma.

Para mitigar impactos de desempenho no MySQL decorrentes do crescimento volumétrico da tabela, serão criados índices compostos cobrindo os campos de status (`PENDING`, `PROCESSING`) e data (`created_at`), limitando as consultas do worker a lotes curtos (`LIMIT 50`).

## Alternativas Consideradas

### Disparo HTTP Síncrono no OrderService
- **Descrição:** Executar o disparo HTTP diretamente dentro do fluxo do método `changeStatus`, aguardando a resposta do cliente para prosseguir ou comitar a transação.
- **Motivo do Descarte:** Descartado categoricamente na reunião técnica ([09:04] Bruno, [09:06] Diego). A transação de mudança de status já é pesada. Qualquer oscilação de rede ou endpoint lento do cliente travaria conexões do banco de dados e threads da API, degradando o tempo de resposta do OMS. Além disso, falhas de conectividade no cliente forçariam o rollback indevido da atualização do pedido, criando um acoplamento operacional inaceitável entre a disponibilidade do parceiro externo e o fluxo de pedidos interno.

### Mensageria Externa Dedicada (Redis Streams / Apache Kafka / RabbitMQ)
- **Descrição:** Publicar os eventos de mudança de status em um broker de mensageria externo dedicado, consumindo as mensagens através de workers assíncronos.
- **Motivo do Descarte:** Descartado na reunião técnica ([09:07] Larissa e Diego). A equipe de engenharia e operações é enxuta, e a adição de um cluster de mensageria externa representaria *overengineering* para o volume atual de tráfego, elevando o custo de infraestrutura, monitoramento e manutenção de dependências operacionais. O MySQL existente suporta a carga com folga através de consultas indexadas e lotes reduzidos.

## Consequências

### Positivas
- **Garantia de Consistência Atômica (ACID):** Não há risco de disparar webhook para uma mudança de status que sofreu rollback, nem de confirmar o status sem que o evento tenha sido registrado na outbox.
- **Simplicidade Operacional e Zero Overhead de Nova Infraestrutura:** Aproveita a infraestrutura MySQL 8.0 e o Prisma ORM já existentes, sem demandar novos servidores, brokers ou pipelines de observabilidade.
- **Desacoplamento Completo do Ciclo de Vida da API:** Falhas ou lentidões externas dos parceiros não afetam a latência nem a estabilidade dos endpoints síncronos de alteração de pedidos do OMS.

### Negativas e Trade-offs
- **Latência Não Instantânea:** Como o worker lê o banco de dados via polling periódico em intervalos de 2 segundos, a entrega não é reativa em milissegundos, impondo uma latência base mínima de até 2 segundos (o que permanece bem abaixo do limite de 10 segundos acordado com os clientes).
- **Consumo Contínuo de Conexões no MySQL:** O ciclo de polling do worker gera consultas constantes contra o banco relacional, exigindo índices rigorosos e governança de expurgo/arquivamento futuro para registros já entregues com mais de 30 dias.

## Integração com o Código Base / Rastreabilidade
- **Arquivo Modificado na Solução:** `src/modules/orders/order.service.ts`, especificamente o método `changeStatus` (linhas 126–165), estendendo a closure de `this.prisma.$transaction(async (tx) => { ... })` para invocar a persistência na tabela outbox.
- **Modelagem de Dados:** Adição do modelo `WebhookOutbox` no arquivo `prisma/schema.prisma` com campos `id` (UUID), `eventId`, `webhookId`, `status`, `payload` (JSON), `attemptCount` e timestamps indexados.
- **Rastreabilidade na Transcrição:** Deliberado e formalizado por Larissa, Bruno e Diego em `[09:04] Bruno`, `[09:06] Diego`, `[09:07] Larissa`, `[09:08] Diego` e `[09:40] Bruno`.
