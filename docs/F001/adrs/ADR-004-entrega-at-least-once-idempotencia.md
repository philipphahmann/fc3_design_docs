# ADR-004: Garantia de Entrega At-Least-Once com Desduplicação Idempotente via Cabeçalho X-Event-Id

## Status
Aceito

## Contexto
Em arquiteturas de comunicação assíncrona baseadas em HTTP sobre redes públicas não confiáveis, oscilações de transporte são inevitáveis. Uma situação clássica ocorre quando o receptor do cliente processa a notificação com sucesso, mas a conexão TCP é interrompida ou sofre timeout antes que o pacote de resposta HTTP 200 OK alcance o remetente.

Nesse cenário, sob a ótica do worker do OMS, a requisição é computada como falha de conexão, acionando o mecanismo automático de retentativa. Se o sistema receptor não for capaz de discernir se a nova requisição é a repetição de um evento já processado, ocorrerão efeitos colaterais duplicados indesejados (como atualização dupla em ERPs, emissão duplicada de ordens de separação em WMSs ou inconsistências de auditoria).

A equipe técnica precisa definir formalmente o modelo de garantias de entrega assumido pelo sistema e estabelecer o mecanismo de proteção contra duplicações.

## Decisão
Assumir formalmente o modelo de entrega **No Mínimo Uma Vez (*At-Least-Once Delivery*)**, complementado pela injeção mandatória do cabeçalho de rastreabilidade e idempotência **`X-Event-Id`**:

1. **Geração e Imutabilidade do Identificador do Evento:**
   - No momento em que a alteração de status do pedido comita no banco de dados, o OMS gera um UUID v4 globalmente único para o evento (`eventId`).
   - Esse UUID é gravado de forma imutável na tabela `webhook_outbox` e nunca se altera ao longo de todas as retentativas que aquele evento venha a sofrer.

2. **Propagação do Identificador no Transporte HTTP:**
   - Em todo disparo HTTP realizado pelo worker (seja na tentativa inicial ou em qualquer uma das 5 retentativas do backoff), o OMS envia obrigatoriamente:
     - O cabeçalho HTTP `X-Event-Id: <UUID>`.
     - O campo `"event_id": "<UUID>"` no corpo do payload JSON.

3. **Delegação da Idempotência ao Consumidor:**
   - O contrato de integração estabelece que o sistema receptor do cliente parceiro deve ser idempotente, mantendo controle de eventos recentemente processados indexados por `event_id` (via tabela de banco de dados ou chave com TTL em cache como Redis).
   - Ao receber uma requisição cujo `X-Event-Id` já foi processado anteriormente, o cliente deve retornar imediatamente HTTP 200 OK sem reexecutar suas regras de negócio internas.

4. **Documentação Explícita para Parceiros:**
   - O Product Manager (Marcos) incluirá orientações destacadas e exemplos de código de desduplicação no portal de desenvolvedores voltado aos integradores.

## Alternativas Consideradas

### Garantia Estrita Exactly-Once Distribuída
- **Descrição:** Coordenar a entrega de modo a garantir matematicamente que o evento seja processado exatamente uma única vez, sem possibilidade de reenvio.
- **Motivo do Descarte:** Descartado categoricamente na reunião técnica ([09:25] Diego e Sofia). Em redes abertas sob protocolo HTTP, garantir *exactly-once* exigiria protocolos de consenso transacional bilateral distribuído (como Two-Phase Commit - 2PC), que adicionam acoplamento extremo, lentidão severa e dependência de suporte a transações distribuídas do lado de cada parceiro heterogêneo. Nenhum provedor de webhooks líder de mercado (Stripe, GitHub, Shopify) adota essa abordagem.

### Garantia At-Most-Once (Disparo Único sem Retentativas / Fire-and-Forget)
- **Descrição:** Tentar disparar a notificação HTTP apenas uma única vez; em caso de falha de socket, timeout ou erro no receptor, simplesmente descartar o evento.
- **Motivo do Descarte:** Descartado na reunião técnica ([09:24] Diego). Falhas transitórias de milissegundos causariam perda definitiva de atualizações cruciais de pedidos (como despacho ou faturamento), quebrando os acordos de nível de serviço com os clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) e forçando-os a manter rotinas de polling redundantes.

## Consequências

### Positivas
- **Resiliência Máxima contra Falhas de Rede:** Garante que nenhuma notificação seja descartada prematuramente em situações de oscilação de pacotes.
- **Arquitetura Desacoplada e Escalável:** Não exige controle de estado distribuído entre cliente e servidor, preservando a simplicidade e a alta performance do worker.
- **Alinhamento com Padrões Globais:** Facilita a adesão de desenvolvedores terceiros já acostumados com arquiteturas de eventos da indústria.

### Negativas e Trade-offs
- **Responsabilidade Transferida ao Cliente:** O consumidor externo é obrigado a construir um mecanismo de desduplicação em seu backend receptor para evitar efeitos colaterais de reprocessamento.
- **Possibilidade de Tráfego Redundante:** O parceiro pode receber o mesmo payload mais de uma vez em cenários de timeout de confirmação.

## Integração com o Código Base / Rastreabilidade
- **Modelagem de Dados:** O campo `eventId` (UUID) é definido como índice único no modelo `WebhookOutbox` no arquivo `prisma/schema.prisma`.
- **Implementação no Worker:** O despachante em `src/worker.ts` injetará estritamente `headers['X-Event-Id'] = event.eventId`.
- **Rastreabilidade na Transcrição:** Deliberado e formalizado por Diego, Bruno, Sofia, Marcos e Larissa em `[09:24] Diego`, `[09:25] Bruno, Diego e Sofia`, `[09:26] Marcos e Larissa`, `[09:44] Diego` e `[09:51] Diego e Larissa`.
