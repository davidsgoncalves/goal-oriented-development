# Migração v6 → v7

## Objetivo

A v7 ativa **propagação ativa da spec** sem mudar a arquitetura de pastas estabelecida na v6. Foco: ferramental pra que a spec deixe de ser local-only e ganhe ciência ativa fora do GOD (Jira/Slack), com review semântico mais profundo e detecção de drift.

Mudanças estruturais:

- **Hooks `before spec` e `after spec`** — adicionados ao `GOD/hooks.md` (default `skip-hook`). Permitem publicar a spec automaticamente em canal externo após ser criada/atualizada.
- **`review --spec` profundo** — além das verificações de v6 (estrutura, lint), agora roda verificações semânticas: cobertura description→spec profunda, cenários cobrem ACs, NFRs adequados ao tipo de feature, ortogonalidade dos ACs, edge cases óbvios. Flag `--quick` mantém só lint.
- **Frontmatter da spec ganha campos opcionais** — `last_published_at` e `published_to` rastreiam ciência ativa. `feedback_cycles` (incrementado por `--review-feedback`) limita ping-pong.
- **`status.md` ganha `spec_version_consumed`** — rastreia qual versão da spec foi consumida pelo último plan/implement. Permite detecção de drift.
- **Modo `spec --review-feedback`** — loop opt-in pra incorporar feedback do stakeholder antes do `plan` rodar. Incrementa `spec_version`. Limite de 2 ciclos antes de exigir conversa síncrona.
- **Sub-skill nova `publish-spec`** — publicação manual de spec em targets configuráveis. Útil pra reposting ou compartilhamento ad-hoc.
- **Freshness check em `plan` e `implement`** — comparam `spec_version` atual vs `spec_version_consumed` em `status.md`. Se diferentes, alertam o usuário antes de prosseguir.

## Arquivos afetados

**Atualizado:**
- `GOD/VERSION` — conteúdo muda de `v6` para `v7`
- `GOD/hooks.md` — adicionados slots `# before spec` e `# after spec` (com default `skip-hook`)

**Inalterados (nenhuma reescrita automática):**
- `GOD/config.md` (specs_path continua igual)
- `GOD/knowledge.md`
- `GOD/patterns.md`
- `GOD/learned-patterns.md`
- `GOD/tasks/` (todas as tasks, `status.md`, `description.md`, `plan.md`, `changelog.md`)
- `GOD/tasks/.archived/` (se existir)
- `<specs_path>/` (especs existentes não são alteradas — campos novos são opcionais)

**Mudanças de contrato (não de arquivos):**
- `spec` executa hooks `before spec` e `after spec`. Após `after spec` rodar, atualiza `last_published_at` e `published_to` no frontmatter da spec.
- `spec --review-feedback` (modo novo) incorpora feedback externo, incrementa `spec_version` e `feedback_cycles`.
- `spec --quick` repassa pra `review --spec --quick` (lint sem semântica).
- `review` ganha verificações semânticas profundas no modo `--spec` (skipáveis com `--quick`).
- `status.md` ganha campo `spec_version_consumed`. Atualizado por `plan` (passo 9) e `implement` (passo 7) com a versão da spec lida.
- `plan` (passo 2) e `implement` (passo 2) checam freshness antes de prosseguir.
- Sub-skill nova `publish-spec` disponível pra publicação manual.
- `pack-up` continua igual (linka spec no PR — não muda).

## Riscos conhecidos

- **Specs criadas em v6 não têm `last_published_at`/`published_to`/`feedback_cycles`.** OK — campos são opcionais. Se nunca foram publicadas via hook `after spec`, ficam ausentes naturalmente.
- **Tasks v6 com `status.md` sem `spec_version_consumed`.** Tratado: o freshness check do plan/implement aceita `null` (ou ausente) como "primeira leitura" e prossegue normalmente. A versão é registrada na próxima execução.
- **Tasks ativas em `phase: planned` ou posterior continuam compatíveis.** As verificações novas (freshness check) só disparam quando há divergência real entre versões. Se a task v6 nunca foi atualizada, não há divergência.
- **Hook `after spec` com instruções defeituosas pode falhar silenciosamente.** Mitigação: `spec` deve reportar erro do hook no relatório final, não só "concluído".
- **Modo `--review-feedback` pode virar ping-pong se mal usado.** Mitigação built-in: contador `feedback_cycles` alerta na 3ª iteração consecutiva.

