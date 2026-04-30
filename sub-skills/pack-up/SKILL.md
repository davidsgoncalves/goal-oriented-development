---
name: pack-up
description: |
  Executa o fluxo de finalização da task: commit, push, criação de PR e ações finais — tudo conforme definido no patterns.md. Processos customizados (testes, linter, etc.) ficam nos hooks `before pack-up`/`after pack-up`. Use quando o usuário mencionar: "pack up", "finalizar task", "fechar task", "empacotar", ou quando a fase de finalização for ativada pelo GOD.
tools: Read, Glob, Grep, Bash, Edit, Write, Agent
---

# Pack-up — Sub-skill de Finalização

> Executa o fluxo de finalização da task: commit, push, criação de PR e ações finais — tudo conforme definido no `patterns.md`. Processos customizados (rodar testes, linter, notificações, etc.) são configurados pelo usuário nos hooks `before pack-up`/`after pack-up` em `GOD/hooks.md`.

## Instruções

Quando o usuário invocar esta skill, execute os seguintes passos **na ordem**:

### 0. Executar hook `before pack-up`

Ler `GOD/hooks.md` e localizar a seção `# before pack-up`.

- Se o conteúdo for `skip-hook`: pular e seguir para o passo 1.
- Se houver instruções em linguagem natural: executá-las integralmente antes de prosseguir. Exemplos comuns são rodar testes, linter, type-check.
- **Se o hook falhar** (ex: testes vermelhos, lint com erro):
  - Informar o usuário sobre a falha
  - Tentar corrigir automaticamente se for algo simples (ex: auto-fix de lint)
  - Se não conseguir corrigir, pausar o pack-up e perguntar ao usuário como prosseguir

### 1. Ler convenções do projeto

Ler o arquivo `GOD/patterns.md` para obter as convenções de formatação:
- Padrão de mensagem de commit
- Padrão de mensagem de PR (título e corpo)

A **branch de destino do PR** (base target) e o **nome da branch da task** não são lidos de `patterns.md` nesta skill — já estão persistidos em `GOD/tasks/{cod-da-task}/status.md` (`branch_base` e `branch`, respectivamente, populados pelo `plan`).

**Multi-project workspace:** se `status.branch` for uma **lista** `[{project, name, base}, ...]` em vez de string, a task afeta múltiplos projetos. Nesse caso, a pack-up itera pelos projetos: para cada entrada, faz `cd` no projeto, commita e abre PR individualmente com a base correspondente. Cada PR é adicionado ao array `prs` do `status.md`.

> Observação: `patterns.md` contém **apenas padrões**. Ações executáveis (criar PR em draft, não atribuir reviewers, adicionar labels, notificar canais, atualizar tickets) ficam no hook `after pack-up` em `GOD/hooks.md`.

### 2. Identificar a task

- Se o contexto da conversa já contém o código da task, usar esse código
- Caso contrário, perguntar ao usuário o código da task (ex: `PROJ-123`)
- Ler `GOD/tasks/{cod-da-task}/description.md` para obter título e descrição da task

### 3. Verificar estado do git

Verificar o estado atual do repositório:
- Branch atual (`git branch --show-current`)
- Alterações staged e unstaged (`git status`, `git diff`)
- Confirmar que existe trabalho para commitar

Se não houver alterações, informar o usuário e encerrar.

### 4. Revisão plano vs execução

Antes de commitar, chamar a sub-skill `review --execution` passando o código da task. A partir da v8, esse review já inclui o eixo de cobertura de ACs.

- Se o relatório retornar **Aprovado**: prosseguir com o commit
- Se o relatório retornar **Ajustes necessários**: apresentar o relatório ao usuário e perguntar se deseja corrigir antes de continuar ou prosseguir mesmo assim
- Se o relatório retornar **Reprovado**: apresentar o relatório ao usuário e pausar o pack-up até que as correções sejam feitas

### 4.5. Gerar matriz de cobertura (a partir da v8)

Chamar a sub-skill `coverage --task {cod} --format markdown` pra gerar a tabela AC × validação que vai entrar no PR description.

- Capturar a saída em markdown — será usada no passo 7 (criar PR).
- Se a saída indicar **ACs órfãos**, exibir aviso ao usuário no terminal:

  > ⚠️ {N} ACs estão sem cobertura registrada (nem teste, nem validação manual). Eles aparecerão como ⚠️ na tabela do PR. Quer (a) voltar pro implement pra anotar, (b) editar `coverage.md` agora, (c) prosseguir mesmo assim?

- Tasks **sem spec** (raras na v8 — só tasks pré-v6 que sobreviveram): `coverage` retorna "não aplicável", e este passo vira no-op (nenhuma tabela injetada).

