---
name: spec
description: |
  Produz a spec da task como artefato canônico, **antes** do init. Aceita input bruto direto (link Jira / código / texto livre), busca dados externos (Jira/Figma), faz Q&A focada em escopo (proibido falar de implementação), detecta o perfil da task (trivial/normal/critical) e escreve `spec.md` no `specs_path` configurado. É a primeira skill do fluxo na v9. Use quando o usuário mencionar: "criar spec", "spec da task", "escrever spec", ou quando começar uma task nova com perfil normal/critical.
tools: Read, Glob, Grep, Bash, Edit, Write, Agent
---

# Spec — Sub-skill de Especificação (entry point do fluxo)

> Produz a spec da task: o **WHAT e por quê**, separado do **HOW** (que vive no `plan.md`). Esta é a **primeira skill do fluxo na v9**: aceita input bruto direto, busca dados externos (Jira/Figma), e produz o artefato canônico antes que qualquer estrutura de execução exista. A spec resultante é um artefato canônico do produto, não um arquivo de workflow — vive no `specs_path` configurado, fora da pasta `GOD/tasks/`.

## Posição no fluxo (v9)

```
spec → [publish-spec] → init → plan → implement → pack-up
^^^^
você está aqui (entry point)
```

A skill `spec` agora **antecede** o `init`. Não depende de `GOD/tasks/{cod}/` existir — escreve direto em `<specs_path>/tasks/{cod}.md`. O `init` virá depois pra criar branch + estrutura de execução.

> **Mudança v9 (spec-first):** spec roda antes de init. Captura input bruto como seção da própria spec (não há mais `description.md` separado em tasks novas). Tasks legacy v8 continuam funcionando — esta skill detecta `GOD/tasks/{cod}/description.md` existente e lê de lá pra retrocompat.

## Flags

- `--review-feedback` — modo de loop de validação. Re-executa a spec incorporando feedback externo (comentário do PM/UX no Jira/Slack). Incrementa `spec_version` no frontmatter. Limite de 2 ciclos automáticos consecutivos antes de exigir conversa síncrona com o stakeholder. Ver seção dedicada abaixo.
- `--quick` — pula verificações semânticas profundas do `review --spec` (mantém só lint estrutural). Útil pra iteração rápida em tasks normais. **Em modo `--quick`, hook `after spec` é executado mas a sugestão de `publish-spec` ao final é silenciada.**
- `--type=normal|critical` — força o perfil da task, dispensando heurística + confirmação. **Não há `--type=trivial` aqui** — task trivial não precisa de spec; o usuário deve rodar `init --type=trivial` diretamente.

## Instruções

Quando o usuário invocar esta skill, execute os seguintes passos **na ordem**:

### 0. Executar hook `before spec`

Ler `GOD/hooks.md` e localizar a seção `# before spec`.

- Se o conteúdo for `skip-hook`: pular e seguir para o passo 0.5.
- Se houver instruções em linguagem natural: executá-las integralmente antes de prosseguir.

### 0.4. Sincronizar repo de specs (sync check)

Antes de checar se a spec existe localmente, garantir que o filesystem reflete o estado remoto. Sem isso, dois devs trabalhando na mesma task podem criar specs divergentes silenciosamente.

1. **Resolver `specs_path`** (apenas pra esta verificação — a resolução completa é no passo 1). Ler `GOD/config.md` ou usar default `docs/specs/`.

2. **Detectar se `<specs_path>` é git:**
   - Rodar `git -C <specs_path> rev-parse --git-dir` (silenciar erro).
   - Se sucede → repo git. Prosseguir com fetch.
   - Se falha → pasta sem git (raro). Pular este passo.

3. **Fetch silencioso:**
   - `git -C <specs_path> fetch --quiet` (~1s típico).
   - Se falhar (sem rede, repo inválido, credenciais): avisar uma vez, perguntar se quer prosseguir mesmo assim. Não abortar automaticamente — dev offline ainda precisa rodar a skill.

