# Migração v5 → v6

## Objetivo

A v6 introduz **Spec-Driven Development** no GOD: a spec da task vira artefato canônico, separada do `plan.md`, vivendo em `<specs_path>/tasks/<cod>.md` (configurável), fora da pasta `GOD/tasks/`. O `specs_path` é o root do repo de specs — subpastas `domains/` e `flows/` são reservadas pra skills futuras.

Mudanças estruturais:

- **Nova sub-skill `spec`** — posicionada entre `init` e `plan` no fluxo principal. Faz Q&A focada em escopo, busca dados em Jira/Figma, escreve `spec.md` em formato EARS com REQs e ACs numerados.
- **`plan` reescrito** — perde a responsabilidade de Q&A de escopo, busca em Jira/Figma e ACs. Passa a ler a spec pronta e focar em **HOW** (arquitetura, arquivos, passos).
- **`description.md` simplificado** — volta a ser apenas input bruto do usuário (como saía do `init` na v5). Q&A migrou pra spec.
- **Novo arquivo `GOD/config.md`** — configuração local da instalação. Contém `specs_path` (onde escrever a spec).
- **Novo campo `spec_path` em `status.md`** — populado pela skill `spec` com o caminho da spec gerada.
- **Nova fase `specified`** — entre `initialized` e `planned`. Set pela skill `spec`.
- **`review` ganha modo `--spec`** — verificações estruturais e de conteúdo (IDs nos ACs, sem vazamento de implementação, sem vagueness, sem contradição).
- **`pack-up` anexa link da spec** no PR description automaticamente.

## Arquivos afetados

**Atualizado:**
- `GOD/VERSION` — conteúdo muda de `v5` para `v6`

**Criado (novo):**
- `GOD/config.md` — configuração local com `specs_path` (perguntado ao usuário durante a migração)

**Inalterados (nenhuma reescrita automática):**
- `GOD/knowledge.md`
- `GOD/patterns.md`
- `GOD/learned-patterns.md`
- `GOD/hooks.md`
- `GOD/tasks/` (todas as tasks, `status.md`, `description.md`, `plan.md`, `changelog.md`)
- `GOD/tasks/.archived/` (se existir)

**Mudanças de contrato (não de arquivos):**
- `init` aponta o próximo passo como `spec` em vez de `plan`. O frontmatter do `description.md` continua igual.
- `spec` é o novo dono de Q&A de escopo, busca em Jira/Figma e geração de ACs.
- `plan` lê `<spec_path>` extraído de `status.md` e referencia REQs/ACs no plano. Não enriquece mais a description.
- `implement` lê a spec antes do plan (passo 2 estendido) e consulta os ACs durante a escrita.
- `pack-up` anexa bloco "📐 Spec" ao corpo do PR quando `spec_path` estiver populado.
- `review` ganha modo `--spec` com verificações estruturais.
- Status enum: nova fase `specified` entre `initialized` e `planned`.

## Riscos conhecidos

- **Tasks v5 ativas continuam em v5.** A migração **não move ACs do `plan.md` antigo pra spec.md** automaticamente. Tasks com `phase: planned` ou `implementing` permanecem no fluxo antigo (sem spec). Quando essas tasks chegarem em `packed-up`, o ciclo termina sem spec — o que é ok pra trabalho em curso.
- **Tasks novas começam em v6.** Após a migração, qualquer `init` rodado deve ser seguido por `spec` (não direto pra `plan`).
- **`specs_path` é decisão importante.** Em multi-project workspace, prefira um repo dedicado (`./myorg-specs/` ou similar) pra ter uma única fonte de verdade. Em single-project, decidir entre dentro do repo (`docs/specs/`) ou repo dedicado ao lado depende de quantos repos vão consumir as specs.
- **Nenhum risco de perda de dados.** A migração só adiciona arquivos e atualiza VERSION; nada é removido nem reescrito.

---

## Passos da migração

### 1. Verificar pré-condições

- Confirmar que `GOD/VERSION` existe e contém `v5`.
- Se não existir ou estiver em versão anterior, abortar — o usuário deve rodar as migrações anteriores primeiro.

### 2. Detectar contexto e configurar `specs_path`

A v6 introduz o conceito de **repo de specs** — `specs_path` é o root, e o GOD escreve em `<specs_path>/tasks/<cod>.md`. Subpastas `domains/` e `flows/` são reservadas pra skills futuras.

#### 2.1. Detectar contexto

Rodar `git rev-parse --show-toplevel` no diretório atual:
- **Sucesso** → contexto **single-project** (estamos dentro de um repo).
- **Falha** → contexto **workspace multi-project** (pasta-mãe sem .git contendo repos).

#### 2.2. Apresentar opções calibradas pelo contexto

**Se single-project:**