### 5. Commit

Criar o commit seguindo o **padrão de mensagem de commit** definido no `patterns.md`:
- Fazer stage das alterações relevantes (`git add` dos arquivos modificados)
- Criar o commit com a mensagem no padrão configurado
- Se o padrão não estiver definido, usar: `{cod-da-task}: {descrição breve da mudança}`

### 6. Push

Fazer push do branch para o remote:
- `git push -u origin {branch-atual}`
- Se o push falhar, informar o usuário

### 7. Criar PR

Criar o Pull Request seguindo o **padrão de mensagem de PR** definido no `patterns.md`:
- Usar `gh pr create` com título e corpo no padrão configurado, apontando para a **branch de destino** = `branch_base` lido de `status.md` (ex: `--base main`). Se `branch_base` estiver ausente (task pré-v4 que não passou pelo novo plan), fallback para o default do repo detectado por `gh`.
- Se o padrão não estiver definido, usar:
  - **Título:** `{cod-da-task}: {título da task}`
  - **Corpo:** descrição da task + resumo das alterações
- **Anexar referência à spec** (a partir da v6): se `spec_path` estiver presente em `status.md`, adicionar ao final do corpo do PR um bloco:

  ```markdown
  ---

  📐 **Spec:** `{spec_path}` (v{spec_version})

  REQs cobertos: {lista lida do plan.md ou da spec}
  ```

  Esse bloco existe pra que quem revisa o PR saiba qual contrato a implementação cumpre — sem precisar abrir 3 arquivos. Se a spec usa path relativo, manter como está (mesmo path que o GOD usa). Em multi-project, o bloco de referência é repetido em cada PR (a spec é a mesma; o que muda é o repo).

- **Anexar matriz de cobertura** (a partir da v8): logo após o bloco da spec, injetar a saída markdown da `coverage` capturada no passo 4.5:

  ```markdown
  📊 **Cobertura de ACs**

  | AC | Status | Validação |
  |----|--------|-----------|
  | AC-001.1 | ✅ | tests/phone-validator.test.ts:42 |
  | AC-001.2 | ✅ | tests/phone-validator.test.ts:55 |
  | AC-001.3 | 👁 | manual: validação visual em staging por PM |
  | AC-002.1 | ⚠️ | (sem cobertura) |

  **Resumo:** 4 ACs · 2 testes · 1 manual · 1 órfão
  ```

  Tasks sem spec não geram este bloco.
- Capturar a URL do PR retornada por `gh pr create` — será salva no `status.md` no próximo passo

### 8. Atualizar status para `packed-up`

Atualizar `GOD/tasks/{cod-da-task}/status.md`:

- `phase`: `packed-up`
- `updated_at`: timestamp ISO 8601 em UTC
- `updated_by`: `pack-up`
- `branch`, `branch_base`: manter os valores atuais
- `learned`: **preservar** o valor atual (não alterar — `learn` é quem controla esse campo)
- `prs`: **fazer append** da URL do PR criado neste pack-up ao array existente. Não sobrescrever — se a task já tinha PRs de pack-ups anteriores (raro, mas possível em tasks com múltiplos repositórios), manter os anteriores

Exemplo antes:
```yaml
prs: []
```
Após o primeiro pack-up:
```yaml
prs:
  - https://github.com/org/vakinha-api/pull/123
```
Após um segundo pack-up no mesmo cod-da-task (outro projeto no mesmo monorepo):
```yaml
prs:
  - https://github.com/org/vakinha-api/pull/123
  - https://github.com/org/vakinha-web/pull/456
```

### 9. Executar hook `after pack-up`

Ler `GOD/hooks.md` e localizar a seção `# after pack-up`.

- Se o conteúdo for `skip-hook`: pular e seguir para o passo 10.
- Se houver instruções em linguagem natural: executá-las integralmente antes do relatório final.

Este hook é o lugar para todas as ações pós-PR: marcar o PR como draft, remover reviewers, adicionar labels, notificar canais do Slack, atualizar status no Jira, postar update num board, etc.

### 10. Reportar resultado

> ✅ Pack-up concluído para `{cod-da-task}`!
>
> 📝 Commit: `{hash}` — {mensagem do commit}
> 🔀 Branch: `{nome-do-branch}`
> 🔗 PR: {url-do-pr}
>
> [Se hooks before/after foram executados, listar resumidamente o que rodou]

---

## Guard-rails

- **Esta skill não escreve em `GOD/knowledge.md`.** O registro no knowledge é responsabilidade exclusiva da skill `learn`, invocada pelo usuário após o pack-up.