4. **Comparar HEAD local vs remoto:**
   - Identificar branch atual: `git -C <specs_path> rev-parse --abbrev-ref HEAD`.
   - Identificar tracking remoto: `git -C <specs_path> rev-parse --abbrev-ref --symbolic-full-name @{u}` (silenciar erro — branch pode não ter upstream).
   - Se sem upstream: pular comparação (não dá pra saber se está atualizado), prosseguir.
   - Comparar: `git -C <specs_path> rev-list --left-right --count HEAD...@{u}` retorna `A B` onde A = commits locais à frente, B = commits remotos à frente.

5. **Reagir conforme estado:**
   - **`B == 0`** (local em paridade ou à frente): tudo OK, prosseguir silenciosamente.
   - **`B > 0`** (remoto tem commits novos não puxados): **alertar duro**:

     > ⚠️ Repo de specs (`<specs_path>`) tem **{B} commits remotos não puxados**.
     >
     > Outro dev pode ter trabalhado em specs nesta janela — incluindo, possivelmente, a spec da task que você quer agora.
     >
     > Como prosseguir?
     > - (a) **Puxar agora** (`git pull --ff-only` no repo de specs) e seguir — recomendado.
     > - (b) **Prosseguir mesmo assim** (risco real de conflito de merge no push depois).
     > - (c) **Abortar** e fazer pull manual primeiro.

   - Se (a) → executar `git -C <specs_path> pull --ff-only`. Se fast-forward falhar (merge não-trivial necessário), abortar com orientação ("seu repo de specs tem divergências locais e remotas. Resolva manual antes de rodar `spec`").
   - Se (b) → prosseguir, mas registrar em memória que houve aviso (não persiste — só pra log do passo final).
   - Se (c) → encerrar silenciosamente.

### 0.5. Identificar a task e capturar input bruto

A spec na v9 **não exige `description.md` prévio**. Aceita input direto. Estados possíveis:

**Caso A — Task nova (fluxo v9 normal):**
- O usuário passa link/código do Jira ou texto livre direto na invocação.
- Se nada foi passado, perguntar: "Link do Jira, código da task ou descrição livre?"
- Extrair o código da task:
  - Link Jira → extrair (ex: `PROJ-123`)
  - Código puro → usar direto
  - Texto livre → pedir um identificador curto (ex: `meu-feature`)
- Guardar o input bruto em memória (vai pra seção `## Input bruto` da spec no passo 8).

**Caso B — Spec já existe em `<specs_path>/tasks/{cod}.md`:**
- Detectar lendo o frontmatter. Se sim, pular pra passo 8 e perguntar ao usuário se quer:
  - (a) re-rodar Q&A e regenerar spec do zero (cuidado: descarta histórico)
  - (b) atualizar a spec sem repetir Q&A (modo `--review-feedback` é mais apropriado pra incorporar feedback)
  - (c) abortar

**Caso C — Task legacy v8 (`GOD/tasks/{cod}/description.md` existe):**
- Detectar e ler o input bruto de lá. Avisar o usuário:

  > 📦 Detectada task legacy (v8): `GOD/tasks/{cod}/description.md` existe. Lendo o input de lá pra retrocompat. A spec gerada vai pra `<specs_path>/tasks/{cod}.md` (estrutura v9 do specs_path).

- Não criar nova seção de input — o `description.md` legacy permanece como referência.

### 1. Resolver o `specs_path` e garantir estrutura mínima

Ler `GOD/config.md` e extrair o valor de `specs_path` (root do repo de specs).

1. **Se `config.md` não existe ou não tem a seção `## specs_path`**: assumir default `docs/specs/` (relativo ao diretório onde o GOD está instalado) e informar o usuário ("usei o default `docs/specs/` — você pode editar `GOD/config.md` pra alterar").
2. **Resolver path:**
   - Se relativo, resolver a partir do diretório onde `GOD/` está instalado.
   - Se absoluto, usar como está.
