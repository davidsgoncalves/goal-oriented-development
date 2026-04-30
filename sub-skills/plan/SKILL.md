---
name: plan
description: |
  Lê a spec já produzida (`spec.md` no `specs_path`), detecta se o ambiente é single-project ou multi-project workspace, resolve a(s) branch(es) da task (nome + base) sem tocar no git, e escreve o plano de implementação focado em **HOW** (arquitetura, arquivos, passos). Não busca dados em Jira/Figma e não faz Q&A de escopo — isso é responsabilidade da skill `spec`. Não cria branch no git — a criação física é responsabilidade do `implement`. Use quando o usuário mencionar: "planejar task", "criar plano", "plan", ou quando a fase de planejamento for ativada pelo GOD.
tools: Read, Glob, Grep, Bash, Edit, Write, Agent
---

# Plan — Sub-skill de Planejamento

> Lê a spec canônica produzida pelo `spec`, consulta knowledge por tasks semelhantes em busca de **padrões técnicos** (não escopo), detecta modo single/multi-project, resolve a(s) branch(es) da task (nome + base por projeto afetado) e escreve o plano de implementação focado em **arquitetura, arquivos e passos**. Não cria branch no git — apenas determina os nomes e bases pra `implement` criar depois.

> **Mudança v6:** Q&A de escopo, busca em Jira/Figma e ACs migraram pra skill `spec`. Esta skill agora é puramente sobre HOW.

## Instruções

Quando o usuário invocar esta skill, execute os seguintes passos **na ordem**:

### 0. Executar hook `before plan`

Ler `GOD/hooks.md` e localizar a seção `# before plan`.

- Se o conteúdo for `skip-hook`: pular e seguir para o passo 1.
- Se houver instruções em linguagem natural: executá-las integralmente antes de prosseguir.

### 1. Identificar a task

- Se o contexto da conversa já contém o código da task, usar esse código
- Caso contrário, perguntar ao usuário o código da task (ex: `PROJ-123`)

### 2. Ler spec canônica e description, checar freshness

A spec é a **fonte de verdade do escopo**. Esta skill nunca altera a spec — apenas a consome.

1. **Ler `GOD/tasks/{cod-da-task}/status.md`** e extrair `spec_path` e `spec_version_consumed`.
   - Se `spec_path` está ausente ou vazio: a skill `spec` não rodou. Orientar o usuário a rodar `spec` primeiro e encerrar.
2. **Ler a spec** em `<spec_path>` (resolver relativo ao diretório do GOD se necessário).
   - Se o arquivo não existe no path indicado: pode ter sido movido manualmente. Pedir ao usuário pra confirmar o caminho atual ou re-rodar `spec`.
   - Extrair `spec_version` do frontmatter da spec (versão atual).
3. **Freshness check (a partir da v7):**
   - Se `spec_version_consumed` é `null` → primeira execução do `plan`, prosseguir.
   - Se `spec_version_consumed` < `spec_version` da spec atual → **a spec mudou desde a última vez que esse plan rodou**. Alertar o usuário:

     > ⚠️ A spec mudou desde a última execução deste plan.
     > - Versão consumida pela última vez: v{spec_version_consumed}
     > - Versão atual da spec: v{spec_version}
     >
     > O plano atual em `plan.md` pode estar desatualizado. Quer (a) rodar `update-plan` antes, (b) reescrever o plano do zero a partir da spec nova, (c) prosseguir mesmo assim?

   - Se igual → spec não mudou, prosseguir normalmente.
4. **Ler `GOD/tasks/{cod-da-task}/description.md`** apenas como contexto adicional (input bruto do usuário). A spec é a fonte canônica — a description não deve ser usada pra determinar requisitos nem critérios.

Extrair da spec:
- Lista de REQs e ACs (com IDs estáveis) — vão ser referenciados nos passos do plano
- Cenários e NFRs — input pra considerações técnicas
- Não-objetivos — input pra evitar scope creep no plano

### 2.1. Detectar modo single-project vs multi-project workspace

Antes de resolver a branch, identificar em qual tipo de ambiente o comando foi chamado:

- Executar `git rev-parse --show-toplevel` no diretório atual (`pwd`).
  - **Se sucede** e retorna um caminho: o diretório atual está dentro de um repositório git — **modo single-project**. A task terá **uma única branch**.
  - **Se falha** (erro "not a git repository"): o diretório atual é um workspace (pasta-mãe contendo múltiplos projetos, cada um com seu próprio `.git`) — **modo multi-project**. A task pode precisar de **uma branch em cada projeto afetado**.

Guardar o modo detectado em memória para os passos seguintes.

### 2.2. Resolver branch(es) da task

Esta skill é a responsável por **determinar** o nome e base da(s) branch(es), mas **não cria no git** — isso fica para o `implement`.

1. **Ler `GOD/patterns.md`** e extrair:
   - **Branch inicial** — uma única entrada (single-project comum) ou lista de entradas `{projeto, branch-base}` (multi-project ou single-project com múltiplos projetos listados)
   - **Padrão de nome de branch** — formato esperado (ex: `task/<cod-da-task>/<descrição-kebab-case>`)