```
📐 A v6 introduz spec-driven development.

A spec de cada task é artefato canônico do produto — committada e lida por todos,
separada da pasta GOD/ (que é workflow individual).

Onde guardar o repo de specs?

Opções:
  1. docs/specs/                                  (default — dentro deste repo)
  2. ../<workspace>/<nome-do-repo-de-specs>/      (repo dedicado ao lado deste)
  3. <path-absoluto>                              (qualquer outro lugar)
  4. <outro-relativo>

Escolha (1/2/3/4) ou Enter pra default:
```

**Se multi-project workspace:**

```
📐 A v6 introduz spec-driven development.

Detectei workspace multi-project. Recomendado: repo dedicado de specs ao lado dos repos de código.

Opções:
  1. ./<nome-do-repo-de-specs>/    (repo dedicado neste workspace — recomendado)
  2. <path-absoluto>
  3. <outro-relativo>

Escolha (1/2/3) ou Enter pra opção 1:
```

#### 2.3. Resolver e criar estrutura

Com `specs_path` decidido:

1. Se a pasta não existe → `mkdir -p <specs_path>`.
2. Se a pasta existe e não é repo git → perguntar `"Inicializar como repo git separado? (s/n)"`. Se sim → `git init` lá. Se não → prosseguir como pasta.
3. Se já é repo git → usar como está.
4. Criar `<specs_path>/tasks/` (subpasta obrigatória da v6).
5. Criar `<specs_path>/README.md` se não existir, usando o template abaixo.
6. **Não criar `domains/` nem `flows/`** — reservados pra skills futuras.

Template do `<specs_path>/README.md`:

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

### 3. Criar `GOD/config.md`

Se `GOD/config.md` **já existir** (improvável, defensivo):
- Se já tem seção `## specs_path` com valor: manter como está, informar no relatório final.
- Se não tem: anexar a seção mantendo o resto.

Se `GOD/config.md` **não existir**, criar com:

```markdown
# Config — Configuração local do GOD

> Configuração específica desta instalação do GOD no projeto. Diferente do `patterns.md`,
> que descreve convenções **do projeto** (lidas por todos), este arquivo descreve onde o
> GOD escreve/lê **artefatos** dentro do filesystem. Pode variar entre desenvolvedores se
> a pasta `GOD/` estiver no `.gitignore`.

## specs_path

Root do repo de specs. O GOD escreve em `<specs_path>/tasks/<cod>.md`. Subpastas
`domains/` e `flows/` são reservadas pra skills futuras.

Aceita relativo (resolve a partir do diretório onde o GOD foi instalado) ou absoluto.

{specs_path}

> Cenários:
> - Local no repo: `docs/specs/`
> - Workspace multi-project: `./myorg-specs/` ou `../myorg-specs/`
> - Repo separado em qualquer lugar: `/Users/eu/projetos/myorg-workspace/myorg-specs/`
```

### 4. Atualizar VERSION

Sobrescrever o conteúdo de `GOD/VERSION` com:

```
v6
```

### 5. Não tocar em tasks ativas

A migração **deliberadamente não modifica** `GOD/tasks/{cod}/` para nenhuma task existente.

- Tasks em `phase: initialized` (sem plano ainda) ficam viáveis na v6 com fluxo `spec → plan → implement → pack-up`.
- Tasks em `phase: planned` ou posteriores **terminam em v5** (sem spec extraída).
- Status enum aceita `specified` retroativamente (nada quebra), mas não é injetado em nenhuma task existente.

### 6. Relatório final

```
✅ Migração v5 → v6 concluída!

VERSION atualizado: v6

Repo de specs configurado:
  📐 specs_path: {valor escolhido}
  📂 Pasta {criada / já existia}
  📂 Subpasta tasks/ {criada}
  📄 README.md {criado / já existia}
  🔧 git: {repo separado inicializado / repo já existente / pasta dentro do projeto / pasta sem git}

Arquivo criado:
  - GOD/config.md

Novidades disponíveis:
  📐 spec (sub-skill nova)         — produz a spec canônica entre init e plan
  📊 review --spec                 — review estrutural da spec
  📂 repo de specs                 — root configurável fora da pasta GOD/
  🏷️ phase: specified              — nova fase entre initialized e planned
  🔗 pack-up anexa link da spec    — no PR description automaticamente

Fluxo atualizado:
  init → spec → plan → implement → pack-up
         ^^^^
         novo passo

Tasks já em andamento (phase: planned ou posterior) continuam no fluxo v5 até o pack-up.
Novas tasks (init rodado após esta migração) seguem o fluxo v6 — após init, rode `spec`.

📋 Próximo passo:
  - (Opcional) Edite GOD/config.md se quiser mudar o specs_path
  - Rode `init` para criar uma nova task no fluxo v6
```
