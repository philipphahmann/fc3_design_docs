# Prompt Especializado: Geração Universal de RFC a partir de Transcrição Técnica

> **Arquivo:** `docs/prompts/prompt_rfc.md`  
> **Finalidade:** Prompt agnóstico e reutilizável para converter deliberações de uma reunião técnica em um Request for Comments (RFC) conciso de arquitetura (2 a 4 páginas).  
> **Artefato Alvo:** `docs/F001/RFC.md` (ou caminho parametrizado em `{{CAMINHO_DESTINO}}`)  

---

## 1. Visão Geral e Como Utilizar

Este prompt orienta um modelo de linguagem a atuar como um **Arquiteto de Soluções e Tech Lead**, sintetizando a visão técnica consolidada de uma feature em uma **RFC (Request for Comments)** formal.

### O Papel da RFC no Pacote de Design Docs
- **PRD:** Define *"Por que e o quê?"* (Visão de Produto e Negócio).
- **ADRs:** Registram *"Por que decidimos pontualmente cada decisão?"* (Decisões isoladas e fechadas).
- **RFC:** Responde 👉 **"Como propomos resolver no todo e o que ainda está em aberto para discussão?"** (Proposta técnica global para revisão da equipe, de 2 a 4 páginas).
- **FDD:** Responde *"Como construir em detalhe?"* (Especificação profunda de código e endpoints).

### Parâmetros de Entrada
Ao invocar o prompt, o usuário deve informar:
- **`ARQUIVO_TRANSCRICAO`**: Caminho relativo ou conteúdo do arquivo de transcrição (ex.: `TRANSCRICAO.md`).
- **`CAMINHO_DESTINO`**: Caminho do arquivo da RFC gerada (ex.: `docs/F001/RFC.md`).
- **`DIRETORIO_ADRS`**: Caminho onde estão localizadas as ADRs já geradas para vinculação de links (ex.: `docs/F001/adrs/`).
- **`CONTEXTO_SISTEMA`** *(opcional)*: Informações sobre o sistema e tecnologias existentes (ex.: `src/`, `prisma/schema.prisma`).

---

## 2. Observações Críticas para o Sucesso do Prompt: Conteúdo Esperado na Transcrição

Para que uma transcrição gere uma RFC consistente, completa e sem lacunas, a reunião gravada precisa ter abordado as seguintes discussões:

| Seção da RFC | O que a Transcrição Precisa Conter para Alimentar a RFC |
| :--- | :--- |
| **1. Metadados e Revisores** | Identificação dos participantes da reunião para constarem como revisores técnicos da proposta. |
| **2. Resumo Executivo (TL;DR)** | Visão concisa da solução técnica que a equipe pretende adotar. |
| **3. Contexto e Problema** | Cenário técnico atual, limitações da arquitetura existente e impacto de não resolver o problema. |
| **4. Proposta Técnica Global** | Visão conceitual da arquitetura proposta (como os componentes interagem sem descer a contratos HTTP minuciosos). |
| **5. Alternativas Consideradas** | Pelo menos **duas abordagens reais descartadas** pela equipe durante a reunião, incluindo os trade-offs que motivaram o descarte. |
| **6. Questões em Aberto** | Pelo menos **dois pontos levantados que não foram fechados**, deixados como pontos de observação futura ou postergados para fases posteriores. |
| **7. Impacto e Riscos** | Impactos colaterais na infraestrutura, banco de dados ou processos existentes, e riscos técnicos associados. |
| **8. Decisões Relacionadas** | Conexão direta com as decisões registradas nas ADRs (pelo menos 2 links para ADRs existentes). |

> ⚠️ **Regra de Não-Ocultação:** Todas as 8 seções da RFC são mandatórias. Se algum tópico não tiver sido discutido na call (ex.: se a equipe fechou tudo e não deixou questões em aberto), a seção **NUNCA deve ser omitida**, devendo conter a anotação explícita: `> *Não identificado na reunião / Nenhuma questão em aberto registrada pelo time.*`

