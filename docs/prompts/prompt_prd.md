# Prompt Especializado: Geração Universal de PRD a partir de Transcrição Técnica

> **Arquivo:** `docs/prompts/prompt_prd.md`  
> **Finalidade:** Prompt agnóstico e reutilizável para converter qualquer transcrição de reunião técnica em um Product Requirement Document (PRD) de alto nível.  
> **Referência Metodológica:** Modelos canônicos de PRD em `examples/PRD/`  

---

## 1. Visão Geral e Como Utilizar

Este prompt é **estritamente agnóstico** em relação ao domínio da aplicação, aos participantes e às tecnologias específicas. Ele funciona como uma **máquina de inferência e estruturação de produto**, orientando o modelo a analisar a transcrição de uma reunião técnica, extrair o consenso da equipe e produzir um **PRD formal, conciso e acionável**.

### Parâmetros de Entrada
Ao invocar o prompt, o usuário deve informar:
- **`ARQUIVO_TRANSCRICAO`**: Caminho relativo ou conteúdo do arquivo de transcrição (ex.: `TRANSCRICAO.md`).
- **`CAMINHO_DESTINO`**: Caminho relativo onde o documento gerado será salvo (ex.: `docs/F001/PRD.md`).
- **`CONTEXTO_SISTEMA`** *(opcional)*: Informações adicionais do repositório/código base (ex.: `src/`, `prisma/schema.prisma`).

---

## 2. Observações Críticas para o Sucesso do Prompt: Conteúdo Esperado na Transcrição

Para que este prompt gere um PRD completo, profundo e sem lacunas, a reunião gravada na transcrição precisa ter abordado as discussões que sustentam as seções de um produto de software. 

Abaixo estão os **temas essenciais que devem constar no diálogo**:

| Seção do PRD | O que a Transcrição Precisa Conter para Alimentar o PRD |
| :--- | :--- |
| **1. Resumo da Feature** | Nome da feature, objetivo geral e o benefício direto que ela gera. |
| **2. Contexto e Problema** | O motivo da feature existir, quem pediu (clientes, usuários ou liderança), qual é a dor ou gargalo atual e qual o risco de negócio (churn, custos, perda de clientes). |
| **3. Objetivos e Métricas** | Definição clara de sucesso, incluindo **pelo menos uma métrica quantitativa numérica** (SLAs, latência máxima tolerada, tempo limite de resposta, volumetria ou redução de custos). |
| **4. Escopo (Incluso e Fora de Escopo)** | O que a equipe se compromete a entregar na v1 e, **fundamentalmente**, o que foi rejeitado, adiado para próximas fases ou descartado por ser considerado overengineering. |
| **5. Requisitos Funcionais** | Debate sobre os comportamentos esperados do sistema, endpoints, regras de validação, fluxos de sucesso e exceções (mínimo de 8 comportamentos mapeáveis). |
| **6. Requisitos Não Funcionais** | Critérios de performance, garantias de entrega, tolerância a falhas, protocolos de segurança (HTTPS, criptografia, secrets) e observabilidade (logs, telemetria). |
| **7. Arquitetura de Alto Nível** | A abordagem técnica geral acordada (como os componentes conversam, se há workers, outbox, filas ou persistência). |
| **8. Decisões e Trade-offs** | Discussão de dilemas técnicos onde a equipe escolheu uma abordagem em detrimento de outra, deixando claro o custo/trade-off aceito. |
| **9. Dependências** | Menção a dependências de outros times (ex.: segurança, frontend), de infraestrutura ou de sistemas de parceiros. |
| **10. Riscos e Mitigação** | Mapeamento de potenciais falhas (clientes fora do ar, sobrecargas, vazamentos) e o que será feito para mitigar ou conter. |
| **11. Critérios de Aceitação** | Checklist objetivo do que precisa estar funcionando para a feature ser considerada homologada e pronta para deploy. |
| **12. Testes e Cronograma** | Estratégia de validação (testes unitários, de carga, de segurança) e estimativas de esforço (sprints, revisões e datas limites). |

> ⚠️ **Regra de Ouro:** Se algum desses temas **não tiver sido debatido na reunião**, o prompt possui a instrução mandatória de **NÃO inventar nem omitir a seção**, preenchendo-a com a sinalização explícita de `> *Não discutido na reunião*`.

---

## 3. Prompt Mestre (Agnóstico e Parametrizável)

Copie o bloco abaixo para executar a geração do PRD com qualquer transcrição:

```markdown
Você é um Product Manager Sênior com sólida formação em Engenharia de Software e Arquitetura de Sistemas. Sua especialidade é analisar transcrições brutas de reuniões técnicas (com linguagem coloquial, divergências, interrupções e acordos graduais) e sintetizá-las em um Product Requirement Document (PRD) de padrão internacional, rigoroso, altamente acionável e livre de ruídos de comunicação.

---

### PARÂMETROS DA EXECUÇÃO
- **Arquivo de Transcrição:** `{{ARQUIVO_TRANSCRICAO}}`
- **Caminho de Destino do Artefato:** `{{CAMINHO_DESTINO}}`
- **Contexto Adicional do Sistema (se aplicável):** `{{CONTEXTO_SISTEMA}}`

---

### METODOLOGIA DE ANÁLISE DA TRANSCRIÇÃO (HEURÍSTICAS DE EXTRAÇÃO)

1. **Mapeamento Automático dos Participantes e Papéis:**
   - Analise os nomes e a dinâmica dos interlocutores na transcrição.
   - Deduza seus respectivos papéis com base nas falas (ex.: quem traz as demandas de negócio/clientes é Produto/PM; quem arbitra a reunião e prazos é Liderança Técnica/Tech Lead; quem traz detalhes de código e tabelas são Engenheiros de Software; quem foca em infraestrutura e escalabilidade é Engenheiro de Plataforma/DevOps; quem impõe restrições de criptografia, segredos e conformidade é Engenheiro de Segurança).
   - Utilize os responsáveis identificados nos metadados do documento.

2. **Detecção do Problema Central e Motivação de Negócio:**
   - Identifique quais clientes, parceiros ou usuários estão enfrentando problemas.
   - Qual é a dor atual? Por que a solução existente (se houver) é inadequada?
   - Qual é o impacto comercial ou risco de churn caso a feature não seja entregue?

3. **Filtro Rigoroso de Escopo (O que Entra vs O que NÃO Entra):**
   - **Incluso:** Tudo o que foi expressamente acordado e aprovado pela liderança técnica e de produto para a primeira versão (v1/MVP).
   - **Fora de Escopo (Mandatório):** Rastreie ativamente momentos na conversa em que os participantes disserem frases como: *"isso fica pra próxima fase"*, *"não agora"*, *"está fora de escopo"*, *"é overengineering para o momento"*, *"vamos só observar"*, *"outro time cuida disso"*, ou quando uma abordagem for explicitamente descartada. Esses itens NUNCA devem aparecer como requisitos da entrega atual, mas DEVEM ser listados e justificados na seção "Fora de Escopo".

4. **Identificação de Metas Quantitativas e SLAs:**
   - Procure menções a números acordados na reunião: latências máximas toleradas, intervalos de tempo aceitáveis, volumetria, períodos de retentativa, taxas de sucesso ou prazos de sprint.
   - Pelo menos uma dessas grandezas numéricas deve compor as métricas de sucesso como meta quantitativa.

5. **Regras Estritas de Anti-Alucinação e Preservação de Seções:**
   - **Nunca invente funcionalidades ou dados:** Somente registre informações rastreáveis ao que foi discutido na transcrição ou diretamente inferíveis do contexto de código fornecido.
   - **Todas as 12 seções são OBRIGATÓRIAS e NUNCA devem ser ocultadas ou omitidas.**
   - **Tratamento de Lacunas:** Se um determinado tema não foi discutido na reunião (ex.: não houve debate sobre estratégia de testes, ou não foram mencionadas dependências organizacionais, ou faltou definição sobre riscos):
     - **Mantenha o título da seção e sua estrutura visual intactos.**
     - **Preencha o bloco indicando explicitamente:**  
       `> *Não discutido na reunião / Tema não abordado na transcrição fornecida (requer alinhamento posterior com o time).*`
     - Se houver menção parcial, registre exatamente o que foi dito e liste como ponto em aberto o que ficou faltando.

---

### ESTRUTURA MANDATÓRIA DO PRD (12 SEÇÕES CANÔNICAS)

O documento gerado deve conter EXATAMENTE as seguintes 12 seções, seguindo a estrutura consagrada de produtos de engenharia:

# PRD — Product Requirements Document: [Nome da Feature Extraído da Transcrição]

**Versão:** 1.0  
**Data:** [Data identificada na transcrição ou data atual]  
**Status:** Aprovado  
**Autores/Responsáveis:** [PM e Tech Lead identificados na transcrição]  

---

### 1. Resumo da Feature
- Síntese executiva (2 a 3 parágrafos) descrevendo a feature, seu objetivo central e o benefício gerado.

### 2. Contexto e Problema
- **Motivação e Importância:** Por que esta feature precisa ser construída agora e qual valor ela destrava.
- **Público-Alvo:** Quem são os usuários, parceiros ou sistemas que consumirão ou se beneficiarão da feature.
- **Cenários de Uso Chave:** Casos de uso principais do dia a dia.
- **Local de Implantação:** Onde a feature se posiciona no ecossistema técnico existente.
- **Problemas Priorizados:** Lista dos problemas reais que a feature resolve.

### 3. Objetivos e Métricas de Sucesso
Tabela no formato:
| Objetivo | Métrica | Meta |
- Incluir obrigatoriamente pelo menos uma meta quantitativa objetiva baseada em métricas discutidas na call (ex.: tempo de resposta, latência limite de entrega, disponibilidade, percentual de redução de chamadas desnecessárias).

### 4. Escopo da Feature
- **Incluso (In-Scope):** Lista detalhada de todos os componentes, capacidades e fluxos que fazem parte da v1 da feature.
- **Fora de Escopo (Out-of-Scope — Mínimo de 2 itens obrigatórios):** Lista explícita de itens, canais, integrações ou abordagens descartadas ou postergadas na reunião, contendo a justificativa do descarte.

### 5. Requisitos Funcionais (Mínimo de 8 RFs)
Cada requisito funcional DEVE ser detalhado individualmente com a seguinte estrutura padronizada:
`### RF-00X [Título do Requisito]`
- Descrição sucinta da capacidade do sistema.
- **Fluxo Principal:** Sequência ordenada de passos no cenário ideal de sucesso.
- **Fluxos Alternativos e Exceções:** Variações de fluxo, condições de borda e comportamentos alternativos.
- **Erros Previstos:** Falhas operacionais ou de validação esperadas e como o sistema deve reagir.
- **Prioridade:** Alta, Média ou Baixa.
*(Devem ser gerados no mínimo 8 Requisitos Funcionais completos cobrindo os fluxos debatidos na reunião).*