2. **Resolução por modo:**

   **Modo single-project (a task se resolve em UMA branch):**
   - Se o `patterns.md` tem uma única entrada de branch inicial: usar direto como `branch_base`.
   - Se tem múltiplas entradas mas o contexto é single-project (o repo atual é um dos listados): identificar qual aplica usando o diretório raiz do repo (`git rev-parse --show-toplevel`) comparado com os nomes listados. Se ambíguo, perguntar ao usuário e memorizar a escolha apenas para esta execução.
   - Aplicar o padrão de nome pra compor o `branch` (ex: `task/PROJ-123/add-phone-field`). Se o padrão exige um slug e a descrição bruta/enriquecida não sugere um claro, perguntar ao usuário.

   **Modo multi-project (a task pode afetar N projetos):**
   - A identificação de **quais** projetos são afetados depende do escopo da task — provavelmente só fica claro após o passo 7 (plano escrito). Nesta etapa, apenas registrar que o modo é multi-project.
   - A resolução efetiva de nome/base por projeto acontece no **passo 8.5** (depois do plano estar escrito e os projetos afetados identificados).

3. **Idempotência** — se o `status.md` já tem `branch` e `branch_base` populados:
   - Confirmar com o usuário: "branch já resolvida como X (base: Y) — manter?"
   - Se manter, pular a re-resolução. Se trocar, refazer.

Guardar em memória o resultado da resolução (no caso single: `branch` string + `branch_base` string; no caso multi: marcador "multi, pendente de identificar projetos afetados").

### 3. Consultar knowledge por padrões técnicos

Ler `GOD/knowledge.md`. Buscar tasks anteriores com **estrutura técnica** semelhante (não escopo — escopo já foi tratado pela `spec`).

Se encontrar, guardar:
- **Códigos de commit** (hash1, hash2, ...) — para análise de referência no passo 4
- **Arquivos principais** — para prever impactos arquiteturais
- **Aprendizados técnicos** — para informar decisões de arquitetura no plano

### 4. Analisar commits de referência

Para cada commit identificado via knowledge (passo 3):
- Executar `git diff {hash}~1..{hash}`
- Analisar padrões, arquivos afetados, abordagens utilizadas
- Extrair lições aplicáveis (não copiar literalmente)

### 5. Coletar contexto técnico do projeto

Buscar e ler arquivos canônicos de **arquitetura/convenções de código** na raiz do projeto (se existirem):
- `CLAUDE.md` — convenções e instruções do projeto para Claude
- `AGENTS.md` — configurações e instruções de agentes
- `ARCHITECTURE.md` — arquitetura do projeto

A busca é case-insensitive. **Não ler `README.md` aqui** — `README.md` é contexto de produto e já foi consumido pela `spec`.

### 6. (Opcional) Q&A técnica com o usuário

Apenas se houver **ambiguidade técnica** real (não de escopo) que afete a estrutura do plano:
- Escolha entre 2+ abordagens arquiteturais viáveis
- Decisão de refatorar vs preservar padrão atual
- Trade-off de dependência (criar novo módulo vs estender existente)

Se a spec é clara e o caminho técnico é óbvio dado o knowledge + arquitetura do projeto, **anuncie que não há dúvidas técnicas** e siga.

**Q&A de escopo é proibida aqui.** Se o usuário levantar uma dúvida de escopo durante esta sessão, anotar e orientar a rodar `update-spec` (na v9) ou voltar pra `spec` reescrevendo.

### 7. Escrever plano de implementação

Preencher o arquivo `GOD/tasks/{cod-da-task}/plan.md` com o plano de implementação. A primeira seção do plano deve ser sempre **"Branch de trabalho"**. O conteúdo varia por modo:

**Modo single-project:**
```markdown
## Branch de trabalho

- **Base:** `{branch_base resolvido no passo 2.2}`
- **Branch da task:** `{branch resolvido no passo 2.2}`
- **Criação:** o `implement` criará a branch no início da execução partindo da base; esta skill apenas resolveu os nomes.

## Resumo
{...}
```

**Modo multi-project (resolução final acontece no passo 11.5, mas o bloco abaixo é escrito em placeholder aqui e depois atualizado no 11.5):**
```markdown
## Branch de trabalho

- **Modo:** multi-project workspace
- **Projetos afetados:** `{lista de diretórios identificados no plano}` (ex: `projeto-api`, `projeto-web`)
- **Branches a criar** (uma por projeto, nome idêntico):
  - `projeto-api`: base `main` → branch `task/PROJ-123/add-phone-field`
  - `projeto-web`: base `develop` → branch `task/PROJ-123/add-phone-field`
- **Criação:** o `implement` criará todas essas branches no início da execução. Esta skill apenas listou nomes e bases.

## Resumo
{...}
```