---

## 3. Prompt Mestre (Agnóstico e Parametrizável)

Copie o bloco abaixo para executar a geração da RFC com qualquer transcrição:

```markdown
Você é um Arquiteto de Software Principal e Tech Lead com ampla experiência na elaboração de propostas arquiteturais formais para times de engenharia. Sua tarefa é analisar uma transcrição de reunião técnica e elaborar uma RFC (Request for Comments) concisa, objetiva e elegante (extensão entre 2 e 4 páginas), submetida à equipe para revisão da solução técnica.

---

### PARÂMETROS DA EXECUÇÃO
- **Arquivo de Transcrição:** `{{ARQUIVO_TRANSCRICAO}}`
- **Caminho de Destino da RFC:** `{{CAMINHO_DESTINO}}`
- **Diretório das ADRs Relacionadas:** `{{DIRETORIO_ADRS}}`
- **Contexto do Sistema / Código Base (se aplicável):** `{{CONTEXTO_SISTEMA}}`

---

### DIRETRIZES DE ALTURA CONCEITUAL E ESTILO (CRÍTICAS)

1. **Nível de Abstração Correto:**
   - A RFC fala em **Arquitetura e Decisão**, não em detalhe de implementação linha a linha.
   - Apresente diagramas conceituais (Mermaid), fluxos de blocos e visão geral dos componentes.
   - NÃO detalhe contratos completos com payloads JSON exaustivos de endpoints ou matrizes de erro extensas (esse nível pertence exclusivamente ao FDD).
   - Mantenha o documento enxuto: conciso entre **2 e 4 páginas** de leitura técnica fluida.

2. **Mapeamento de Revisores:**
   - Extraia da transcrição os nomes dos participantes técnicos presentes na discussão e liste-os no campo de **Revisores** dos metadados da RFC.

3. **Alternativas Reais com Trade-offs:**
   - A seção "Alternativas Consideradas" deve listar obrigatoriamente **pelo menos 2 alternativas reais debatidas e descartadas** na reunião, detalhando claramente a motivação do descarte e o trade-off envolvido.

4. **Questões em Aberto:**
   - Identifique **pelo menos 2 pontos de dúvida técnica, itens postergados para observação ou decisões deixadas em aberto** na reunião.

5. **Links Diretos para ADRs:**
   - Na seção "Decisões Relacionadas", crie links diretos em Markdown para **pelo menos 2 ADRs** existentes no diretório `{{DIRETORIO_ADRS}}`.
   - Utilize a sintaxe de link relativo: `[ADR-NNN - Título](adrs/ADR-NNN-titulo.md)`.

6. **Anti-Alucinação e Tratamento de Lacunas:**
   - Baseie todas as informações estritamente na transcrição e no contexto do sistema fornecido.
   - Nenhuma das 8 seções abaixo pode ser ocultada. Se um tema não tiver sido abordado na reunião, mantenha o cabeçalho intacto e declare:  
     `> *Não discutido na reunião / Tema não abordado na transcrição fornecida.*`

---

### ESTRUTURA MANDATÓRIA DA RFC (8 SEÇÕES)

A RFC gerada deve conter EXATAMENTE as seguintes 8 seções:

# RFC — Request for Comments: [Título da Proposta Técnica Extraído da Transcrição]

## 1. Metadados
- **Autor(es):** [Tech Lead / Arquiteto identificado na transcrição]
- **Status:** Proposta Aberta para Revisão (ou Aprovado com Condições)
- **Data:** [Data da reunião ou atual]
- **Revisores:** [Lista de participantes presentes na reunião técnica com seus papéis identificados]

---

## 2. Resumo Executivo (TL;DR)
- Síntese concisa da proposta em 2 a 3 parágrafos: qual é o problema central, qual abordagem técnica está sendo proposta para solucioná-lo e os principais pilares da solução.

---

## 3. Contexto e Problema
- Cenário atual da arquitetura e limitações do sistema que motivaram a proposta.
- Por que a abordagem atual é insustentável técnica ou operacionalmente.
- Premissas e restrições técnicas que delimitam a solução.

---

## 4. Proposta Técnica
- **Visão Geral da Arquitetura:** Desenho macro da solução proposta, relacionando os novos componentes aos módulos existentes.
- **Diagrama Arquitetural:** Diagrama conceitual em Mermaid (`graph TD` ou `sequenceDiagram`) demonstrando o ciclo de vida e a interação entre os componentes.
- **Fluxo Geral de Funcionamento:** Como a informação trafega do evento de origem até o consumidor ou processador final.
- **Estratégia de Desacoplamento e Resiliência:** Abordagem de isolamento, assincronismo, políticas de tolerância a falhas e recuperação.

---

## 5. Alternativas Consideradas
*(Mínimo de 2 alternativas reais discutidas e descartadas na reunião)*

### 5.1 [Nome da Alternativa Descartada 1]
- **Descrição da Abordagem:** [O que foi cogitado.]
- **Por que foi Descartada:** [Argumentos técnicos da equipe (ex.: complexidade de infraestrutura desnecessária, risco de concorrência, acoplamento transacional).]
- **Trade-off:** [O que se perderia ou ganharia se tivesse sido escolhida.]

### 5.2 [Nome da Alternativa Descartada 2]
- **Descrição da Abordagem:** ...
- **Por que foi Descartada:** ...
- **Trade-off:** ...

---

## 6. Questões em Aberto e Próximos Passos
*(Mínimo de 2 questões ou pontos técnicos deixados em aberto ou adiados na reunião)*

- **Questão 1: [Título do Ponto em Aberto]**  
  [Descrição da dúvida técnica, o que ficou acordado observar em produção ou o critério que definirá a decisão em fase futura.]
- **Questão 2: [Título do Ponto em Aberto]**  
  [Descrição do ponto postergado e impacto esperado.]

---

## 7. Impactos e Riscos Técnicos
- **Impacto em Sistemas Existentes:** Alterações requeridas no banco de dados, rotinas de deploy, processos de worker ou APIs legadas.
- **Riscos Arquiteturais e Mitigações:** Potenciais gargalos (ex.: contention de banco, sobrecarga de rede, clientes lentos) e medidas de proteção adotadas.

---

## 8. Decisões Relacionadas (ADRs)
*(Vínculo com pelo menos 2 ADRs geradas para o projeto)*

- [ADR-001 - Título da Decisão](adrs/ADR-001-titulo-kebab-case.md): Resumo em uma linha de como esta decisão fundamenta a proposta.
- [ADR-002 - Título da Decisão](adrs/ADR-002-titulo-kebab-case.md): Resumo em uma linha da decisão e seu papel na arquitetura.
*(Incluir links adicionais se houver mais ADRs relevantes no pacote).*

---

### INSTRUÇÕES FINAIS DE FORMATAÇÃO E SAÍDA:
1. Gere o documento no idioma Português (Brasil).
2. Escreva o conteúdo em Markdown puro, iniciando diretamente pelo título `# RFC — Request for Comments: ...`.
3. Mantenha a extensão e densidade no padrão formal de 2 a 4 páginas de leitura técnica.
4. Não inclua comentários introdutórios antes do Markdown nem notas explicativas ao final. O conteúdo gerado deve ser exatamente o texto a ser gravado em `{{CAMINHO_DESTINO}}`.
```

---

## 4. Guia de Aplicação com Exemplo Prático

Para aplicar este prompt na transcrição do desafio atual:

```bash
# Exemplo de parametrização:
ARQUIVO_TRANSCRICAO: "TRANSCRICAO.md"
CAMINHO_DESTINO: "docs/F001/RFC.md"
DIRETORIO_ADRS: "docs/F001/adrs"
CONTEXTO_SISTEMA: "Código Node.js/TypeScript em src/ e modelagem Prisma em prisma/schema.prisma"
```
