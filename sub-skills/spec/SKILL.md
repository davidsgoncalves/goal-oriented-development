---
name: spec
description: |
  Produz a spec da task como artefato canônico, separada do plan. Lê a description bruta, busca dados externos (Jira/Figma), faz Q&A focada em escopo (proibido falar de implementação), escreve `spec.md` no `specs_path` configurado e roda review básico. É o passo entre `init` e `plan` no fluxo. Use quando o usuário mencionar: "criar spec", "spec da task", "escrever spec", ou quando a fase de spec for ativada pelo GOD.
tools: Read, Glob, Grep, Bash, Edit, Write, Agent
---

# Spec — Sub-skill de Especificação

> Produz a spec da task: o **WHAT e por quê**, separado do **HOW** (que vive no `plan.md`). Esta é a primeira skill a tocar dados externos (Jira/Figma) e a fazer Q&A com o usuário sobre escopo. A spec resultante é um artefato canônico do produto, não um arquivo de workflow — vive no `specs_path` configurado, fora da pasta `GOD/tasks/`.

## Posição no fluxo

```
init → spec → plan → implement → pack-up
       ^^^^
       você está aqui
```

`init` salvou apenas o input bruto. `plan` precisa da spec pronta pra começar a falar de arquitetura. **Esta skill é a ponte.**

## Flags

- `--review-feedback` — modo de loop de validação. Re-executa a spec incorporando feedback externo (comentário do PM/UX no Jira/Slack). Incrementa `spec_version` no frontmatter. Limite de 2 ciclos automáticos consecutivos antes de exigir conversa síncrona com o stakeholder. Ver passo 14 abaixo.
- `--quick` — passa adiante pro `review --spec`, pulando verificações semânticas profundas (mantém só lint estrutural). Útil pra iteração rápida.

## Instruções

Quando o usuário invocar esta skill, execute os seguintes passos **na ordem**:

### 0. Executar hook `before spec`

Ler `GOD/hooks.md` e localizar a seção `# before spec`.

- Se o conteúdo for `skip-hook`: pular e seguir para o passo 0.5.
- Se houver instruções em linguagem natural: executá-las integralmente antes de prosseguir.

### 0.5. Identificar a task

- Se o contexto da conversa já contém o código da task, usar esse código
- Caso contrário, perguntar ao usuário o código da task (ex: `PROJ-123`)

Verificar se `GOD/tasks/{cod-da-task}/description.md` existe. Se não existir, orientar o usuário a rodar `init` primeiro e encerrar.

### 1. Resolver o `specs_path` e garantir estrutura mínima

Ler `GOD/config.md` e extrair o valor de `specs_path` (root do repo de specs).

1. **Se `config.md` não existe ou não tem a seção `## specs_path`**: assumir default `docs/specs/` (relativo ao diretório onde o GOD está instalado) e informar o usuário ("usei o default `docs/specs/` — você pode editar `GOD/config.md` pra alterar").
2. **Resolver path:**
   - Se relativo, resolver a partir do diretório onde `GOD/` está instalado.
   - Se absoluto, usar como está.
3. **Garantir estrutura mínima** (criação defensiva — o `install` deveria ter feito, mas pode ter sido pulado em instalações pré-v6 ou se o usuário moveu manualmente):
   - Se `<specs_root>/` não existe: criar com `mkdir -p`.
   - Se `<specs_root>/tasks/` não existe: criar com `mkdir -p`.
   - Se `<specs_root>/README.md` não existe: criar com o template canônico abaixo.

Template do `<specs_root>/README.md` (criar apenas se não existir):

