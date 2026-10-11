# ADR-002: Política de Retentativas com Backoff Exponencial Escalonado e Tabela de Dead Letter Queue (DLQ)

## Status
Aceito

## Contexto
Endpoints receptores de clientes parceiros B2B estão sujeitos a indisponibilidades transitórias (quedas temporárias de rede, picos de carga locais, lentidão no banco interno e janelas de manutenção planejada de até 2 horas). Caso a primeira tentativa de despacho HTTP falhe (por timeout de conexão ≥ 10s ou resposta com status HTTP 4xx/5xx), o sistema precisa implementar um mecanismo resiliente de reenvio sem sobrecarregar o cliente receptor nem bloquear a fila de processamento dos demais clientes.

Por outro lado, caso um cliente parceiro tenha desativado definitivamente o seu receptor ou alterado sua rota sem atualização prévia no OMS, o sistema não pode retentar o envio eternamente, sob pena de consumir recursos de banco e rede indefinidamente com eventos órfãos.

## Decisão
Implementar uma política determinística de **5 tentativas com backoff exponencial escalonado**, seguida do isolamento de falhas persistentes em uma **tabela dedicada de Dead Letter Queue (`webhook_dead_letter`)**:

1. **Escalonamento de Intervalos:**
   - Tentativa 1 (imediata): disparada no próximo ciclo de polling do worker após a criação do evento.
   - Tentativa 2 (1ª retentativa): agendada para **1 minuto** após a 1ª falha.
   - Tentativa 3 (2ª retentativa): agendada para **5 minutos** após a 2ª falha.
   - Tentativa 4 (3ª retentativa): agendada para **30 minutos** após a 3ª falha.
   - Tentativa 5 (4ª retentativa): agendada para **2 horas** após a 4ª falha.
   - Tentativa 6 (5ª retentativa final): agendada para **12 horas** após a 5ª falha.
   - **Janela de Cobertura Total:** Aproximadamente **15 horas** contínuas entre a primeira falha e a desistência definitiva.

2. **Isolamento em Tabela Separada de DLQ:**
   - Esgotadas as 5 retentativas sem sucesso, o evento é removido ou arquivado na tabela `webhook_outbox` e seus dados são persistidos na tabela `webhook_dead_letter`.
   - A tabela armazena o identificador original, payload congelado, histórico resumido das tentativas, último código HTTP retornado e timestamp do descarte.

3. **Replay Manual Administrativo:**
   - Exposição do endpoint `POST /admin/webhooks/dead-letter/:id/replay`, protegido pelo middleware `requireRole('ADMIN')`, permitindo que operadores reabilitem o evento transferindo-o novamente para a tabela `webhook_outbox` com status `PENDING` após a correção da falha pelo parceiro.

## Alternativas Consideradas

### Retentativas Agressivas Curtas (3 tentativas em até 30 minutos)
- **Descrição:** Realizar apenas 3 tentativas com intervalos curtos (ex.: 1m, 5m, 15m), descartando a mensagem após 30 minutos.
- **Motivo do Descarte:** Descartado na reunião técnica ([09:16] Diego). Parceiros corporativos realizam manutenções planejadas que frequentemente duram entre 1 e 2 horas. Uma política restrita a 30 minutos descartaria precocemente centenas de eventos legítimos, exigindo intervenção manual constante da equipe de suporte e degradando a confiança dos clientes B2B.

### Retentativas Indefinidas / Infinitas sem Teto
- **Descrição:** Continuar retentando o envio para sempre aplicando um teto fixo de intervalo (ex.: a cada 24 horas) até que o cliente responda com sucesso.
- **Motivo do Descarte:** Descartado na reunião técnica ([09:15] Diego). Clientes que descontinuam servidores ou removem endpoints sem aviso gerariam acúmulo infinito de linhas ativas na tabela outbox, degradando índices do MySQL e desperdiçando conexões de rede do worker em chamadas natimortas.

### Marcação de Falha em Coluna Flag na Própria Tabela Outbox
- **Descrição:** Manter os eventos falhos na mesma tabela `webhook_outbox`, alterando apenas uma coluna `status = 'FAILED'`.
- **Motivo do Descarte:** Descartado na reunião técnica ([09:18] Diego e Larissa). O acúmulo de registros com falha permanente na tabela outbox fragmentaria os índices de busca rápida (`status = 'PENDING'`), encarecendo as consultas em lote a cada 2 segundos realizadas pelo worker. A tabela separada isola o fluxo quente do histórico frio.

## Consequências

### Positivas
- **Alta Tolerância a Indisponibilidades Corporativas:** A janela de 15 horas absorve com folga paradas noturnas, manutenções em finais de semana ou reinicializações prolongadas nos sistemas dos clientes.
- **Higiene e Performance no Banco de Dados:** A tabela outbox permanece limpa e compacta, contendo apenas registros ativos no ciclo de vida corrente.
- **Auditoria e Capacidade de Recuperação:** Eventos não entregues não desaparecem; ficam arquivados na DLQ com motivo documentado e podem ser reprocessados sob demanda por endpoint administrativo seguro.

### Negativas e Trade-offs
- **Complexidade de Modelagem e Transição:** Exige gerenciar o ciclo de vida de duas tabelas relacionais (`webhook_outbox` e `webhook_dead_letter`) e implementar transições atômicas entre elas.
- **Necessidade de Intervenção Manual em Falhas Graves:** Caso o parceiro permaneça fora do ar por mais de 15 horas, o reprocessamento exigirá acionamento humano através da rota administrativa de replay.

## Integração com o Código Base / Rastreabilidade
- **Modelagem Prisma:** Criação dos modelos `WebhookOutbox` e `WebhookDeadLetter` no arquivo `prisma/schema.prisma`.
- **Controle de Acesso RBAC:** O endpoint de replay integrará o middleware de autenticação existente `src/middlewares/auth.middleware.ts`, invocando `requireRole('ADMIN')`.
- **Códigos de Erro Padronizados:** Integração de erros de exaustão e validação (`WEBHOOK_MAX_RETRIES_EXCEEDED`, `WEBHOOK_DLQ_NOT_FOUND`) no padrão de `src/shared/errors/index.ts`.
- **Rastreabilidade na Transcrição:** Deliberado e formalizado por Larissa, Bruno e Diego em `[09:14] Larissa`, `[09:15] Diego`, `[09:16] Bruno e Diego`, `[09:17] Larissa`, `[09:18] Diego e Bruno`, `[09:35] Diego` e `[09:36] Sofia e Larissa`.