Seções restantes do plano (comuns a ambos os modos):
```markdown
## Spec de referência

- Caminho: `{spec_path}`
- REQs cobertos: REQ-001, REQ-002, ...
- ACs amarrados aos passos: ver "Passos de implementação"

## Arquivos afetados
{arquivos que serão criados ou modificados, agrupados por projeto quando multi-project}

## Passos de implementação
{passos ordenados, cada um com referência aos ACs que cobre, ex: "Passo 3: implementar validação de telefone (cobre AC-001.1, AC-001.2)"}

## Considerações técnicas
{decisões arquiteturais, padrões a seguir, trade-offs, baseadas no knowledge + ARCHITECTURE.md/CLAUDE.md}
```

A seção "Branch de trabalho" é **obrigatória** e deve vir no topo. O `implement` lê diretamente do `status.md`, mas o registro explícito no `plan.md` serve como instrução literal pro humano que for ler o plano e deixa a decisão auditável.

**Critérios de aceitação NÃO ficam no plano.** Eles vivem na `spec.md` (com IDs estáveis). O plano apenas **referencia** os ACs cobertos por cada passo. Se você sentir que falta um critério, é sinal de que a spec está incompleta — pause e rode `spec` novamente em vez de inventar critério no plano.

### 8. Revisão do plano

Após escrever o plano, chamar a sub-skill `review --plan` passando o código da task.

- Se o relatório retornar **Aprovado**: apresentar o plano ao usuário para validação final
- Se o relatório retornar **Ajustes necessários**: avaliar as correções sugeridas, aplicar as pertinentes no `plan.md` e apresentar o plano corrigido ao usuário
- Se o relatório retornar **Reprovado**: reescrever o plano com base no feedback e rodar a review novamente

### 8.5. Resolução final de branches em modo multi-project

Executar apenas se o modo detectado no passo 2.1 foi **multi-project**.

Neste ponto o plano já está escrito e a lista de **projetos afetados** está explícita nos "Arquivos afetados" / "Passos de implementação".

1. **Identificar projetos afetados** — extrair do plano a lista de diretórios de projeto que serão modificados (ex: `projeto-api`, `projeto-web`). Se houver ambiguidade, perguntar ao usuário antes de prosseguir.
2. **Ler `GOD/patterns.md`** seção "Branch inicial" — obter a branch-base de cada projeto listado.
3. **Para cada projeto afetado**, resolver:
   - `project`: nome do projeto (ex: `projeto-api`)
   - `base`: branch-base desse projeto (do patterns)
   - `name`: aplicar o padrão de nome ao código da task (normalmente o mesmo nome em todos os projetos, ex: `task/PROJ-123/add-phone-field`)
4. **Atualizar o bloco "Branch de trabalho" do `plan.md`** substituindo o placeholder pelos valores finais.

Esta etapa **não toca em git** — apenas registra os nomes planejados. A criação física de cada branch é feita pelo `implement` (ver `implement/SKILL.md` passo 2.05).

### 9. Atualizar status

Após a review ser aprovada (e o usuário validar o plano), atualizar `GOD/tasks/{cod-da-task}/status.md`:

- `phase`: `planned`
- `updated_at`: timestamp ISO 8601 em UTC
- `updated_by`: `plan`
- **Single-project:**
  - `branch`: **nome da branch resolvida no passo 2.2** (string, ex: `task/PROJ-123/add-phone-field`)
  - `branch_base`: **branch base** (string, ex: `main`)
- **Multi-project:**
  - `branch`: **lista de objetos**, um por projeto afetado:
    ```yaml
    branch:
      - project: projeto-api
        name: task/PROJ-123/add-phone-field
        base: main
      - project: projeto-web
        name: task/PROJ-123/add-phone-field
        base: develop
    ```
  - `branch_base`: `null` (redundante — cada entrada do `branch` já carrega sua base)
- `spec_version_consumed`: **`spec_version` da spec lida no passo 2** (int). Garante que o próximo `plan` ou `implement` saiba se a spec mudou depois.
- `learned`: preservar o valor atual
- `prs`: preservar o valor atual

### 10. Executar hook `after plan`

Ler `GOD/hooks.md` e localizar a seção `# after plan`.

- Se o conteúdo for `skip-hook`: encerrar.
- Se houver instruções em linguagem natural: executá-las integralmente antes de encerrar.

---

## Guard-rails

- **Esta skill não escreve em `GOD/knowledge.md`.** Apenas a skill `learn` pode fazê-lo. Aqui, o knowledge é apenas **lido** para encontrar tasks semelhantes (passo 3).
- **Esta skill não toca na spec.** A spec é o contrato de escopo, dona exclusiva da skill `spec`. Se o plano revelar que o escopo está incompleto ou contraditório, pause e oriente o usuário a re-rodar `spec` — não invente requisito no `plan.md`.
- **Esta skill não enriquece `description.md`.** O `init` cria o arquivo bruto; ele permanece bruto. Q&A de escopo, links, cenários estão na spec.
- **Esta skill é a dona da detecção single/multi-project e da resolução dos nomes/bases de branch.** O `init` deixa `branch: null`; o `plan` determina; o `implement` cria no git.
- **Esta skill não cria branch no git.** Nem em single nem em multi. Só resolve nomes/bases e persiste em `status.md` e `plan.md`. A criação física é responsabilidade exclusiva do `implement`.