3. **Garantir estrutura mínima:**
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
├── tasks/                      # Spec autocontida por task (uma por entrega)
│   ├── <cod>.md                # ex: PROJ-123.md
│   └── <cod>-changelog.md      # (v9, sob demanda) histórico de mudanças via update-spec
├── domains/                    # (reservado) Feature specs eternas por domínio do produto
└── flows/                      # (reservado) Índices que amarram múltiplas features (jornadas e2e)
```

`tasks/` é populado pelo framework GOD (skill `spec`) na fase de SDD por task.
`domains/` e `flows/` serão populados por skills futuras quando feature specs eternas e fluxos cross-feature forem suportados.

## Convenções

- Toda spec começa com frontmatter: `spec_version`, `task`, `created_at`, `updated_at`.
- ACs têm IDs estáveis: `AC-NNN.N` (ex: `AC-001.2`). Não renomear depois de citar em código/teste.
- Specs não falam de implementação — proibido mencionar framework, biblioteca, banco. Isso é HOW, vai pro plan da task.
- Specs são commitadas como artefato de produto. Histórico via `git log` e via `<cod>-changelog.md` (quando `update-spec` foi usado).
```

Guardar em memória: `<specs_root>` (root) e `<tasks_dir>` = `<specs_root>/tasks/`. A spec final será escrita em `<tasks_dir>/{cod-da-task}.md`.

### 1.5. Detectar perfil da task

Determinar o perfil que afeta cerimônia ao final do fluxo. Três perfis:

| Perfil | Quando se aplica | Comportamento ao final |
|--------|------------------|-------------------------|
| `trivial` | Mudança cosmética (copy, typo, dep upgrade) | **Não cabe nesta skill.** Spec aborta com sugestão de rodar `init --type=trivial` direto |
| `normal` | Feature curta, bug com escopo claro, sem stakeholder externo ativo | Spec gerada, `publish-spec` **não sugerido** ao final |
| `critical` | Feature grande, múltiplos stakeholders, exige aprovação externa | Spec gerada, `publish-spec` **sugerido** ao final pra travar antes do `init` |

**Resolução de perfil:**

1. Se `--type=normal` ou `--type=critical` foi passado → usar direto, pular heurística.
2. Caso contrário, aplicar **heurística**:
   - **Sinais de `trivial`** (qualquer um basta): input ≤ 80 caracteres + sem código Jira; texto contém termos como "typo", "trocar copy", "renomear botão", "atualizar dep", "mudar texto" sem detalhes; nenhum AC explícito mencionado.
   - **Sinais de `critical`** (qualquer um basta): input vem de Jira com >3 comentários; descrição menciona stakeholder externo (PM, UX, legal, compliance); descrição menciona impacto em múltiplos sistemas/projetos; ACs já mencionam métricas de SLA, LGPD, compliance, ou auditoria.
   - **Default**: `normal`.
3. **Confirmação rápida com o usuário** (sempre, mesmo sem ambiguidade — custa pouco e evita pular publish-spec por engano):

   > 🎯 Detectei o perfil desta task como **`{perfil}`**.
   >
   > - `trivial` → pula spec, vai direto pro `init --type=trivial`.
   > - `normal` → spec sem publicação automática, segue pro `init`.
   > - `critical` → spec + sugestão de `publish-spec` pra validar com stakeholder antes do `init`.
   >
   > Confirma o perfil **`{perfil}`**? ([s] confirma / [t] trivial / [n] normal / [c] critical)

4. Se o usuário escolheu `trivial`, **abortar imediatamente** com:

   > ⏭️ Task `{cod}` é trivial. Spec dispensada.
   >
   > 💡 Próximo passo: `init {cod} --type=trivial` (cria branch sem passar por spec/plan).

5. Guardar perfil escolhido em memória (vai pro frontmatter da spec).

### 2. Buscar dados no Jira (se aplicável)

Se o input bruto contém um link ou código do Jira:

- Acessar o Jira via MCP Atlassian (ex: `getJiraIssue`) e obter:
  - Título da task
  - Descrição completa
  - Links do Figma (nos campos ou na descrição)
  - Critérios de aceitação que o ticket já trouxer
  - Parent task e subtasks (contexto mais amplo)
  - Comentários relevantes (entram como insumo da Q&A)

