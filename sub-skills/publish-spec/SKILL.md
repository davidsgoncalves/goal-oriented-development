---
name: publish-spec
description: |
  Publica/republica a spec de uma task em canais externos (Jira, Slack, stdout). Útil pra reposting manual, compartilhar com novo público, ou publicar quando o hook `after spec` estava em `skip-hook` no momento da criação. Não modifica a spec — apenas formata e dispara a publicação. Use quando o usuário mencionar: "publicar spec", "republicar spec", "compartilhar spec", "publish spec".
tools: Read, Glob, Grep, Bash, Edit, Write
---

# Publish-spec — Sub-skill de Publicação Manual da Spec

> Publica a spec de uma task em destinos externos (Jira, Slack, stdout, ou outros configurados). Não modifica a spec — apenas formata e dispara. Funciona como complemento manual ao hook automático `after spec`.

## Quando usar

- O hook `after spec` estava como `skip-hook` quando a spec foi criada e agora você quer publicar
- A spec precisa ser **republicada** depois de uma atualização
- Quer compartilhar com público adicional não previsto no hook (ex: hook publica no Jira, mas você quer postar no Slack também desta vez)
- Quer só **gerar o conteúdo formatado** sem publicar (modo `--target stdout`)

## Flags

- `--target <destino>` — destino da publicação. Valores: `jira`, `slack`, `stdout`, `notion`, ou qualquer string que o hook reconheça. Pode ser repetida (ex: `--target jira --target slack`). **Default:** lê de `GOD/config.md` seção `## publish_spec_default_target`; se ausente ou vazia, usa `stdout`.
- `--dry-run` — gera o conteúdo formatado e mostra ao usuário, sem disparar nenhuma publicação. Útil pra revisar antes.

## Default configurável no GOD/config.md

Pra evitar passar `--target` em toda execução (ex: você sempre publica no Jira), defina o default em `GOD/config.md`:

```markdown
## publish_spec_default_target

jira
```

A skill lê esse valor quando `--target` não é passado. Aceita um único target ou múltiplos separados por vírgula:

```markdown
## publish_spec_default_target

jira, slack
```

Equivale a chamar `publish-spec --target jira --target slack`.

### Ordem de precedência

1. Flag `--target` na invocação (override pontual; pode ser repetida)
2. `GOD/config.md` → `## publish_spec_default_target`
3. Sem default → `stdout` (comportamento original)

### Quando o default é ignorado

- Target resolvido (default ou flag) é **inválido** — avisar o usuário e cair pra `stdout`.
- Target requer integração indisponível (ex: `jira` mas MCP Atlassian fora) — avisar e cair pra `stdout` apenas pra esse target específico (outros targets resolvidos seguem normalmente).

## Instruções

### 0. Identificar a task

- Se o contexto da conversa já contém o código da task, usar esse código
- Caso contrário, perguntar ao usuário o código (ex: `PROJ-123`)

### 1. Verificar pré-requisitos

Ler `GOD/tasks/{cod-da-task}/status.md` e extrair `spec_path`.

- Se `spec_path` é `null` ou ausente: a task não tem spec. Orientar a rodar `spec` primeiro e encerrar.
- Se o arquivo no `spec_path` não existe: avisar e pedir confirmação.

### 1.5. Resolver target(s)

Determinar a lista de targets a usar nesta execução, na seguinte ordem:

1. **Se `--target` foi passado** (uma ou mais vezes): usar esses valores. Pular config.
2. **Senão**, ler `GOD/config.md` e procurar seção `## publish_spec_default_target`:
   - Se a seção existe e tem valor não-vazio: parsear (split por vírgula, trim espaços) e usar.
   - Se a seção existe mas está vazia, ou não existe: cair pro próximo passo.
3. **Senão**, usar `stdout` (default original retrocompatível).

Validar cada target resolvido:

- Targets conhecidos (`jira`, `slack`, `stdout`, `notion`): aceitar direto.
- Targets desconhecidos: procurar em `GOD/hooks.md` por seção `# publish-spec target: <nome>`. Se existe, aceitar. Se não, avisar o usuário ("target `<nome>` desconhecido — defina em `hooks.md` ou use um conhecido") e remover da lista.
- Se sobrar a lista vazia após validação, cair pra `stdout` e avisar.

Reportar a fonte do target ao usuário na primeira saída do comando: `Target(s): [jira, slack] (via GOD/config.md)` ou `Target: stdout (default)`.