```markdown
# Specs

Repositório canônico de specs do produto. Cada spec é o **contrato de escopo** de uma feature/task — REQs em formato EARS, ACs com IDs estáveis, cenários e NFRs explícitos.

## Estrutura

```
specs/
├── tasks/                # Spec autocontida por task (uma por entrega)
│   └── <cod>.md          # ex: PROJ-123.md
├── domains/              # (reservado) Feature specs eternas por domínio do produto
└── flows/                # (reservado) Índices que amarram múltiplas features (jornadas e2e)
```

`tasks/` é populado pelo framework GOD (skill `spec`) na fase de SDD por task.
`domains/` e `flows/` serão populados por skills futuras quando feature specs eternas e fluxos cross-feature forem suportados.

## Convenções

- Toda spec começa com frontmatter: `spec_version`, `task`, `created_at`, `updated_at`.
- ACs têm IDs estáveis: `AC-NNN.N` (ex: `AC-001.2`). Não renomear depois de citar em código/teste.
- Specs não falam de implementação — proibido mencionar framework, biblioteca, banco. Isso é HOW, vai pro plan da task.
- Specs são commitadas como artefato de produto. Histórico via `git log`.
```

Guardar em memória: `<specs_root>` (root) e `<tasks_dir>` = `<specs_root>/tasks/`. A spec final será escrita em `<tasks_dir>/{cod-da-task}.md`.

### 2. Ler `description.md`

Ler `GOD/tasks/{cod-da-task}/description.md` para obter o input bruto que o `init` salvou.

A description pode estar em dois estados:
- **Bruta** (apenas input do usuário, como saída do `init`) — prosseguir com enriquecimento nos passos 3-7.
- **Já enriquecida em invocação anterior do `spec`** — pular para o passo 8 e perguntar ao usuário se quer re-enriquecer (refazer Q&A) ou só atualizar a spec sem repetir a coleta.

### 3. Buscar dados no Jira (se aplicável)

Se o input bruto contém um link ou código do Jira:

- Acessar o Jira via MCP Atlassian (ex: `getJiraIssue`) e obter:
  - Título da task
  - Descrição completa
  - Links do Figma (nos campos ou na descrição)
  - Critérios de aceitação que o ticket já trouxer
  - Parent task e subtasks (contexto mais amplo)

Se o input bruto é apenas texto manual (sem código Jira), pular este passo.

> Esta busca é responsabilidade desta skill (migrou do `plan` na v6). O `plan` agora lê a spec pronta e foca em arquitetura.

### 4. Consultar knowledge por tasks semelhantes

Ler `GOD/knowledge.md`. Verificar se há tasks anteriores com contexto semelhante.

Se encontrar, registrar:
- Códigos de commits relacionados (pra incluir como referência na spec, opcional)
- Aprendizados relevantes que podem afetar **escopo** (não implementação)

### 5. Analisar design do Figma (se aplicável)

Se o Jira trouxe links do Figma, ou se o input bruto já continha links:

- Acessar o Figma via MCP Figma (`get_design_context`)
- Identificar **estados, fluxos, edge cases visuais** — isso vira input pra Q&A e pra cenários da spec
- **Não detalhar implementação visual aqui** (cores, componentes específicos) — isso é HOW, vai pro plan

Se não houver links do Figma, pular.

### 6. Coletar contexto do produto

Ler arquivos canônicos do projeto que descrevem **produto**, não código:
- `README.md` (descrição do produto)
- `CLAUDE.md` (instruções do projeto)
- Specs já existentes em `<specs_root>/tasks/` que possam ter relação

A busca é case-insensitive. Não ler ARCHITECTURE.md aqui — isso é HOW, fica pro plan.

### 7. Sessão de Q&A com o usuário — escopo apenas

Fazer perguntas agrupadas ao usuário focadas em **escopo** e **comportamento esperado**. **Proibido perguntar sobre implementação** (linguagem, framework, padrão arquitetural, biblioteca). Se uma dúvida técnica aparecer naturalmente, anotar pra repassar ao `plan`.

Cobrir:

**Objetivo:**
- Qual problema essa task resolve?
- Quem é o usuário/ator afetado?

**Comportamento esperado:**
- O que deve acontecer no happy path?
- Edge cases relevantes?
- Cenários de erro?

**Escopo:**
- O que está explicitamente FORA do escopo?
- A task afeta apenas usuários novos / apenas existentes / ambos?
- Há limites temporais ou condicionais?

