---
name: init-tree
description: |
  Inicializa em lote um conjunto de tasks a partir de uma árvore do Jira (tipicamente Epic → Stories → Subtasks). Desce recursivamente a árvore a partir de um nó-raiz passado pelo usuário, cria pastas de contexto para nós internos em `GOD/tasks/` e gera specs em batch para folhas em `<specs_path>/tasks/{cod}.md`. Não cria estrutura de execução (pasta GOD/tasks/ por folha) — isso fica pro `init` rodar depois manualmente, após aprovação das specs. Não toca no git. Use quando o usuário mencionar: "init em lote", "init tree", "iniciar Epic", "iniciar várias tasks", "subtasks do Jira", ou passar um link/código de Epic/Story com subtasks.
tools: Read, Glob, Grep, Bash, Edit, Write, Agent
---

# Init-Tree — Sub-skill de Inicialização em Lote via Árvore do Jira (v9)

> Inicializa em lote as tasks de uma árvore do Jira. Recebe um nó-raiz (Epic, Story ou Task com subtasks), desce recursivamente, filtra por status, confirma com o usuário e produz: (a) pastas de contexto pra nós internos em `GOD/tasks/`; (b) specs por folha em `<specs_path>/tasks/{cod}.md`. **Não cria estrutura de execução nem roda `init` por folha** — isso fica explicitamente pro usuário rodar depois, depois que as specs forem revisadas/aprovadas. Esse é o gate da inversão v9.

> **Mudança v9 (spec-first em batch):** antes (v8), init-tree delegava ao `init` que criava `description.md` em cada folha. Agora delega ao `spec` em modo batch — o que cada folha recebe é uma spec rascunho gerada a partir do Jira, sem Q&A interativa. O usuário refina cada spec individualmente (`spec {cod}` interativo) e roda `init {cod}` quando aprovada.

## Banner

Ao iniciar esta skill, **antes de qualquer outra ação**, exiba exatamente este bloco no terminal:

```
  ██████   ██████  ██████  
 ██       ██    ██ ██   ██ 
 ██   ███ ██    ██ ██   ██ 
 ██    ██ ██    ██ ██   ██ 
  ██████   ██████  ██████  
  Goal Oriented Development
```

## Pré-requisitos

- MCP Atlassian disponível e autenticado (`getJiraIssue`, `searchJiraIssuesUsingJql`). Sem isso, a skill encerra com orientação para conectar.
- `GOD/` existe na versão atual (a orquestradora já garante isso antes de delegar).
- `GOD/config.md` com `specs_path` resolvível (ou usar default `docs/specs/`).

## Flags

- `--refresh` — re-lê o Jira e re-roda `spec --batch` em todas as folhas já existentes (regenera spec rascunho a partir dos dados atuais do Jira). Sem essa flag, specs existentes são preservadas intactas.
- `--skip-spec` — apenas cria pastas de contexto; não gera specs por folha. Útil se você já produziu as specs por outro caminho ou só quer mapear a árvore documentalmente.

## Instruções

Quando o usuário invocar esta skill, execute os passos **na ordem**:

### 1. Receber input da raiz

O usuário deve fornecer **uma** das opções:

- **Link do Jira do nó-raiz** — ex: `https://empresa.atlassian.net/browse/PROJ-100`
- **Código do nó-raiz** — ex: `PROJ-100`

Se o usuário não passou nada, perguntar: "Qual o código ou link do nó-raiz da árvore (Epic, Story ou Task com subtasks)?"

Extrair o código (ex: `PROJ-100`) do input.

### 2. Buscar a árvore completa no Jira (recursivo)

Começando pelo nó-raiz, montar a árvore de issues descendo por todos os níveis:

1. `getJiraIssue(PROJ-100)` — obter título, descrição, tipo (Epic/Story/Task/Subtask), status, e lista de **filhos imediatos** (no Jira isso pode aparecer como `subtasks`, `issuelinks` do tipo "is parent of", ou filhos de Epic via JQL `"Epic Link" = PROJ-100`).
2. Para cada filho encontrado, repetir o fetch recursivamente até que um nó não tenha mais filhos (folha).
3. Montar a estrutura em memória:
   ```
   {
     code: "PROJ-100",
     type: "Epic",
     title: "...",
     status: "In Progress",
     children: [
       {
         code: "PROJ-101",
         type: "Story",
         ...
         children: [
           { code: "PROJ-103", type: "Subtask", children: [] },
           ...
         ]
       },
       ...
     ]
   }
   ```