### 2. Ler a spec

Ler o conteúdo de `<spec_path>` e extrair:
- `spec_version` do frontmatter
- Lista de REQs (com nomes/descrições curtas)
- Contagem total de ACs
- Cenários (tipos: happy/edge/erro)
- NFRs declarados (não vazios)
- Referências (Jira/Figma)

### 3. Formatar conteúdo por destino

Pra cada `--target`, formatar adequadamente. Templates default abaixo (cada hook customizado pode override):

**`stdout`** (markdown puro):

```markdown
📐 SPEC-{cod-da-task} — {título} (v{spec_version})

## Resumo
- {N} requisitos, {M} critérios de aceitação
- Cenários: {qtd happy/edge/erro}
- NFRs declarados: {lista de NFRs não vazios}

## Requisitos
- REQ-001: {nome}
- REQ-002: {nome}
- ...

📂 Spec completa: <spec_path>
🔗 Jira: {url ou —}
🎨 Figma: {url ou —}

> Se algum critério não bate com o esperado, comente antes do dev começar.
```

**`jira`** (markdown adaptado pro Jira):

```
*Spec atualizada* (SPEC-{cod-da-task} v{spec_version})

{N} requisitos, {M} ACs · cenários: {qtd}

*Requisitos:*
- REQ-001: {nome}
- REQ-002: {nome}

*NFRs declarados:* {lista}

Spec completa: [link pra repo]
Se algum critério não bate, responda neste comentário antes da implementação começar.
```

Postar via MCP Atlassian (`addCommentToJiraIssue`) no ticket correspondente.

**`slack`** (markdown Slack-flavored):

```
:scroll: *Spec disponível pra revisão* (PROJ-123 v{spec_version})

>{N} REQs, {M} ACs

Resumo:
• REQ-001: {nome}
• REQ-002: {nome}

NFRs: {lista}

Link: {spec_path ou URL}

Stakeholders: PM/UX/QA — chequem antes do dev começar.
```

Postar no canal configurado (depende de integração disponível).

**Outros targets** — se o `--target` não for um dos default acima, procurar em `GOD/hooks.md` por seção `# publish-spec target: <nome>` com instruções customizadas. Se não existir, abortar com erro: "Target `{nome}` não conhecido. Defina em `hooks.md` como `# publish-spec target: {nome}` com instruções, ou use um target conhecido (jira, slack, stdout)."

### 4. Modo `--dry-run`

Se a flag estiver presente, mostrar o conteúdo formatado pra cada target solicitado e perguntar:

> Conteúdo gerado pra `{target}`:
>
> ```
> {conteúdo}
> ```
>
> Publicar? (s/n)

Se `n`: encerrar sem publicar. Se `s`: prosseguir com a publicação real.

### 5. Disparar publicação

Pra cada target (exceto `stdout` e `--dry-run`):
- Executar a publicação (chamada MCP, comando shell, etc.)
- Capturar resultado (sucesso/falha + URL retornada se houver)

### 6. Atualizar frontmatter da spec

Após publicação bem-sucedida em pelo menos um target real (não `stdout`), atualizar o frontmatter de `<spec_path>`:

```yaml
last_published_at: {timestamp-iso-8601-utc}
published_to: [<targets publicados com sucesso, mesclados com publicações anteriores se houver>]
```

`published_to` deve ser um set (sem duplicatas) — se já tinha `[jira]` e agora publicou em `slack`, vira `[jira, slack]`.

### 7. Reportar resultado

> ✅ Spec publicada!
>
> 📡 Targets:
>   - jira: ✓ ({URL do comentário})
>   - slack: ✓ ({permalink})
>   - stdout: ✓
>   - notion: ✗ ({motivo da falha})
>
> 📐 Frontmatter atualizado: last_published_at, published_to: [jira, slack]

---

## Guard-rails

- **Esta skill não modifica o conteúdo da spec.** Apenas atualiza frontmatter (`last_published_at`, `published_to`).
- **Esta skill não escreve em `GOD/knowledge.md` nem em `GOD/learned-patterns.md`.**
- **Esta skill não roda hook `after spec`.** Pra evitar dupla publicação, o hook só dispara via `spec`/`spec --review-feedback`. `publish-spec` é manual e independente.
- **Esta skill não tem hook próprio.** Auxiliar, não tem ciclo de fluxo.
