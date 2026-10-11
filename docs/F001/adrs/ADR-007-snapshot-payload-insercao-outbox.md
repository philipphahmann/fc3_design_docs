# ADR-007: Snapshot Imutável do Payload no Momento da Inserção na Outbox

## Status
Aceito

## Contexto
Com a segregação temporal entre o momento da transação do pedido (`OrderService.changeStatus`) e o instante do disparo da notificação HTTP pelo worker assíncrono, surge um dilema clássico de consistência de dados em arquiteturas orientadas a eventos:
1. O registro na tabela `webhook_outbox` deve armazenar apenas o identificador do pedido (`orderId`) e os status envolvidos, delegando ao worker a responsabilidade de consultar o banco de dados e serializar o pedido no momento do envio?
2. Ou o payload JSON deve ser totalmente serializado e congelado como um *snapshot* histórico dentro da própria transação do pedido?

Adicionalmente, era necessário determinar a granularidade dos dados incluídos na notificação, respeitando o teto de 64 KB estipulado pelos requisitos não funcionais de segurança e evitando sobrecarga de I/O de rede.

## Decisão
Adotar a estratégia de **Snapshot Imutável Pré-Renderizado na Inserção da Outbox**, com **payload enxuto e conciso**:

1. **Geração Síncrona do Snapshot:**
   - No momento em que o método `OrderService.changeStatus` executa a transação SQL, a função utilitária de publicação monta a estrutura JSON final do evento e a persiste no campo `payload` (tipo JSON no MySQL / Prisma) da tabela `webhook_outbox`.
   - Se o pedido sofrer alterações rápidas subsequentes antes de o worker processar a outbox (ex.: transições consecutivas `PAID` -> `PROCESSING` -> `SHIPPED`), cada evento despachado refletirá com absoluta precisão o estado histórico exato no instante em que aquela mudança foi comitada.

2. **Estrutura e Campos do Payload:**
   - O payload JSON conterá estritamente os campos essenciais do evento:
     - `event_id`: Identificador UUID v4 imutável.
     - `event_type`: String semântica padronizada (`order.status_changed`).
     - `timestamp`: Timestamp ISO 8601 da ocorrência da transição.
     - `order_id`: UUID do pedido no OMS.
     - `order_number`: Código legível do pedido.
     - `customer_id`: Identificador da conta parceira B2B.
     - `from_status`: Status anterior do pedido.
     - `to_status`: Novo status comitado.
     - `total_cents`: Valor total do pedido em centavos.

3. **Omissão Deliberada dos Itens do Pedido (`OrderItem`):**
   - A lista de produtos/itens (`items`) não será incluída no payload de notificação de webhook. Caso o parceiro integrador necessite da composição detalhada dos itens de linha, ele pode efetuar uma consulta sob demanda no endpoint existente `GET /orders/:id`.

## Alternativas Consideradas

### Persistência de Referências com Montagem Tardia pelo Worker
- **Descrição:** A tabela outbox armazenaria apenas chaves estrangeiras (`orderId`, `fromStatus`, `toStatus`), e o worker faria um `tx.order.findUnique(...)` para ler o pedido no momento do disparo HTTP.
- **Motivo do Descarte:** Descartado na reunião técnica ([09:51] Bruno, Larissa e Diego). Em cenários de processamento em lote ou retentativas de clientes lentos, o pedido poderia já ter avançado para outros status antes de a primeira notificação ser enviada. A leitura tardia geraria o risco de "leitura fantasma" ou anacronismo, enviando dados divergentes do momento histórico da transição.

### Inclusão Completa dos Itens do Pedido (Full Payload com OrderItem[])
- **Descrição:** Incluir a lista detalhada de todos os itens comprados, quantidades, preços unitários e metadados de produto dentro do payload do webhook.
- **Motivo do Descarte:** Descartado na reunião técnica ([09:23] Sofia, [09:43] Diego e Bruno). Pedidos B2B corporativos podem conter centenas de itens, o que inflaria o tamanho do payload para dezenas ou centenas de kilobytes. Isso sobrecarregaria o armazenamento do MySQL, aumentaria a latência de serialização e violaria o teto de 64 KB acordado para prevenir exaustão de banda e ataques de negação de serviço.

## Consequências

### Positivas
- **Fidelidade Histórica Absoluta:** O evento enviado ao cliente representa com precisão cirúrgica o estado do pedido no momento exato em que o status mudou, imune a modificações posteriores no registro da tabela `orders`.
- **Desempenho Otimizado no Worker:** O worker não precisa executar queries adicionais com joins contra a tabela `orders` ou `order_items` para montar o disparo; basta ler o campo `payload` pronto na outbox e assinar o HMAC.
- **Payload Leve e Previsível:** O tamanho do JSON permanece inferior a 2 KB na vasta maioria dos eventos, bem distante do teto de segurança de 64 KB.

### Negativas e Trade-offs
- **Consumo de Armazenamento no Banco:** A persistência de strings JSON na tabela outbox ocupa mais espaço em disco do que o armazenamento de simples referências numéricas/UUIDs.
- **Consulta Complementar pelo Cliente:** Clientes que precisarem dos dados completos dos itens comprados terão que realizar uma chamada HTTP `GET /orders/:id` avulsa.

## Integração com o Código Base / Rastreabilidade
- **Função de Publicação:** O método `changeStatus` em `src/modules/orders/order.service.ts` invocará `publishWebhookEvent(tx, order, from, to)`, que montará o snapshot diretamente a partir dos dados do objeto `order` retornado na transação.
- **Modelagem de Dados:** Coluna `payload` com tipo `Json` no modelo `WebhookOutbox` do arquivo `prisma/schema.prisma`.
- **Rastreabilidade na Transcrição:** Deliberado e formalizado por Bruno, Larissa e Diego em `[09:23] Sofia`, `[09:43] Diego`, `[09:44] Bruno`, `[09:51] Bruno e Larissa` e `[09:52] Larissa e Diego`.
