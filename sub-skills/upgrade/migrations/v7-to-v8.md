# Migração v7 → v8

## Objetivo

A v8 introduz **rastreabilidade AC × validação**: cada AC declarado na spec ganha um link explícito até o que valida ele (teste automatizado anotado por comentário leve, ou validação manual em `coverage.md`). A matriz é injetada no PR description automaticamente, alimentando review e auditoria.

Mudanças estruturais:

- **Sub-skill nova `coverage`** — parseia comentários `// covers: AC-X` em testes + lê `coverage.md` (validações manuais) → gera matriz AC × validação. Pode ser invocada manualmente ou pelo `pack-up`/`review`. Skill de uso interno do GOD (executada pelo Claude na máquina do dev) — não tem CLI standalone nem se integra com pipelines de CI.
- **`implement` ganha passo 5.5** — após escrever testes, anota `// covers: AC-X` no comentário acima do teste. Pra ACs sem teste (FE), oferece registrar em `coverage.md`.
- **`pack-up` ganha passo 4.5** — chama `coverage` antes de criar PR, alerta sobre ACs órfãos, injeta tabela markdown no PR description.
- **`review --execution` ganha eixo de cobertura** — cruza ACs da spec com matriz da `coverage`. ACs órfãos viram "Ajustes necessários", nunca "Reprovado".
- **Arquivo opcional `GOD/tasks/{cod}/coverage.md`** — criado sob demanda pela skill `coverage` ou `implement`. Contém validações manuais + cache da matriz parseada.

## Arquivos afetados

**Atualizado:**
- `GOD/VERSION` — conteúdo muda de `v7` para `v8`

**Inalterados (nenhuma reescrita automática):**
- `GOD/config.md`
- `GOD/knowledge.md`
- `GOD/patterns.md`
- `GOD/learned-patterns.md`
- `GOD/hooks.md`
- `GOD/tasks/` (todas as tasks, `status.md`, `description.md`, `plan.md`, `changelog.md`)
- `GOD/tasks/.archived/` (se existir)
- `<specs_path>/` (especs existentes não são alteradas)

**Novo (criado sob demanda, não pela migração):**
- `GOD/tasks/{cod}/coverage.md` — criado pela skill `coverage` ou `implement` quando pela primeira vez precisar registrar validação manual ou cachear matriz.

**Mudanças de contrato (não de arquivos):**
- Sub-skill nova `coverage` disponível.
- `implement` ganha passo 5.5 — anotação `// covers: AC-X` em testes novos/modificados.
- `pack-up` ganha passo 4.5 — chama `coverage` e injeta tabela no PR.
- `review --execution` ganha eixo de cobertura no relatório.
- ACs órfãos no PR são visíveis (⚠️) mas não bloqueiam — skill é tolerante por design e roda apenas no fluxo interno do GOD (máquina do dev, via Claude).

## Riscos conhecidos

- **Tasks pré-v6 sem spec.** A skill `coverage` retorna "não aplicável" silenciosamente — `pack-up` e `review --execution` pulam o eixo de cobertura. Sem regressão.
- **Tasks v6/v7 com spec mas sem testes anotados.** Todos os ACs aparecerão como ⚠️ órfãos na primeira execução pós-upgrade. Não bloqueia — apenas mostra estado real. Dev pode ir anotando retroativamente ou registrar como manual.
- **Comentário `// covers:` em sintaxe não suportada.** A skill `coverage` tem regex unificada cobrindo `//`, `#` e `/* */` — cobre TS/JS, Ruby, Python, Go, Java, C#. Linguagens fora dessa lista exigem extensão da regex (futuro).
- **Anotação manual entra em conflito com teste anotado.** Se um AC aparece no `coverage.md` E como `// covers:` em algum teste, a matriz mostra ambos (✅👁) — não é erro, é informação enriquecida.
- **Nenhum risco de perda de dados.** A migração só atualiza VERSION; nenhum arquivo é removido nem reescrito.

---

## Passos da migração

### 1. Verificar pré-condições

- Confirmar que `GOD/VERSION` existe e contém `v7`.
- Se não existir ou estiver em versão anterior, abortar — o usuário deve rodar as migrações anteriores primeiro.

### 2. Atualizar VERSION

Sobrescrever o conteúdo de `GOD/VERSION` com:

```
v8
```

### 3. Não tocar em tasks ativas

A migração **deliberadamente não modifica** nenhum arquivo dentro de `GOD/tasks/{cod}/` ou `<specs_path>/`. Os comportamentos novos (anotação AC↔teste pelo `implement`, geração de matriz pelo `coverage`, injeção de tabela pelo `pack-up`) só atuam em execuções futuras.

Tasks já em `phase: implementing` ou posterior não terão anotação retroativa de `// covers:` nos testes existentes — isso fica como trabalho manual opcional do dev. ACs dessas tasks aparecerão como órfãos na matriz (⚠️) até serem anotados.

### 4. Relatório final

```
✅ Migração v7 → v8 concluída!

VERSION atualizado: v8

Nenhum arquivo modificado além do VERSION.

Novidades disponíveis:
  📊 coverage (skill nova)         — matriz AC × validação por task
  📝 implement passo 5.5           — anota `// covers: AC-X` nos testes escritos
  🔗 pack-up passo 4.5             — chama coverage, injeta tabela no PR
  🔍 review --execution            — ganha eixo de cobertura (alerta, não bloqueia)
  📂 coverage.md por task          — criado sob demanda pra validações manuais

Tasks em andamento continuam compatíveis. ACs sem `// covers:` anotado aparecerão
como órfãos (⚠️) na matriz até serem anotados ou registrados como manuais.

Próximos passos sugeridos:
  - Em tasks novas, o `implement` vai anotar automaticamente os testes que escrever
  - Em tasks v7 já em `implementing`, anote retroativamente os testes existentes
    com `// covers: AC-X` se quiser cobertura completa no PR
  - Rode `coverage PROJ-XXX --format terminal` em qualquer task pra ver estado atual
```