**NFRs (não-funcionais):**
- Performance importa nessa task? Algum limite explícito?
- LGPD / privacidade? PII envolvida?
- Acessibilidade?
- Observabilidade — algum evento precisa ser logado?

Diretrizes:
- **Não pergunte demais** — foque no que é ambíguo ou faltante.
- **Não pergunte de menos** — garanta que entendeu objetivo, ACs e edge cases.
- **Agrupe perguntas** — várias de uma vez, não uma por uma.
- Se a task é trivial e o input já é claro, **anuncie que não há dúvidas** e siga.

**Registrar todo o Q&A** — vai pra seção "Discovery" da spec.

### 8. Escrever `spec.md`

Escrever o arquivo em `<tasks_dir>/{cod-da-task}.md` (= `<specs_root>/tasks/{cod-da-task}.md`). Se já existir, pedir confirmação antes de sobrescrever.

Template canônico:

```markdown
---
spec_version: 1
task: {cod-da-task}
created_at: {timestamp-iso-8601-utc}
updated_at: {timestamp-iso-8601-utc}
---

# SPEC-{cod-da-task} — {título da task}

## Objetivo

{Uma frase clara do problema sendo resolvido. Sem solução técnica.}

## Não-objetivos

{O que está explicitamente fora de escopo. Se nada relevante: "Nenhum não-objetivo declarado.".}

## Discovery

{Q&A do passo 7 em formato pergunta/resposta. Se não houve Q&A: "Escopo claro a partir do input — sem dúvidas levantadas.".}

## Requisitos

### REQ-001: {nome curto}
**WHEN** {contexto disparador} **THEN** o sistema **SHALL** {comportamento esperado}.

#### Critérios de aceitação
- AC-001.1: {critério testável e específico}
- AC-001.2: ...

### REQ-002: ...
...

## Cenários

### Happy path
{Descrição do fluxo esperado em condições normais.}

### Edge cases
- {edge case 1}: {comportamento esperado}
- {edge case 2}: ...

### Erros
- {erro possível}: {comportamento esperado}

## NFRs

- **Performance:** {limite explícito ou "não aplicável"}
- **Segurança / LGPD:** {requisitos ou "sem dado pessoal envolvido"}
- **Acessibilidade:** {requisitos ou "não aplicável"}
- **Observabilidade:** {eventos a logar ou "não aplicável"}

## Referências

- Jira: {url ou "—"}
- Figma: {url ou "—"}
- Tasks semelhantes (knowledge): {códigos ou "nenhuma"}
```

**Regras de qualidade:**
- ACs são **testáveis e específicos** — "deve ser rápido" é vago; "responde em <200ms" é testável.
- Critérios usam IDs estáveis (`AC-001.1`) — vão ser referenciados depois.
- Não vazar implementação — se a spec menciona React, banco, framework, é violação.
- EARS (`WHEN ... THEN ... SHALL ...`) é o formato preferido pros REQs. Se um REQ não cabe em EARS, prosa imperativa é aceitável.

### 9. Atualizar referências cruzadas

Em `GOD/tasks/{cod-da-task}/description.md`, ao final, adicionar bloco de referência:

```markdown

---

> Spec canônica: `<tasks_dir>/{cod-da-task}.md` (criada por `spec` em {timestamp})
```

Esse bloco serve como ponteiro pro próximo dev/IA achar a spec sem precisar abrir `GOD/config.md`.

### 10. Rodar review

Chamar a sub-skill `review --spec` passando o código da task. Se a flag `--quick` foi passada pra esta skill, repassar pra `review --spec --quick`.

- Se o relatório retornar **Aprovado**: prosseguir.
- Se o relatório retornar **Ajustes necessários**: avaliar correções, aplicar as pertinentes na `spec.md` e seguir.
- Se o relatório retornar **Reprovado**: reescrever a spec com base no feedback e rodar a review novamente.