Se o input bruto é apenas texto manual (sem código Jira), pular este passo.

### 3. Consultar knowledge por tasks semelhantes

Ler `GOD/knowledge.md`. Verificar se há tasks anteriores com contexto semelhante.

Se encontrar, registrar:
- Códigos de commits relacionados (pra incluir como referência na spec, opcional)
- Aprendizados relevantes que podem afetar **escopo** (não implementação)

### 4. Analisar design do Figma (se aplicável)

Se o Jira trouxe links do Figma, ou se o input bruto já continha links:

- Acessar o Figma via MCP Figma (`get_design_context`)
- Identificar **estados, fluxos, edge cases visuais** — isso vira input pra Q&A e pra cenários da spec
- **Não detalhar implementação visual aqui** (cores, componentes específicos) — isso é HOW, vai pro plan

Se não houver links do Figma, pular.

### 5. Coletar contexto do produto

Ler arquivos canônicos do projeto que descrevem **produto**, não código:
- `README.md` (descrição do produto)
- `CLAUDE.md` (instruções do projeto)
- Specs já existentes em `<specs_root>/tasks/` que possam ter relação

A busca é case-insensitive. Não ler ARCHITECTURE.md aqui — isso é HOW, fica pro plan.

### 6. Sessão de Q&A com o usuário — escopo apenas

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

### 7. Resolver título e composição final

Antes de escrever:
- **Título** da spec: vem do Jira (se disponível) ou perguntar ao usuário.
- **Timestamp**: ISO 8601 em UTC para `created_at` e `updated_at`.
- **Frontmatter `profile`**: o perfil resolvido no passo 1.5 (`normal` ou `critical`).

### 8. Escrever `spec.md`

Escrever o arquivo em `<tasks_dir>/{cod-da-task}.md` (= `<specs_root>/tasks/{cod-da-task}.md`). Se já existir, pedir confirmação antes de sobrescrever.

Template canônico (v9):

```markdown
---
spec_version: 1
task: {cod-da-task}
profile: {normal|critical}
created_at: {timestamp-iso-8601-utc}
updated_at: {timestamp-iso-8601-utc}
---

# SPEC-{cod-da-task} — {título da task}

## Input bruto

{input cru fornecido pelo usuário ou trazido do Jira — ipsis litteris, sem enriquecer.
Caso a task seja legacy v8 e a leitura tenha vindo de `GOD/tasks/{cod}/description.md`,
copiar o input dali. Esta seção substitui o `description.md` separado em tasks v9.}

## Objetivo

{Uma frase clara do problema sendo resolvido. Sem solução técnica.}

## Não-objetivos

{O que está explicitamente fora de escopo. Se nada relevante: "Nenhum não-objetivo declarado.".}

## Discovery

{Q&A do passo 6 em formato pergunta/resposta. Se não houve Q&A: "Escopo claro a partir do input — sem dúvidas levantadas.".}

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
- A seção `## Input bruto` preserva o material original sem edição — não tente "limpar".

### 9. Rodar review

Chamar a sub-skill `review --spec` passando o código da task. Se a flag `--quick` foi passada pra esta skill, repassar pra `review --spec --quick`.

- Se o relatório retornar **Aprovado**: prosseguir.
- Se o relatório retornar **Ajustes necessários**: avaliar correções, aplicar as pertinentes na `spec.md` e seguir.
- Se o relatório retornar **Reprovado**: reescrever a spec com base no feedback e rodar a review novamente.

Verificações que o `--spec` faz (a partir da v7):
- **Estrutura sempre:** ACs têm IDs estáveis, cada REQ tem AC, todas as seções obrigatórias.
- **Lint sempre:** sem vazamento de implementação, sem vagueness, sem contradição.
- **Semânticas (skipadas com `--quick`):** cobertura input→spec profunda, cenários cobrem ACs, NFRs adequados, ortogonalidade, edge cases óbvios.

### 10. Executar hook `after spec`

Ler `GOD/hooks.md` e localizar a seção `# after spec`.

