# ADR-003: Autenticação Criptográfica com HMAC-SHA256, Secret Única por Endpoint e Rotação Gradual

## Status
Aceito

## Contexto
O Order Management System (OMS) transmitirá notificações contendo dados comerciais sensíveis de pedidos (identificador do pedido, valores financeiros em centavos, cliente e transições de status) para endpoints receptores na internet pública. 

Para que os clientes parceiros confiem nas notificações recebidas, o mecanismo de comunicação deve atender a três propriedades criptográficas fundamentais:
1. **Autenticidade:** O cliente precisa comprovar matematicamente que o disparo se originou legitimamente dos servidores do OMS.
2. **Integridade:** O cliente precisa ter garantia de que o payload JSON não foi alterado em trânsito por intermediários maliciosos.
3. **Proteção contra Ataques de Repetição (Replay Attacks):** Um ator malicioso que capture um pacote válido não deve ser capaz de reenviá-lo repetidamente para induzir o cliente a duplicar processamentos internos.

Adicionalmente, requisitos de governança de segurança exigem capacidade de rotação periódica ou emergencial de credenciais sem interrupção abrupta no fluxo de negócios (*zero-downtime*).

## Decisão
Adotar o padrão da indústria de assinatura digital simétrica via **HMAC-SHA256**, atrelado a **segredos exclusivos por endpoint** e suporte a **rotação gradual com 24 horas de tolerância**:

1. **Assinatura HMAC-SHA256 e Cabeçalhos:**
   - No momento do disparo, o worker serializa o corpo do evento em JSON e computa o hash HMAC com algoritmo SHA-256 utilizando a chave secreta exclusiva cadastrada para aquele endpoint.
   - O hash hexadecimal resultante é injetado no cabeçalho HTTP `X-Signature`.
   - São injetados simultaneamente o cabeçalho `X-Timestamp` (em formato ISO 8601, permitindo ao receptor descartar requisições com divergência temporal excessiva) e o cabeçalho `X-Webhook-Id` (para identificação unívoca do registro cadastral pelo cliente).

2. **Isolamento de Segredos (Zero Compartilhamento Global):**
   - Cada endpoint de webhook possui sua própria chave secreta gerada com alta entropia no momento do cadastro. Não há chave mestra ou credencial compartilhada entre clientes ou entre múltiplos endpoints do mesmo parceiro.

3. **Ciclo de Rotação Gradual (Grace Period de 24 Horas):**
   - Criação do endpoint `POST /webhooks/:id/rotate-secret`.
   - Ao ser acionado, o sistema gera uma nova chave secreta e move a chave vigente para o campo de chave prévia (`previousSecret`), registrando a data da alteração (`secretRotatedAt`).
   - Durante uma janela de tolerância de **24 horas**, o sistema mantém ambas as chaves válidas para conferência pelo cliente. Após 24 horas, a chave antiga é revogada e descartada permanentemente.

4. **Revisão de Segurança Obrigatória:**
   - O código do módulo criptográfico e a gestão de chaves serão submetidos a uma janela formal de homologação de **2 dias úteis** com a Engenheira de Segurança (Sofia) antes do deploy em produção.

## Alternativas Consideradas

### Chave Secreta Estática Global da Plataforma
- **Descrição:** Utilizar uma única chave secreta compartilhada para todos os clientes ou para todos os webhooks de um mesmo cliente.
- **Motivo do Descarte:** Descartado categoricamente na reunião técnica ([09:21] Sofia e Diego). Se um cliente cometer um erro operacional (como expor a chave em logs públicos de aplicação ou repositórios), toda a plataforma seria comprometida, exigindo revogação em massa e impactando parceiros não relacionados.

### Autenticação Simples por API Key / Bearer Token Estático no Header
- **Descrição:** Enviar uma chave de autenticação estática no cabeçalho `Authorization: Bearer <token>` em cada requisição de notificação.
- **Motivo do Descarte:** Descartado na reunião técnica ([09:20] Sofia). Tokens estáticos comprovam a identidade do emissor mas **não protegem a integridade do payload**. Qualquer interceptação ou modificação em trânsito não seria detectada pelo cliente, além de ser vulnerável a ataques de repetição simples caso o token seja capturado.

### Rotação Imediata sem Período de Convivência (Hard Switchover)
- **Descrição:** Ao solicitar nova secret, invalidar a chave anterior no mesmo milissegundo.
- **Motivo do Descarte:** Descartado na reunião técnica ([09:21] Sofia). Clientes corporativos operam múltiplos clusters de servidores e necessitam de uma janela operacional de transição para distribuir a nova chave em seus ambientes de produção sem rejeitar pacotes em trânsito.

## Consequências

### Positivas
- **Aderência aos Melhores Padrões do Mercado:** Alinha o OMS às práticas de referências globais (GitHub, Stripe, Shopify), permitindo que os parceiros utilizem bibliotecas de validação prontas.
- **Proteção Completa da Informação:** Garante autenticidade, integridade e mitigação de replay attacks sem onerar a complexidade de chaves assimétricas públicas/privadas.
- **Mitigação de Impacto em Incidentes:** Vazamento de uma secret compromete exclusivamente um único endpoint de um único cliente, sendo contornável via rotação em minutos.

### Negativas e Trade-offs
- **Esforço de Implementação no Consumidor:** Exige que a equipe técnica do cliente parceiro escreva lógica de validação criptográfica HMAC e confira o timestamp do request antes de processar o evento.
- **Governança de Estado de Segredos no Banco:** O modelo de dados precisa armazenar campos de chave ativa, chave prévia e timestamp de corte para controlar o período de convivência de 24 horas.

## Integração com o Código Base / Rastreabilidade
- **Biblioteca Criptográfica:** Uso exclusivo do módulo nativo `crypto` do Node.js (`crypto.createHmac('sha256', secret).update(rawBody).digest('hex')`), evitando dependências externas não auditadas.
- **Modelagem de Dados:** O modelo de persistência no Prisma (`prisma/schema.prisma`) contemplará as colunas `secret`, `previousSecret` e `secretRotatedAt` como campos protegidos que nunca são serializados em respostas de `GET`.
- **Rastreabilidade na Transcrição:** Deliberado e formalizado por Sofia, Diego, Bruno e Larissa em `[09:19] Sofia`, `[09:20] Sofia e Bruno`, `[09:21] Sofia`, `[09:22] Diego e Sofia`, `[09:44] Diego e Sofia`, `[09:46] Sofia` e `[09:47] Larissa`.