Verificações que o `--spec` faz (a partir da v7):
- **Estrutura sempre:** ACs têm IDs estáveis, cada REQ tem AC, todas as seções obrigatórias.
- **Lint sempre:** sem vazamento de implementação, sem vagueness, sem contradição.
- **Semânticas (skipadas com `--quick`):** cobertura description→spec profunda, cenários cobrem ACs, NFRs adequados, ortogonalidade, edge cases óbvios.

### 11. Atualizar `status.md`

Atualizar `GOD/tasks/{cod-da-task}/status.md`:

- `phase`: `specified`
- `updated_at`: timestamp ISO 8601 em UTC
- `updated_by`: `spec`
- `spec_path`: caminho da spec gerada (string, ex: `docs/specs/tasks/PROJ-123.md` ou `/Users/eu/projetos/vakinha-workspace/vakinha-specs/tasks/PROJ-123.md`)
- `branch`, `branch_base`, `learned`, `prs`: preservar valores atuais

### 12. Executar hook `after spec`

Ler `GOD/hooks.md` e localizar a seção `# after spec`.

- Se o conteúdo for `skip-hook`: pular e seguir para o passo 13.
- Se houver instruções em linguagem natural: executá-las integralmente antes do relatório final.

Casos comuns: postar comentário no Jira da task com resumo da spec, notificar canal Slack, atualizar card Linear/Trello.

**Pós-execução do hook**, atualizar o frontmatter da spec recém-escrita com:

```yaml
last_published_at: {timestamp-iso-8601-utc}
published_to: [<lista de destinos extraída do hook executado, ex: jira, slack>]
```

Esses campos rastreiam ciência ativa de quem foi notificado. Se o hook for `skip-hook`, **não** definir esses campos (omitir do frontmatter).

### 13. Reportar resultado

> ✅ Spec criada para `{cod-da-task}`!
>
> 📐 Arquivo: `<tasks_dir>/{cod-da-task}.md` (spec_version: {N})
> 📊 {N requisitos, M critérios de aceitação registrados}
> 📡 Publicada em: {lista de destinos, ou "não publicada (hook after spec = skip-hook)"}
>
> 💡 Próximo passo: rode `plan` — ela vai ler a spec e produzir o plano de implementação (arquitetura, arquivos, passos).
>
> 💬 Se chegar feedback do PM/UX/stakeholder antes do `plan`, rode `spec --review-feedback` pra incorporar.

---

### Modo `--review-feedback` (loop de validação leve)

Modo opt-in. Acionado quando o usuário roda `spec --review-feedback` numa task que **já tem spec criada** (`spec_path` populado em `status.md`). Serve pra incorporar feedback recebido fora do GOD (comentário no Jira, mensagem no Slack, conversa).

**Quando usar:**
- O hook `after spec` publicou a spec num canal externo (Jira/Slack), o stakeholder respondeu, e você quer atualizar a spec antes do `plan` rodar.
- Você revisou a spec sozinho depois e quer ajustar com nota explícita.
- Stakeholder sinalizou em conversa síncrona algo que muda escopo, mas a feature ainda não foi pra `plan`.

**Pré-requisitos:**
- `status.md` deve ter `spec_path` populado (spec criada anteriormente).
- A task ainda não passou de `phase: specified` — se já está em `planned`/`implementing`/etc, **abortar** e orientar a usar `update-spec` (que entra na v9 — por enquanto, recomendar conversa síncrona + retomar do plan).

**Passos:**

1. **Solicitar o feedback.** Pedir ao usuário pra colar o feedback bruto (comentário do Jira, mensagem do Slack, ou texto livre):

   > Cole aqui o feedback do stakeholder (pode ser comentário do Jira, mensagem do Slack, ou texto livre). Eu vou identificar quais REQs/ACs ele afeta:

2. **Verificar contagem de ciclos.** Ler o frontmatter da spec atual e contar quantos `--review-feedback` consecutivos já rodaram (campo `feedback_cycles` — ver passo 5). Se já chegou em 2 ciclos seguidos sem `plan` rodar entre eles, **alertar:**

   > ⚠️ Esta é a 3ª iteração consecutiva de feedback sem `plan` rodar. Isso costuma sinalizar que a conversa precisa ser síncrona (vídeo/presencial), não por troca escrita. Quer (a) seguir mesmo assim, (b) abortar e marcar pra conversa síncrona?