- Se o conteúdo for `skip-hook`: pular e seguir para o passo 11.
- Se houver instruções em linguagem natural: executá-las integralmente antes do relatório final.

Casos comuns: postar comentário no Jira da task com resumo da spec, notificar canal Slack, atualizar card Linear/Trello.

**Pós-execução do hook**, atualizar o frontmatter da spec recém-escrita com:

```yaml
last_published_at: {timestamp-iso-8601-utc}
published_to: [<lista de destinos extraída do hook executado, ex: jira, slack>]
```

Esses campos rastreiam ciência ativa de quem foi notificado. Se o hook for `skip-hook`, **não** definir esses campos (omitir do frontmatter).

### 11. Reportar resultado

Texto adaptado ao perfil:

**Para `profile: critical`:**

> ✅ Spec criada para `{cod-da-task}` (perfil: critical)!
>
> 📐 Arquivo: `<tasks_dir>/{cod-da-task}.md` (spec_version: 1)
> 📊 {N requisitos, M critérios de aceitação registrados}
> 📡 Hook after spec: {executado / skip}
>
> 🚧 **Esta task é crítica. Não rode `init` ainda** — recomendo travar a spec antes:
>
> 1. Rode `publish-spec {cod-da-task}` pra publicar em Jira/Slack/etc.
> 2. Aguarde validação do stakeholder.
> 3. Se vier feedback: `spec --review-feedback {cod-da-task}` pra incorporar.
> 4. Quando aprovada: `init {cod-da-task}` (cria branch + estrutura de execução).

**Para `profile: normal`:**

> ✅ Spec criada para `{cod-da-task}` (perfil: normal)!
>
> 📐 Arquivo: `<tasks_dir>/{cod-da-task}.md` (spec_version: 1)
> 📊 {N requisitos, M critérios de aceitação registrados}
> 📡 Hook after spec: {executado / skip}
>
> 💡 Próximo passo: `init {cod-da-task}` — cria branch + estrutura de execução apontando pra esta spec.
>
> 💬 Se chegar feedback antes do `init`, rode `spec --review-feedback {cod-da-task}` pra incorporar.

**Em modo `--quick`**, suprimir o bloco "Próximo passo" relativo a `publish-spec` mesmo se perfil for critical (pode ser rodado manual depois).

---

## Modo `batch` (invocação programática pelo init-tree)

Modo não-interativo. Acionado quando `init-tree` chama `spec` programaticamente pra cada folha de uma árvore Jira. Difere do fluxo interativo nestes pontos:

| Comportamento | Modo interativo (padrão) | Modo `batch` |
|---------------|--------------------------|---------------|
| Q&A com usuário (passo 6) | Executa | **Pula** |
| Heurística de perfil (passo 1.5) | Aplica + confirma | Aplica **sem confirmar** (default `normal`, override só se sinais claros de `critical`) |
| `review --spec` (passo 9) | Executa | **Pula** |
| Hook `after spec` (passo 10) | Executa | **Pula** |
| Frontmatter | `draft` ausente | `draft: true` |
| Reportar (passo 11) | Imprime instruções pro usuário | Retorna silenciosamente |

**Output esperado:** spec rascunho em `<specs_path>/tasks/{cod}.md` que o usuário refina depois rodando `spec {cod}` no modo interativo. Quando o usuário roda `spec {cod}` interativo numa task com `draft: true`, esta skill detecta a flag e oferece:

> 📝 Esta spec foi gerada em batch (rascunho). Quer:
> - (a) Re-rodar Q&A completa e regenerar a spec (mantém input bruto)
> - (b) Apenas refinar manualmente o que já está aqui
> - (c) Manter como rascunho e abortar

Em (a): roda fluxo normal completo (passos 6, 9, 10), substitui `draft: true` por ausência da flag (spec deixa de ser rascunho), incrementa `spec_version`.

Em (b): apenas roda `review --spec`, pede edição manual do conteúdo entre rodadas, mantém ou remove `draft` quando o usuário sinalizar.

---