**Observações:**
- O tipo de hierarquia no Jira varia por projeto. Usar relacionamento parent/subtask, não assumir tipos.
- Se fetch falhar pra algum nó (permissão, erro de API), **não abortar**: marcar o nó como "fetch falhou" e seguir.

### 3. Filtrar por status

1. **Ler filtro de status do `patterns.md`** — seção opcional `## Status Jira a ignorar em batch`. Default:
   ```
   Done, Cancelled, Closed, Resolved, Won't Do
   ```

2. **Aplicar o filtro apenas às folhas.** Contextos (nós internos) nunca são filtrados.

3. Marcar cada nó:
   - `will_create: true` se é contexto OU folha que passou no filtro
   - `will_create: false` se é folha com status filtrado

### 4. Detectar duplicatas (idempotência)

Para cada nó com `will_create: true`:
- **Contexto** — checar `GOD/tasks/{cod}/`. Se existe, marcar `existing: true`.
- **Folha** — checar `<specs_path>/tasks/{cod}.md`. Se existe, marcar `existing: true`.
- Se a flag `--refresh` está ativa, marcar `will_refresh: true` em ambos os casos.
- Se a pasta/spec não existe: marcar `new: true`.

### 5. Mostrar preview + confirmação

Apresentar visualização da árvore:

```
🌲 init-tree PROJ-100

Árvore detectada no Jira:
  PROJ-100 "Epic: Redesign onboarding" (Epic)                [context, novo]
    PROJ-101 "Story: Tela de boas-vindas" (Story)            [context, novo]
      PROJ-103 "Implementar header" (Subtask, In Progress)   [spec, novo]
      PROJ-104 "Implementar CTA" (Subtask, Backlog)          [spec, novo]
      PROJ-105 "A/B test botão" (Subtask, Done)              [pulado por filtro]
    PROJ-102 "Story: Fluxo de senha" (Story)                 [context, existente — preservado]
      PROJ-106 "Validação no front" (Subtask, To Do)         [spec, existente — preservado]
      PROJ-107 "Endpoint /reset" (Subtask, Cancelled)        [pulado por filtro]

Resumo:
  - Contextos novos:  3
  - Specs novas:      2 (geradas em batch a partir do Jira, sem Q&A interativa)
  - Existentes:       1 contexto + 1 spec (preservados)
  - Pulados:          2 folhas (filtro de status)

⚠️  Importante (v9 spec-first):
  - Esta skill NÃO cria estrutura de execução por folha (GOD/tasks/{cod}/plan.md, status.md).
  - Você refina cada spec depois com `spec {cod}` (Q&A interativa) e roda `init {cod}` quando aprovada.

Confirmar? (sim / não)
```

- **Sim** → prosseguir.
- **Não** → encerrar sem criar nada.

### 6. Criar estruturas

Percorrer a árvore em **pré-ordem** (pai antes dos filhos).

**Para nó interno (contexto):**

Criar pasta `GOD/tasks/{cod-do-no}/` contendo apenas `description.md` (não tem `plan.md` nem `status.md` — contextos não passam por execução):

```markdown
---
kind: context
jira_type: {Epic|Story|Task}
parent: {cod-do-pai ou null se raiz}
children: [{cod-filho-1}, {cod-filho-2}, ...]
---

# {cod-do-no} — {título do Jira}

## Descrição

{descrição completa do Jira}

## Metadados do Jira

- **Tipo:** {Epic|Story|Task}
- **Status:** {status atual no Jira}
- **Link:** {url do Jira}

## Filhos diretos

{lista dos filhos diretos, com código e título de cada um}

---

> Esta é uma pasta de contexto (não uma task real). Não há `plan.md` nem `status.md` aqui. As tasks reais (folhas) têm spec em `<specs_path>/tasks/` e estrutura de execução em `GOD/tasks/{cod}/` quando o usuário roda `init {cod}` após aprovar a spec.
```

**Para folha (task real):**

Se a flag `--skip-spec` está ativa, pular a folha (apenas registrar no relatório).

Caso contrário, **delegar à skill `spec` em modo batch**:

- Invocar `spec` programaticamente passando:
  - `code`: código do Jira da folha
  - `mode`: `batch` (sinaliza modo não-interativo)
  - `parent`: código do pai imediato (registrar no contexto)
  - `link`: URL do Jira

