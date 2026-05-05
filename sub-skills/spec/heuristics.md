# Heurísticas da skill `spec`

> Arquivo separado lido pelo passo 5.5 da skill `spec` (análise pré-Q&A). Contém detectores que varrem o input bruto procurando excessos (HOW dentro de WHAT) e gaps (WHAT que falta), além do checklist de self-validação rodado antes de delegar pro `review --spec`.

## 1. Detecção de excessos (HOW dentro do WHAT)

Sinais de "implementação no ticket" que devem ser **movidos pra seção `## Notas técnicas (input pro plan)`** ou removidos.

| Sinal | Como detectar | Ação |
|-------|---------------|------|
| Bloco de código | regex pra fences markdown ` ``` ` seguidos de linguagem ou de `def `, `function `, `class `, `const ` dentro do bloco | Marcar como "candidato a sair pro plan" → oferecer mover pra `## Notas técnicas` |
| Schema técnico detalhado | menção a "polimórfico", "índice", "FK", `BIGINT`, `VARCHAR`, `NOT NULL` | Marcar como "candidato a sair pro plan" |
| Nome de framework/lib | match na lista de palavras-flag (seção 4 abaixo) em REQ/AC/Objetivo | Marcar como **vazamento** — oferecer reescrever sem mencionar |
| Pseudo-código embutido | `def método ... end`, `function nome() { ... }`, `class Foo { }` em prosa | Marcar como "candidato a sair pro plan" |
| Nomes de classe/método específicos | `Moderation#moderate_image`, `UserService.create()` | OK como **referência** (caminho:linha em "Contexto"), NÃO OK como prescrição em REQ/AC |
| Decisão de banco/storage | "tabela X", "coluna Y", "redis", "kafka" no corpo da spec | Marcar como vazamento → mover ou remover |

**Regra de ouro:** se o sinal está na seção "Contexto" descrevendo o **estado atual** do sistema, OK. Se está prescrevendo **como fazer**, vai pro plan.

## 2. Detecção de gaps (WHAT que falta)

Pra cada gap detectado, a Q&A do passo 6 deve perguntar **especificamente sobre ele**. Se nenhum gap, a Q&A pula o bloco inteiro.

| Gap | Como detectar | Pergunta a fazer no passo 6 |
|-----|---------------|------------------------------|
| **Objetivo vago** | Texto curto sem substantivo de ação claro, ou múltiplos objetivos misturados | "Qual o problema **único** que esta task resolve?" |
| **Ator humano não nomeado** | Menciona "admin", "usuário", "PM" sem qualificação | "Quem efetivamente faz/consome? Time específico? Há SLA implícito?" |
| **Não-objetivos ausentes** | Nenhuma menção explícita do que NÃO entra | "Há algo no escopo aparente que você quer deixar claro que NÃO entra?" |
| **NFRs ausentes — Performance** | Nenhuma menção a latência, throughput, janela | "Há limite de performance? (latência, throughput, janela de cron)" |
| **NFRs ausentes — LGPD/PII** | PII aparente (telefone, email, CPF) sem menção a privacidade | "Há dado pessoal envolvido? Como tratar retenção?" |
| **NFRs ausentes — Observabilidade** | Sem menção a log/métrica/alerta | "Algo precisa ser logado/alertado?" |
| **NFRs ausentes — Auditabilidade** | Operação financeira/admin sem menção a auditoria | "Quem fez o quê precisa ficar registrado?" |
| **Cenários de erro ausentes** | Sem menção a "se X falha", "fallback", "timeout", "down" | Listar dependências externas detectadas e perguntar comportamento de cada falha |
| **Comportamento não-testável** | Tudo descrito como ação ("X é gravado em Y") sem contexto disparador (`WHEN`) | Reescrever em EARS na geração; perguntar só se ambíguo |
| **Critérios sem ID estável** | Critérios em prosa, sem `AC-NNN.N` | Numerar na geração; não precisa perguntar |

**Regra de ouro:** não perguntar o que já está claro no input. O passo 5.5 produz lista de gaps; a Q&A só toca neles.

## 3. Detecção de tamanho — task simples vs feature

Sinais de que é **feature com subtasks** (precisaria de spec pai + N specs filhas):

| Sinal | Peso |
|-------|------|
| Mais de 1 sistema/repo afetado (api + web, web + worker, etc.) | forte |
| Mais de 1 modelo/tabela criado/alterado | forte |
| Mais de 1 endpoint criado | médio |
| Migração de dados envolvida | forte |
| Texto > ~150 linhas | médio |
| Subtasks já declaradas no input (Jira ou texto livre) | forte |
| Mais de ~5 REQs distintos identificáveis | médio |