3. **Analisar o feedback contra a spec.** Ler a spec atual e identificar:
   - Que REQs ou ACs o feedback afeta diretamente
   - Que mudanças sugere (ampliar AC, adicionar REQ, remover não-objetivo, etc.)
   - Se o feedback contradiz a spec atual ou apenas amplia

4. **Apresentar mudanças propostas pro usuário.** Antes de aplicar:

   > Feedback identifica as seguintes mudanças:
   > - **AC-001.2:** ampliar pra aceitar telefone fixo (8 dígitos)
   > - **REQ-003:** adicionar (exportação CSV de telefones no admin)
   > - **AC-002.1:** remover (deprecated pelo PM)
   >
   > Aplicar tudo? (s) ou ajustar antes? (a)

5. **Aplicar mudanças e incrementar versão.** Editar a spec em `<spec_path>`:
   - Aplicar as mudanças aprovadas
   - Incrementar `spec_version` (ex: 1 → 2)
   - Atualizar `updated_at` no frontmatter
   - Adicionar/incrementar campo `feedback_cycles` no frontmatter (default 0; cada `--review-feedback` += 1)
   - Adicionar bloco no final da spec (ou em seção "Histórico de feedback"):

     ```markdown
     ## Histórico de feedback

     ### v{N} — {timestamp}
     **Origem:** {Jira comment | Slack | conversa | texto livre}
     **Resumo:** {1 linha do que foi pedido}
     **Aplicado:** {lista de REQs/ACs alterados}
     ```

6. **Re-rodar `review --spec`** (sem `--quick`) pra validar que a spec atualizada continua consistente.

7. **Re-executar `after spec` hook** se configurado, pra notificar que a spec foi atualizada.

8. **Atualizar `status.md`:**
   - `phase`: continua `specified` (não muda — task ainda não foi planejada)
   - `updated_at`: novo timestamp
   - `updated_by`: `spec`
   - `spec_version_consumed`: **NÃO atualizar aqui** (esse campo só muda quando `plan` ou `implement` rodam — assim eles detectam o drift na próxima vez).

9. **Reportar:**

   > ✅ Spec atualizada com feedback!
   >
   > 📐 spec_version: v{N-1} → v{N}
   > 📝 Mudanças: {resumo}
   > 🔄 Ciclos de feedback consecutivos: {feedback_cycles}/2 antes de exigir conversa síncrona
   > 📡 Re-publicada em: {destinos do after spec}
   >
   > 💡 Próximo passo: se não há mais feedback esperado, rode `plan`. Se outro ciclo chegar, `spec --review-feedback` de novo.

**Reset do contador `feedback_cycles`:**
- `plan` rodando com sucesso → reset pra 0 (a spec foi consumida).
- `implement` rodando → reset pra 0.

---

## Guard-rails

- **Esta skill é a única dona da spec.** O `plan` lê e referencia, mas não escreve. O `pack-up` linka, mas não edita. `update-spec` (v9, futuro) também escreverá — mas ainda não existe.
- **Esta skill não escreve em `GOD/knowledge.md` nem em `GOD/learned-patterns.md`.** Apenas a skill `learn` pode fazê-lo.
- **Esta skill não toca em git.** Não cria branch, não comita.
- **Esta skill não fala de implementação.** Q&A, REQs e ACs são puramente sobre comportamento e escopo. Se um detalhe técnico aparecer, ele vai pra anotação separada que o `plan` consome — não pro corpo da spec.
- **Modo `--review-feedback` só roda em `phase: specified`.** Tasks que já passaram pro `plan`/`implement` exigem mecanismo diferente (v9: `update-spec` com propagação delta).
- **Hooks `before/after spec` (v7).** Antes da v7 esses hooks não existiam. Após upgrade pra v7, eles aparecem em `hooks.md` com default `skip-hook` — usuário ativa quando quiser.
