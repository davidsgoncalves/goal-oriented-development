---
name: install
description: |
  Configura a arquitetura de pastas e arquivos GOD no projeto do usuário e verifica integrações MCP opcionais. Use quando o usuário mencionar: "instalar god", "install", "configurar god", "setup god", ou na primeira execução do framework.
tools: Read, Glob, Grep, Bash, Edit, Write
---

# Install — Sub-skill de Instalação

> Configura a arquitetura de pastas e arquivos GOD no projeto do usuário e verifica integrações MCP opcionais.

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

## Instruções

Quando o usuário invocar esta skill, execute os seguintes passos **na ordem**:

### 0. Verificar se já existe instalação anterior

Antes de qualquer coisa, verificar o estado das pastas no diretório raiz.

- **Se `GOD/` existe e tem `GOD/VERSION` com conteúdo `v9`:** informar que o projeto já está instalado na versão atual e encerrar.
- **Se `GOD/` existe mas `GOD/VERSION` não existe (ou aponta pra versão anterior):** informar que é uma instalação de versão antiga e sugerir rodar a skill `upgrade` em vez de reinstalar. Não sobrescrever arquivos existentes.
- **Se `GOD/` não existe mas `GDD/` existe:** instalação legada da skill GDD detectada. Informar o usuário e sugerir rodar `upgrade` (ou `migrate`) para migrar automaticamente de GDD para GOD. Não instalar do zero — os dados do usuário (tasks, knowledge, patterns, hooks) serão preservados pela migração. Encerrar sem criar nada.
- **Se nem `GOD/` nem `GDD/` existem:** prosseguir com a instalação.

### 1. Criar estrutura GOD

No diretório raiz do projeto do usuário, crie a seguinte estrutura:

```
GOD/
├── VERSION
├── config.md
├── knowledge.md
├── patterns.md
├── learned-patterns.md
├── hooks.md
└── tasks/
```

- `VERSION` — arquivo com conteúdo `v9` (uma linha, sem espaços)
- `config.md` — configuração local do GOD nesse projeto (ver passo 1.5 abaixo). Contém `specs_path` (onde a spec da task vai morar)
- `knowledge.md` — criado com template padrão (ver seção abaixo)
- `patterns.md` — criado com template padrão (ver seção abaixo)
- `learned-patterns.md` — criado com template padrão (ver seção abaixo)
- `hooks.md` — criado com template padrão (ver seção abaixo)
- `tasks/` — pasta vazia para armazenar tasks do projeto

### 1.5. Configurar `specs_path` e criar `GOD/config.md`

A partir da v6, a spec de cada task vive **fora** da pasta `GOD/`. Motivo: a spec é ativo canônico do produto — committada e lida por todos —, enquanto a pasta `GOD/` é workflow individual e pode ser ignorada do git.

**Conceito-chave:** o `specs_path` é o **root do repo de specs**, não a pasta direto onde as tasks são escritas. O GOD escreve em `<specs_path>/tasks/<cod>.md`. Subpastas como `domains/` e `flows/` são reservadas pra skills futuras (feature specs eternas, fluxos cross-feature) — não criadas agora.

#### 1.5.1. Detectar contexto e apresentar opções

Antes de perguntar, **detectar o contexto** rodando `git rev-parse --show-toplevel` no diretório atual:

- **Sucesso (está dentro de repo git)** → contexto **single-project**: dentro de um repositório.
- **Falha** → contexto **workspace multi-project**: pasta-mãe contendo múltiplos repos.

Apresentar opções calibradas pelo contexto:

**Se single-project:**

```
📐 Onde guardar as specs das tasks?

A spec é artefato canônico do produto — committável separadamente, lida por todos.

Opções:
  1. docs/specs/                                  (default — dentro deste repo)
  2. ../<workspace>/<nome-do-repo-de-specs>/      (repo dedicado ao lado deste, no workspace pai)
  3. <path-absoluto>                              (qualquer outro lugar — você digita)
  4. <outro-relativo>                             (qualquer outro path relativo a aqui)

Escolha (1/2/3/4) ou tecle Enter pra default:
```

**Se multi-project workspace:**

```
📐 Onde guardar as specs das tasks?

Detectei que você está num workspace multi-project (pasta sem .git contendo repos).
Recomendado: repo dedicado de specs ao lado dos repos de código.

Opções:
  1. ./<nome-do-repo-de-specs>/    (repo dedicado neste workspace — recomendado)
  2. <path-absoluto>               (qualquer outro lugar — você digita)
  3. <outro-relativo>              (qualquer outro path relativo a aqui)

Escolha (1/2/3) ou tecle Enter pra opção 1:
```

#### 1.5.2. Resolver e criar a estrutura

Com o `specs_path` decidido, executar:

1. **Se a pasta não existe**: criar com `mkdir -p <specs_path>`.
2. **Se a pasta existe e não é repo git**: perguntar `"Inicializar como repo git separado? (s/n)"`. Se sim, rodar `git init` lá. Se não, prosseguir como pasta normal.
3. **Se a pasta existe e já é repo git**: usar como está. Não tocar.
4. **Criar subpasta `tasks/`** dentro do `specs_path` (`mkdir -p <specs_path>/tasks/`).
5. **Criar README mínimo** em `<specs_path>/README.md` se não existir (ver template abaixo). Se já existir, não tocar.
6. **Não criar `domains/` nem `flows/` agora** — reservados pra skills futuras escreverem sob demanda.

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

#### 1.5.3. Criar `GOD/config.md`

Conteúdo:

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
> - Repo separado em qualquer lugar: `/Users/eu/projetos/<workspace>/<repo-de-specs>/`

## publish_spec_default_target

Default usado pela sub-skill `publish-spec` quando `--target` não é passado.
Aceita um único target ou múltiplos separados por vírgula.

Valores conhecidos: `jira`, `slack`, `stdout`, `notion`. Targets customizados
exigem definição em `GOD/hooks.md` como `# publish-spec target: <nome>`.

(deixe vazio pra usar `stdout` como default original)

> Exemplos:
> - `jira` — toda execução de `publish-spec` sem flag publica no Jira
> - `jira, slack` — publica nos dois ao mesmo tempo
> - (vazio) — comportamento original: `stdout`
```

Substituir `{specs_path}` pelo valor decidido. A seção `publish_spec_default_target` fica vazia por padrão (retrocompat). Se o usuário inicializou como repo git separado nesse passo, anotar isso no relatório final.

### 2. Preencher template do `VERSION`

Conteúdo exato:

```
v9
```

### 3. Preencher template do `knowledge.md`

O arquivo `GOD/knowledge.md` deve ser criado com o seguinte template:

```markdown
# Knowledge — Registro de Tasks

> Registro de tasks finalizadas com referências de commit e aprendizados. Usado pelo `init` para encontrar tasks semelhantes e pelo `plan` para aproveitar decisões anteriores. **Escrito apenas pela skill `learn`** — nenhuma outra skill modifica este arquivo.

## Tasks finalizadas

<!-- Formato:
### {cod-da-task} — {breve descrição}
- **commits:** {hash1}, {hash2}, ...
- **arquivos principais:** {lista dos arquivos mais relevantes}
- **aprendizados:** {o que foi aprendido, decisões tomadas, armadilhas evitadas}
-->
```

### 4. Preencher template do `patterns.md`

O arquivo `GOD/patterns.md` deve ser criado com o seguinte template para o usuário preencher:

```markdown
# Patterns — Convenções do projeto

> Este arquivo define **apenas os padrões** do projeto: branches, commits, PRs e filtros auxiliares. Lido pelas skills `plan` (branch inicial + padrão), `pack-up` (commit + PR) e `init-tree` (status Jira a ignorar).
> Ações executáveis (criar PR em draft, rodar testes, notificar canais, atualizar tickets) ficam no `hooks.md`.

## Branch inicial
Descreva aqui o branch inicial (ex: `main`, `master`, `develop`).
Se o repositório contém múltiplos projetos com branches diferentes, liste cada um.

Exemplo:
- projeto-web — `develop`
- projeto-api — `main`

## Padrão de nome de branch
Descreva o padrão esperado para nomes de branches de task.
Ex: `task/<cod-da-task>/<descrição-em-ingles-kebab-case>`

## Padrão de mensagem de commit
Descreva o formato esperado do commit (header, body, footer, idioma, tipos permitidos).

## Padrão de mensagem de PR
Descreva o formato do título e corpo do PR, idioma e seções obrigatórias.

## Status Jira a ignorar em batch
Lista de status do Jira que `init-tree` deve pular ao processar folhas (subtasks) em lote. Se a seção estiver ausente ou vazia, o default é: `Done`, `Cancelled`, `Closed`, `Resolved`, `Won't Do`.

Exemplo (um por linha, nome exato como aparece no Jira):
- Done
- Cancelled
- Closed
- Resolved
- Won't Do
```

### 4.5. Preencher template do `learned-patterns.md`

O arquivo `GOD/learned-patterns.md` deve ser criado com o seguinte template:

```markdown
# Learned Patterns — Regras aprendidas nas tasks

> Regras de estilo, convenções e armadilhas registradas após a revisão de PR de tasks anteriores. Lido pelo `implement` logo após a escrita de código (passo de verificação contra padrões) e escrito pelo `learn` após a revisão do PR.
>
> **Escrito apenas pela skill `learn`** — nenhuma outra skill modifica o conteúdo.
>
> **Escopos** (cada regra tem um):
> - `geral` — aplica-se a todos os projetos.
> - `linguagem: <lang>` — aplica-se a todos os projetos naquela linguagem.
> - `projeto: <nome>` — aplica-se somente àquele projeto.
>
> **Convenção:** adicionar novas regras ao final da lista, numeradas. Não remover, apenas marcar como revogada se necessário (`~~riscado~~` + motivo).

