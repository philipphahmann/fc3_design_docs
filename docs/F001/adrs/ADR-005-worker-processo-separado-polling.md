# ADR-005: Worker Desacoplado em Processo Node.js Separado com Polling Periódico

## Status
Aceito

## Contexto
Com a adoção do padrão Transactional Outbox (ADR-001), o OMS necessita de um despachante assíncrono para ler eventos pendentes no MySQL, efetuar requisições HTTP POST para endpoints externos e gerenciar o ciclo de retentativas.

A aplicação base atual é inicializada exclusivamente pelo arquivo `src/server.ts`, que sobe o servidor HTTP Express atendendo requisições síncronas de gerenciamento de pedidos, autenticação e catálogo.

A equipe precisou definir a arquitetura de runtime para a execução desse despachante:
1. Deve ele rodar dentro da mesma instância do servidor Express ou em um processo Node.js independente?
2. Como o despachante deve ser acionado: via mecanismos de notificação do banco de dados ou via loop de consulta periódica (*polling*)?

## Decisão
Implementar o despachante de webhooks como um **processo Node.js autônomo e desacoplado (`src/worker.ts`)**, operando via **polling contínuo no banco de dados a cada 2 segundos**:

1. **Entry-Point e Script Dedicados:**
   - Criação do arquivo `src/worker.ts`, operando de forma análoga a `src/server.ts`, com inicialização independente e script npm dedicado no `package.json` (`npm run worker`).
   - A lógica central de processamento de lote residirá no módulo `src/modules/webhooks/webhook.worker.ts`.

2. **Intervalo de Polling e Parâmetros de Execução:**
   - O worker operará em loop contínuo com intervalo de sono (*sleep*) de **2 segundos** entre ciclos de busca.
   - Cada ciclo busca registros na tabela `webhook_outbox` com status `PENDING` ordenados por `created_at ASC`, limitando a leitura a lotes de 50 eventos (`LIMIT 50`).
   - Cada chamada HTTP para endpoint de cliente adota um timeout estrito de **10 segundos**.

3. **Pool de Conexões Dedicado:**
   - O processo do worker instanciará seu próprio objeto `PrismaClient`. Embora aponte para o mesmo banco e utilize a mesma variável de ambiente `DATABASE_URL`, o pool de conexões do worker não concorre diretamente com o pool alocado para a API HTTP.

4. **Operação Single-Worker com Ordenação Implícita:**
   - Nesta primeira versão, o sistema operará com uma única instância ativa do worker. Isso garante que os eventos de um mesmo pedido sejam despachados na exata ordem cronológica de sua criação (`created_at ASC`), sem necessidade de travas distribuídas complexas. A escalabilidade horizontal com particionamento por hash de `order_id` é deixada para fases futuras.

## Alternativas Consideradas

### Despacho em Background Dentro do Mesmo Processo da API Express (setInterval em src/server.ts)
- **Descrição:** Executar uma rotina de intervalo (`setInterval` ou fila em memória) dentro da própria instância do servidor HTTP que responde às rotas REST da API.
- **Motivo do Descarte:** Descartado categoricamente na reunião técnica ([09:11] Diego e Larissa). Se a API sofrer reinicialização decorrente de deploy, crash inesperado ou reciclagem de container, o processador de webhooks é interrompido simultaneamente. Além disso, clientes externos lentos retendo conexões HTTP por até 10 segundos consumiriam sockets, descritores de arquivo e recursos do *event-loop* da API, degradando o tempo de resposta das operações críticas dos operadores humanos.

### Notificação Reativa via Trigger de Banco de Dados ou Listener SQL
- **Descrição:** Utilizar gatilhos no MySQL para alertar imediatamente o despachante assim que uma linha for inserida na outbox.
- **Motivo do Descarte:** Descartado na reunião técnica ([09:09] Diego e Bruno). O MySQL não possui suporte nativo a comandos pub/sub de conexão (como o `LISTEN/NOTIFY` do PostgreSQL). Gatilhos em MySQL executam estritamente código SQL interno e não notificam processos externos no sistema operacional sem extensões inseguras ou hacks em arquivos. O polling de 2 segundos cumpre com folga a meta de entrega em até 10 segundos exigida pelos clientes B2B.

## Consequências

### Positivas
- **Isolamento de Falhas e Recursos:** Quedas ou saturações de memória na API não derrubam o despachante de webhooks, e rajadas de notificações externas não consomem o event-loop da API REST.
- **Independência Operacional de Deploy e Escala:** Permite iniciar, parar, reiniciar ou atualizar o worker sem impactar o uptime da API de atendimento a clientes.
- **Conformidade com SLA de Latência:** O ciclo a cada 2 segundos assegura que 100% dos eventos comitem e alcancem o parceiro em menos de 10 segundos em condições normais de carga.

### Negativas e Trade-offs
- **Complexidade de Orquestração em Produção:** Exige gerenciar dois processos/containers separados na infraestrutura (`server` e `worker`).
- **Limitação de Concorrência Inicial:** A premissa de worker único limita o throughput máximo a cenários onde a vazão sequencial atende o volume de pedidos, exigindo particionamento distribuído quando a empresa atingir patamares muito superiores de transações por segundo.

## Integração com o Código Base / Rastreabilidade
- **Padrão de Entrada:** Espelha a convenção arquitetural de inicialização já existente em `src/server.ts`, instanciando variáveis de ambiente de `src/config/env.ts` e logger de `src/shared/logger/index.ts`.
- **Instância Prisma:** O arquivo `src/worker.ts` importará e gerenciará um novo `PrismaClient` com encerramento gracioso via sinais `SIGTERM` e `SIGINT`.
- **Rastreabilidade na Transcrição:** Deliberado e formalizado por Diego, Bruno e Larissa em `[09:09] Bruno e Diego`, `[09:10] Marcos e Larissa`, `[09:11] Diego, Larissa e Bruno`, `[09:12] Larissa e Diego`, `[09:13] Diego e Larissa`, `[09:28] Bruno e Diego` e `[09:29] Diego e Bruno`.
