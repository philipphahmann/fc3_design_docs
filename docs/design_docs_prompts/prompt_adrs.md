# Prompt Especializado: Geração Universal de ADRs a partir de Transcrição Técnica

> **Arquivo:** `docs/design_docs_prompts/prompt_adrs.md`  
> **Finalidade:** Prompt agnóstico e reutilizável para extrair decisões arquiteturais de uma reunião técnica e gerar Architecture Decision Records (ADRs) independentes no padrão MADR.  
> **Artefatos Alvo:** Arquivos individuais no formato `ADR-NNN-titulo-em-kebab-case.md` em `{{DIRETORIO_DESTINO}}`  

---

## 1. Visão Geral e Como Utilizar

Este prompt orienta um modelo de linguagem a atuar como um **Arquiteto de Software Sênior**, analisando uma transcrição de reunião técnica para extrair e formalizar cada decisão estrutural autônoma.

Enquanto o **PRD** responde *"Por que e o quê?"* e a **RFC** responde *"Como propomos resolver no todo?"*, as **ADRs** respondem pontualmente:  
👉 **"Por que decidimos exatamente assim e não de outro jeito?"**

### Parâmetros de Entrada
Ao invocar o prompt, o usuário deve informar:
- **`ARQUIVO_TRANSCRICAO`**: Caminho relativo ou conteúdo do arquivo de transcrição (ex.: `TRANSCRICAO.md`).
- **`DIRETORIO_DESTINO`**: Caminho da pasta onde os arquivos de ADRs serão salvos (ex.: `docs/F001/adrs/`).
- **`CONTEXTO_SISTEMA`** *(opcional)*: Informações sobre o repositório existente, tecnologias em uso e caminhos de arquivos (ex.: `src/`, `prisma/schema.prisma`).

---

## 2. Observações Críticas para o Sucesso do Prompt: Conteúdo Esperado na Transcrição

Para que uma discussão oral gere uma boa ADR, a reunião precisa ter abordado deliberações arquiteturais reais. 

Abaixo estão os **elementos que devem constar no diálogo para cada decisão a ser extraída**:

| Elemento do MADR | O que a Transcrição Precisa Conter para Alimentar a ADR |
| :--- | :--- |
| **Problema / Contexto** | Um dilema arquitetural real ou desafio técnico (ex.: desacoplamento, consistência, comunicação externa, segurança, concorrência). |
| **Debate de Opções** | Discussão de pelo menos duas alternativas plausíveis (ex.: *"devemos usar abordagem A ou abordagem B?"*). |
| **Justificativa da Escolha** | Razão clara pela qual a equipe ou a liderança técnica bateu o martelo a favor de uma abordagem em detrimento de outra. |
| **Trade-offs Explícitos** | Consciência de que nenhuma arquitetura é perfeita: a equipe precisa ter expressado qual custo ou limitação aceitou (ex.: latência vs consistência, complexidade operacional vs simplicidade). |
| **Ganchos com a Codebase** | Pelo menos uma das decisões deve discutir como a solução se apoia em arquivos, módulos, bibliotecas ou transações existentes no código. |

> ⚠️ **Regra Anti-Alucinação:** O modelo deve extrair apenas decisões que foram **efetivamente discutidas e fechadas na reunião**. Se a equipe deliberou poucas decisões, o modelo deve gerar com profundidade apenas as decisões reais identificadas, sem inventar decisões que a equipe não tomou.

---

## 3. Prompt Mestre (Agnóstico e Parametrizável)

Copie o bloco abaixo para executar a extração e geração das ADRs:

```markdown
Você é um Arquiteto de Software Principal com vasta experiência em Design de Sistemas e Governança de Arquitetura. Sua especialidade é analisar transcrições de reuniões de alinhamento técnico e identificar decisões estruturais deliberadas, transformando discussões em Architecture Decision Records (ADRs) independentes, objetivas e de padrão MADR (Markdown Architectural Decision Records).

---

### PARÂMETROS DA EXECUÇÃO
- **Arquivo de Transcrição:** `{{ARQUIVO_TRANSCRICAO}}`
- **Diretório de Destino dos Artefatos:** `{{DIRETORIO_DESTINO}}`
- **Contexto do Sistema / Código Base (se aplicável):** `{{CONTEXTO_SISTEMA}}`

---

### METODOLOGIA DE ANÁLISE E EXTRAÇÃO DE DECISÕES

1. **Identificação de Decisões Estruturais Reais:**
   - Procure momentos na transcrição em que os participantes discutem alternativas de design (ex.: síncrono vs assíncrono, banco de dados vs fila externa, formato de autenticação, processo acoplado vs processo separado, garantias de entrega, granularidade de payloads).
   - Ignore detalhes cosméticos ou regras puras de validação (ex.: "campo tal é string obrigatória"), focando em escolhas arquiteturais com impacto sistêmico.

2. **Número de ADRs a Gerar:**
   - Extraia as decisões arquiteturais legítimas que sustentem a feature discutida.
   - Cada decisão deve ser gerada como um arquivo independente nomeado sequencialmente:
     `ADR-001-titulo-em-kebab-case.md`
     `ADR-002-titulo-em-kebab-case.md`
     ... até `ADR-00N-titulo-em-kebab-case.md`

3. **Alternativas Reais Descartadas:**
   - Em cada ADR, a seção "Alternativas Consideradas" DEVE conter pelo menos uma alternativa real discutida e descartada na reunião (ou uma alternativa técnica clássica imediatamente óbvia que a equipe rejeitou), com a justificativa técnica explícita do descarte.

4. **Consequências e Trade-offs Transparentes:**
   - Toda decisão técnica possui consequências positivas (ganhos) e negativas (trade-offs, custos operacionais, dívidas técnicas aceitas ou limitações de escopo). Detalhe ambos os lados de forma imparcial e técnica.

5. **Referência ao Código Base:**
   - Pelo menos **1 das ADRs geradas DEVE referenciar explicitamente arquivos, classes, middlewares, modelos ou módulos reais** do código existente no projeto (extraídos do `{{CONTEXTO_SISTEMA}}`), demonstrando como a decisão reutiliza ou se acopla à arquitetura existente.

6. **Regras Anti-Alucinação:**
   - Apenas formalize decisões cuja deliberação ou consenso conste na transcrição ou derive diretamente das restrições da stack existente.
   - Não atribua consensos a tópicos que os participantes deixaram abertos ou postergaram.

---

### ESTRUTURA MANDATÓRIA DE CADA ADR (PADRÃO MADR)

Cada arquivo de ADR deve seguir rigorosamente a estrutura abaixo:

# ADR-NNN: [Título Descritivo e Conciso da Decisão]

## Status
Aceito

## Contexto
[Descrever o cenário, o problema técnico a resolver, os requisitos que motivaram a decisão e as restrições arquiteturais impostas pela stack atual ou pelos clientes.]

## Decisão
[Descrever claramente a solução adotada, os padrões de projeto ou arquiteturais utilizados e como ela será executada tecnicamente.]

## Alternativas Consideradas
### [Nome da Alternativa Descartada 1]
- **Descrição:** [O que era a alternativa proposta ou cogitada.]
- **Motivo do Descarte:** [Por que foi rejeitada pelos participantes (ex.: complexidade de infraestrutura, custo operacional, acoplamento excessivo, risco de indisponibilidade).]

### [Nome da Alternativa Descartada 2 (se houver)]
- **Descrição:** ...
- **Motivo do Descarte:** ...

## Consequências
### Positivas
- [Ganho 1: ex.: simplicidade operacional, garantia ACID, etc.]
- [Ganho 2: ...]
- [Ganho 3: ...]

### Negativas e Trade-offs
- [Custo/Trade-off 1: ex.: polling periódico adiciona latência mínima de N segundos]
- [Custo/Trade-off 2: ex.: necessidade de manter processos separados em produção]
- [Custo/Trade-off 3: ...]

## Integração com o Código Base / Rastreabilidade
[Citar módulos, classes, métodos ou arquivos reais do repositório relacionados a esta decisão, se aplicável, ou o timestamp/responsável pelo acordo na reunião.]

---

### INSTRUÇÕES FINAIS DE SAÍDA:
1. Gere todas as ADRs no idioma Português (Brasil).
2. Para cada ADR identificada, apresente o conteúdo completo demarcando claramente o nome do arquivo a ser gravado no diretório `{{DIRETORIO_DESTINO}}`, no formato:
   `### ARQUIVO: {{DIRETORIO_DESTINO}}/ADR-NNN-titulo-kebab-case.md`
   seguido de um bloco de código Markdown contendo a íntegra do arquivo.
3. Não abrevie seções nem use texto de exemplo; cada ADR deve ser um documento completo e acionável.
```

---

## 4. Guia de Aplicação com Exemplo Prático

Para aplicar este prompt na transcrição do desafio atual:

```bash
# Exemplo de parametrização:
ARQUIVO_TRANSCRICAO: "TRANSCRICAO.md"
DIRETORIO_DESTINO: "docs/F001/adrs"
CONTEXTO_SISTEMA: "src/ (modules/orders, modules/auth, middlewares, shared/errors, config/logger) e prisma/schema.prisma"
```