---

<!-- Formato de cada regra:

## N. Título curto da regra

**Escopo:** <geral | linguagem: X | projeto: Y>

Descrição da regra.

**Por quê:** motivo — contexto que permite julgar casos de borda.

**Como aplicar:** quando/onde a regra incide. Pode incluir blocos "Bom" e "Ruim" com exemplos curtos ilustrando a regra (não são exemplos específicos de uma task — são ilustrações da regra).
-->
```

### 5. Preencher template do `hooks.md`

O arquivo `GOD/hooks.md` deve ser criado com o seguinte template:

```markdown
# Hooks — Pontos de extensão por step

> Cada seção abaixo é um hook opcional. Se você quiser que algo seja executado antes ou depois de um step do fluxo, escreva aqui em linguagem natural — a skill correspondente vai ler e executar.
> Se não quer nada nesse hook, deixe o valor `skip-hook`.
>
> Apenas os steps do fluxo principal têm hooks: `init`, `spec`, `plan`, `implement`, `pack-up`.
> Ferramentas auxiliares (`learn`, `update-plan`, `review`, `status`, `publish-spec`) não têm hooks.

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

# after plan
skip-hook

# before implement
skip-hook

# after implement
skip-hook

# before pack-up
skip-hook

# after pack-up
skip-hook
```

**Exemplos comuns de uso pro `after spec`** (preencha quando quiser ativar):

- Postar comentário no ticket Jira da task com link da spec, contagem de REQs/ACs e aviso "se algum critério não bate, comente aqui antes do dev começar"
- Postar mensagem em canal Slack `#produto` com resumo da spec
- Linkar URL da spec no card Linear/Trello correspondente

Após criar, informe ao usuário que ele deve preencher o `patterns.md` com as convenções do projeto e, opcionalmente, preencher os hooks no `hooks.md`.

### 6. Gitignore (opcional)

Pergunte ao usuário se deseja adicionar a pasta `GOD/` ao `.gitignore` do projeto.

- **Se sim:** adicione `GOD/` ao `.gitignore` existente (ou crie o arquivo se não existir)
- **Se não:** siga para o próximo passo

### 7. Verificar integrações opcionais

Verifique o que está disponível no ambiente do usuário:

- **Figma** (`claude.ai Figma` MCP) — verificar se está disponível e autenticado. Melhora `plan` e `implement` com análise de design.
- **Jira/Atlassian** (`claude.ai Atlassian` MCP) — verificar se está disponível e autenticado. Permite que `plan` busque dados da task automaticamente.
- **GitHub CLI (`gh`)** — executar `gh --version` e `gh auth status` para verificar instalação e autenticação. É **altamente recomendado** porque:
  - O `pack-up` usa `gh pr create` para abrir PRs
  - O `clean-up` usa `gh pr view` para verificar status de merge dos PRs

Nenhuma dessas integrações é obrigatória, mas sem `gh` a experiência do `pack-up` e do `clean-up` fica degradada (criação manual de PR, verificação manual de merge).

### 8. Reportar resultado

Montar a resposta listando o que está ok e o que está faltando:

```
✅ Instalação v9 concluída! Estrutura GOD criada.

📐 Repo de specs configurado: {specs_path}
   • Pasta {criada / já existia}
   • Estrutura {tasks/ criada}
   • README {criado / já existia}
   • git: {repo separado inicializado / repo já existente / pasta dentro do projeto / pasta sem git}

🔌 Integrações:
  [✓/✗] Figma MCP — {status}
  [✓/✗] Jira (Atlassian) MCP — {status}
  [✓/✗] GitHub CLI (gh) — {status}

{Se algum item estiver ✗, sugerir conexão/instalação com link apropriado:}
  - Figma: conectar MCP em Claude
  - Jira: conectar MCP em Claude
  - gh: https://cli.github.com (depois `gh auth login`)

📋 Próximos passos:
  1. Preencha `GOD/patterns.md` com as convenções do seu projeto
  2. (Opcional) Preencha slots de `GOD/hooks.md` que você quer customizar
  3. `GOD/learned-patterns.md` começa vazio — a skill `learn` vai preenchê-lo após a revisão de PR
  4. Rode `spec` para iniciar sua primeira task (v9 spec-first). Fluxo: spec → [publish-spec] → init → plan → implement → pack-up. Pra mudança trivial (typo, copy), pule direto pra `init {cod} --type=trivial`.
```

---

## Guard-rails

- **Esta skill não escreve em `GOD/knowledge.md` nem em `GOD/learned-patterns.md`** (apenas cria os arquivos vazios com template). Atualizações de conteúdo são responsabilidade exclusiva de `learn`.
- **Esta skill não sobrescreve instalações existentes.** Se detectar uma versão anterior já instalada, orienta o usuário a rodar `upgrade` ou nenhuma ação.