## Modo `--review-feedback` (loop de validação leve)

Modo opt-in. Acionado quando o usuário roda `spec --review-feedback {cod}` numa task que **já tem spec criada**. Serve pra incorporar feedback recebido fora do GOD (comentário no Jira, mensagem no Slack, conversa).

**Quando usar:**
- O hook `after spec` ou `publish-spec` publicou a spec num canal externo, o stakeholder respondeu, e você quer atualizar a spec antes do `init` rodar.
- Você revisou a spec sozinho depois e quer ajustar com nota explícita.
- Stakeholder sinalizou em conversa síncrona algo que muda escopo, mas a feature ainda não passou por `init`.

**Pré-requisitos:**
- A spec deve existir em `<specs_path>/tasks/{cod}.md`.
- A task **ainda não passou por `init`** — verificar se `GOD/tasks/{cod}/status.md` existe. Se existir, **abortar** e orientar a usar `update-spec` (que aplica mudança pós-init com propagação de delta).

**Passos:**

1. **Solicitar o feedback.** Pedir ao usuário pra colar o feedback bruto:

   > Cole aqui o feedback do stakeholder (pode ser comentário do Jira, mensagem do Slack, ou texto livre). Eu vou identificar quais REQs/ACs ele afeta:

2. **Verificar contagem de ciclos.** Ler o frontmatter da spec atual e contar `feedback_cycles`. Se já chegou em 2 ciclos seguidos sem `init` rodar entre eles, **alertar:**

   > ⚠️ Esta é a 3ª iteração consecutiva de feedback sem `init` rodar. Isso costuma sinalizar que a conversa precisa ser síncrona (vídeo/presencial), não por troca escrita. Quer (a) seguir mesmo assim, (b) abortar e marcar pra conversa síncrona?

3. **Analisar o feedback contra a spec.** Identificar:
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

5. **Aplicar mudanças e incrementar versão.** Editar a spec:
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

8. **Reportar:**

   > ✅ Spec atualizada com feedback!
   >
   > 📐 spec_version: v{N-1} → v{N}
   > 📝 Mudanças: {resumo}
   > 🔄 Ciclos de feedback consecutivos: {feedback_cycles}/2 antes de exigir conversa síncrona
   > 📡 Re-publicada em: {destinos do after spec}
   >
   > 💡 Próximo passo: se não há mais feedback esperado, rode `init {cod}`. Se outro ciclo chegar, `spec --review-feedback {cod}` de novo.

**Reset do contador `feedback_cycles`:**
- `init` rodando com sucesso → reset pra 0 (a spec foi consumida na criação da task).
- `update-spec` rodando → reset pra 0 (mudança formal pós-init usa outro mecanismo).

---

## Guard-rails

- **Esta skill é a única dona da spec inicial.** O `plan` lê e referencia, mas não escreve. O `pack-up` linka, mas não edita. `update-spec` (v9) escreve mudanças pós-init.
- **Esta skill não escreve em `GOD/knowledge.md` nem em `GOD/learned-patterns.md`.** Apenas a skill `learn` pode fazê-lo.
- **Esta skill não toca em git.** Não cria branch, não comita.
- **Esta skill não cria `GOD/tasks/{cod}/`.** Estrutura de execução é responsabilidade do `init`. A spec vive em `<specs_path>/`, isolada da execução.
- **Esta skill não fala de implementação.** Q&A, REQs e ACs são puramente sobre comportamento e escopo. Se um detalhe técnico aparecer, ele vai pra anotação separada que o `plan` consome — não pro corpo da spec.
- **Modo `--review-feedback` só roda antes do `init`.** Se `GOD/tasks/{cod}/status.md` existe, abortar e orientar `update-spec`.
- **Hooks `before/after spec` (v7).** Continuam disparando sempre, independente do perfil. O perfil afeta apenas a sugestão final ao usuário sobre rodar `publish-spec`.
- **Task trivial não passa por aqui.** Spec aborta com sugestão de `init --type=trivial`. Não tente forçar uma "spec mínima" pra typo — isso polui o repo de specs.