---

## Passos da migração

### 1. Verificar pré-condições

- Confirmar que `GOD/VERSION` existe e contém `v6`.
- Se não existir ou estiver em versão anterior, abortar — o usuário deve rodar as migrações anteriores primeiro.

### 2. Adicionar slots de hook no `GOD/hooks.md`

Ler `GOD/hooks.md`. Localizar a seção `# after init`. **Inserir imediatamente após o bloco `after init`** (e antes do `before plan`) os dois novos slots:

```markdown

# before spec
skip-hook

# after spec
skip-hook

```

Resultado esperado da seção (após inserção):

```markdown
# before init
skip-hook

# after init
skip-hook

# before spec
skip-hook

# after spec
skip-hook

# before plan
skip-hook

...
```

**Não modificar slots existentes** — `before init`, `after init`, `before plan`, `after plan`, `before implement`, `after implement`, `before pack-up`, `after pack-up` permanecem como o usuário os deixou.

### 3. Atualizar nota de cabeçalho do `hooks.md`

Se a nota inicial do `hooks.md` ainda diz "Apenas os steps do fluxo principal têm hooks: `init`, `plan`, `implement`, `pack-up`", atualizar pra incluir `spec`:

> Apenas os steps do fluxo principal têm hooks: `init`, `spec`, `plan`, `implement`, `pack-up`.

E atualizar a lista de "Ferramentas auxiliares (sem hooks)" pra incluir `publish-spec`:

> Ferramentas auxiliares (`learn`, `update-plan`, `review`, `status`, `publish-spec`) não têm hooks.

Se o usuário customizou essa nota, **não sobrescrever** — apenas alertar no relatório final que a nota está desatualizada e sugerir revisar manualmente.

### 4. Atualizar VERSION

Sobrescrever o conteúdo de `GOD/VERSION` com:

```
v7
```

### 5. Não tocar em tasks ativas

A migração **deliberadamente não modifica** `GOD/tasks/{cod}/status.md` nem `<specs_path>/tasks/<cod>.md` de nenhuma task existente. Os campos novos (`spec_version_consumed`, `last_published_at`, `published_to`, `feedback_cycles`) são opcionais e populados conforme as skills v7 forem rodando em novas execuções.

### 6. Relatório final

```
✅ Migração v6 → v7 concluída!

VERSION atualizado: v7

Arquivo atualizado:
  - GOD/hooks.md (slots `# before spec` e `# after spec` adicionados, default skip-hook)

Novidades disponíveis:
  📡 hooks before/after spec       — propagação automática da spec após criação/atualização
  🔍 review --spec profundo        — verificações semânticas (cobertura, cenários, NFRs, ortogonalidade, edge cases)
  ⚡ review --spec --quick         — pula semânticas (mantém lint)
  🔄 spec --review-feedback        — incorpora feedback do stakeholder, incrementa spec_version
  📤 publish-spec (skill nova)     — publicação manual em targets configuráveis (jira, slack, stdout, custom)
  🚦 freshness check               — plan/implement detectam quando spec mudou desde a última execução
  📋 status.md ganha spec_version_consumed
  📐 spec frontmatter ganha last_published_at, published_to, feedback_cycles

Tasks já em andamento continuam compatíveis. Campos novos são opcionais e populados nas próximas execuções.

Próximos passos sugeridos (opcionais):
  - Editar GOD/hooks.md preenchendo `after spec` se quiser publicação automática (exemplos no template do install)
  - Rodar `publish-spec PROJ-XXX --target jira --dry-run` numa task existente pra testar formatação
```