- A skill `spec` em modo batch deve:
  - Buscar dados do Jira (passo 2 do spec)
  - Pular Q&A interativa (passo 6 do spec)
  - Aplicar perfil heurístico SEM confirmação interativa (default `normal`)
  - Escrever spec em `<specs_path>/tasks/{cod}.md` com frontmatter:
    ```yaml
    spec_version: 1
    task: {cod}
    profile: {normal|critical}  # heurística sem confirmação
    draft: true                  # sinaliza spec rascunho — pendente de Q&A interativa
    created_at: ...
    updated_at: ...
    ```
  - Não rodar `review --spec` (será rodado quando o usuário refinar com `spec {cod}` interativo)
  - Não rodar hook `after spec` (evita publicação prematura de rascunho)
  - Retornar silenciosamente

> **Comportamento idempotente:** se a spec já existe e `--refresh` não está ativo, init-tree pula essa folha (nem chama `spec`). Se `--refresh` está ativo, re-chama `spec --batch` que sobrescreve a spec existente preservando a flag `draft` se a spec original ainda for rascunho.

**Para `will_refresh: true` (flag `--refresh`):**
- **Contextos:** reescrever `description.md` com dados atualizados do Jira (preservar ordem dos campos; `children` reflete a árvore atual).
- **Specs (folhas):** re-chamar `spec --batch` que regenera o conteúdo. Se a spec existente já tinha `draft: false` (foi refinada via `spec` interativo), **não sobrescrever** — apenas atualizar a seção `## Input bruto` da spec com o novo texto do Jira e adicionar nota:

  ```markdown
  > Atualizado do Jira em {timestamp} por init-tree --refresh. Estrutura preservada (spec já refinada).
  ```

**Falha parcial:** se a criação de algum nó falhar, registrar erro, continuar. Persistir o que foi criado.

### 7. Reportar resultado

```
✅ init-tree PROJ-100 concluído!

Criadas:
  📁 3 pastas de contexto (Epic + 2 Stories)
  📐 2 specs rascunho (PROJ-103, PROJ-104) em <specs_path>/tasks/

Preservadas (já existiam):
  📁 PROJ-102 (contexto)
  📐 PROJ-106 (spec, draft: false — refinada anteriormente)

Puladas (filtro de status):
  PROJ-105 (Done), PROJ-107 (Cancelled)

Falhas (se houver):
  ⚠️ PROJ-XXX — {motivo}

💡 Próximos passos (v9):
  1. Refine as specs rascunho rodando `spec {cod}` por folha — abre Q&A interativa, classifica perfil, valida com `review --spec`. As novas: PROJ-103, PROJ-104.
  2. Quando a spec estiver aprovada (e publicada via `publish-spec` se for crítica), rode `init {cod}` pra criar estrutura de execução.
  3. Pastas de contexto (PROJ-100, PROJ-101, PROJ-102) ficam só como referência documental — não passam por plan/implement.
  4. Pra atualizar a partir do Jira, rode `init-tree PROJ-100 --refresh`.
```

---

## Guard-rails

- **Esta skill não toca no git.** Nem em folhas nem em contextos.
- **Esta skill não escreve em `GOD/knowledge.md`.** Apenas a skill `learn`.
- **Esta skill não chama `init`/`plan`/`implement` automaticamente.** Downstream é sempre manual, uma folha por vez. Esse é o gate da inversão v9: spec primeiro, execução só depois de aprovação humana.
- **Esta skill não roda Q&A interativa nas specs em batch.** Specs nascem como `draft: true` e o usuário refina depois. Tentar fazer Q&A em batch pra 50 folhas vira teatro — usuário não vai responder com qualidade pra cada uma.
- **Esta skill não publica specs (não roda hook `after spec`).** Rascunho não é pra ser publicado.
- **Esta skill não cria `GOD/tasks/{cod}/plan.md` ou `status.md` pra folhas.** Estrutura de execução nasce com `init {cod}`, nunca aqui.
- **Esta skill não apaga pastas existentes.** Sem `--refresh`, existentes preservadas intactas. Com `--refresh`, descrições/specs são atualizadas mas estrutura de execução (se já criada via `init`) nunca é tocada.
- **Esta skill não assume nomes de status Jira.** Usa lista configurável em `patterns.md` ou default documentado.
- **Esta skill não aborta em falha parcial.** Continua criando o que der, reporta no final.