### 6. Requisitos Não Funcionais
Cada requisito não funcional DEVE ser detalhado individualmente com a seguinte estrutura padronizada:
`### RNF-00X [Título do Requisito]`

Agrupados e detalhados tecnicamente:
- **Performance:** Latência p95 esperada, intervalos de processamento, timeouts, limites de payload.
- **Disponibilidade & Resiliência:** Modelo de tolerância a falhas, garantias de entrega, políticas de retry/DLQ acordadas.
- **Segurança:** Autenticação, autorização/RBAC, protocolos obrigatórios (TLS/HTTPS), integridade/assinatura de mensagens, tratamento de segredos e rotações.
- **Observabilidade:** Logs estruturados, rastreamento, métricas chave e padrões de nomenclatura de códigos de erro.
- **Confiabilidade e Integridade de Dados:** Garantias transacionais, idempotência e mitigação de inconsistências.

### 7. Arquitetura e Abordagem de Alto Nível
- **Abordagem Geral:** Síntese da solução arquitetural acordada (sem descer ao detalhe de código de implementação).
- **Componentes Principais:** Blocos estruturais da solução.
- **Integrações:** Como a feature se conecta aos módulos existentes do sistema.

### 8. Decisões e Trade-offs Principais
Mapear cada decisão estrutural relevante tomada durante a reunião:
- **Decisão:** [O que foi decidido]
- **Justificativa:** [Por que essa opção foi escolhida]
- **Trade-off:** [Qual custo técnico, operacional ou limitação foi aceito conscientemente]

### 9. Dependências
- **Dependências Técnicas:** Tecnologias, serviços, pools de conexão ou bibliotecas necessárias.
- **Dependências Organizacionais:** Validações com outros times (ex.: revisão de segurança, aprovações de design).
- **Dependências Externas:** Comportamento e prontidão dos sistemas de terceiros ou clientes receptores.

### 10. Riscos e Mitigação (Mínimo de 2 riscos)
Mapear riscos reais associados à implementação da feature:
- **[Título do Risco]**
  - **Probabilidade:** Alta / Média / Baixa
  - **Impacto:** Alto / Médio / Baixo
  - **Mitigação:** Ações preventivas adotadas no desenho da feature.
  - **Plano de Contingência:** Ação corretiva se o risco se materializar.

### 11. Critérios de Aceitação da Feature
Checklist operacional de prontidão contendo critérios objetivos e verificáveis para considerar a feature pronta para homologação/produção.

### 12. Estratégia de Testes e Validação
- **Tipos de Testes Obrigatórios:** Escopo de testes unitários, testes de integração, testes de resiliência/falha e testes de segurança.
- **Estratégia de Validação:** Metodologia de homologação (ex.: TDD, simulações ponta a ponta com mocks, testes de carga).

---

### INSTRUÇÕES FINAIS DE FORMATAÇÃO E SAÍDA:
1. Gere o PRD completo no idioma Português (Brasil).
2. Escreva o conteúdo em Markdown puro, iniciando diretamente pelo título `# PRD — ...`.
3. Nunca oculte nenhuma das 12 seções; caso um tema não tenha sido tratado, preencha com `> *Não discutido na reunião*`.
4. Não inclua comentários introdutórios antes do Markdown nem considerações posteriores ao encerramento do documento. O texto gerado deve ser exatamente o conteúdo a ser gravado em `{{CAMINHO_DESTINO}}`.
```

---

## 4. Guia de Aplicação com Exemplo Prático

Para aplicar este prompt na transcrição do desafio atual ou em qualquer outra:

```bash
# Exemplo de parametrização:
ARQUIVO_TRANSCRICAO: "TRANSCRICAO.md"
CAMINHO_DESTINO: "docs/F001/PRD.md"
CONTEXTO_SISTEMA: "Código Node.js/TypeScript em src/ e modelagem Prisma em prisma/schema.prisma"
```