**Threshold:** 2+ sinais "forte" OU 3+ sinais "médio" → feature.

Quando detecta feature, a skill oferece (passo 6.5 da spec):
- (a) Gerar spec pai + N specs de subtasks (cria múltiplos arquivos em `<specs_path>/tasks/`)
- (b) Continuar como spec única (você sabe o que está fazendo)
- (c) Abortar e usar `init-tree` se a árvore Jira já existe

## 4. Lista de palavras-flag de framework leak

Match **case-insensitive** em REQs/ACs/Objetivo (NÃO em "Contexto" nem em "Notas técnicas"). Se detectado, marcar como vazamento e oferecer reescrever.

**Frontend frameworks:**
- React, Vue, Angular, Svelte, Solid, Preact, Next.js, Nuxt, Remix, Astro
- useState, useEffect, useMemo, useCallback, useReducer, useContext (hooks)
- Redux, Zustand, MobX, Recoil, Jotai, RxJS, NgRx
- Tailwind, styled-components, emotion, MUI, Chakra, Ant Design

**Backend frameworks:**
- Rails, ActiveRecord, Django, Flask, Laravel, Spring, Express, Fastify, NestJS, FastAPI
- Sidekiq, Resque, Celery, Bull, Hangfire (job queues)

**Databases / Storage:**
- Postgres, PostgreSQL, MySQL, MariaDB, SQLite, MongoDB, Redis, Memcached, DynamoDB, Cassandra, Elasticsearch, Solr, ClickHouse, BigQuery, Snowflake

**Message brokers / Streaming:**
- Kafka, RabbitMQ, NATS, SQS, SNS, Pub/Sub, Kinesis

**Cloud / Infra:**
- AWS, GCP, Azure, Heroku, Vercel, Netlify, Cloudflare, Fly.io, Render
- S3, EC2, Lambda, Cloud Run, Cloud Functions

**Linguagens explícitas em comportamento:**
- "deve usar Python", "implementar em Go", "código TypeScript" — REQ não deve impor linguagem

**Exceções (NÃO marcar como vazamento):**
- Quando o nome aparece em `## Contexto` descrevendo estado atual: "o sistema atual usa Postgres" — OK.
- Quando aparece em `## Notas técnicas (input pro plan)`: OK — é justamente onde mora.
- Quando é parte do nome do produto/projeto (ex: "API do Stripe" como integração externa): OK como referência funcional.

## 5. Self-validação inline (checklist final antes de delegar pro review)

Rodar antes do passo 9 (review --spec). Se falha, **tenta corrigir inline** (sem precisar do subagent). Se não conseguir, sinaliza pro usuário antes de seguir.

| Check | Severidade | Ação se falha |
|-------|-----------|----------------|
| Cada REQ tem ao menos 1 AC | bloqueia | Pedir AC ao usuário (loop até cobrir) |
| ACs têm ID estável (`AC-NNN.N`) | corrige inline | Numerar automaticamente |
| Sem palavras-tabu sem métrica nos ACs ("rápido", "fácil", "intuitivo", "escalável", "seguro" sem qualificação) | bloqueia | Reescrever ou perguntar métrica |
| Sem framework leak em REQs/ACs/Objetivo | corrige inline ou pergunta | Mover pra "Notas técnicas" se usuário concorda |
| Cada AC tem cenário associado (happy/edge/erro) | sugere | Sugerir cenário; não bloqueia |
| Seção NFRs presente (mesmo que com "não aplicável") | corrige inline | Adicionar com placeholders "não aplicável" |
| Não-objetivos declarados | sugere | Sugerir; não bloqueia se realmente vazio |
| Ator humano nomeado | sugere | Sugerir; não bloqueia |
| `## Input bruto` preservado sem edição | bloqueia | Restaurar do estado original |
| Para feature: subtasks têm escopo independente | sugere | Realocar entre subtasks |
| Para feature: ordem de subtasks reflete dependência real | sugere | Pedir ao usuário |

**Regra de ouro:** self-validação corrige o trivial; review --spec (subagent isolado) faz peer-review semântica profunda. As duas se complementam, não substituem.

---

## Notas de implementação pra skill `spec`

- A skill carrega este arquivo no passo 5.5 (após coletar contexto, antes da Q&A).
- Gera em memória uma lista estruturada de "achados" (excessos + gaps + tamanho).
- Passo 6 (Q&A) consome a lista e pergunta apenas sobre o que faltou.
- Passo 6.5 (novo, feature split): se tamanho = feature, oferece quebrar.
- Passo 8.5 (novo, self-val): roda checklist inline.
- Passo 9 (review): continua delegando pro `review --spec` (subagent) — peer-review independente do que self-validação fez.
