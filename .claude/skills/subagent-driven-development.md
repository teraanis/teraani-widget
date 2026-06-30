---
name: subagent-driven-development
description: Use when executing implementation plans through specialized subagents within a single session. Covers task dispatch, dual-verdict review, progress ledger format, and context compaction survival.
---

# Subagent-Driven Development

## Conceito Central

Despacha um subagente implementador por tarefa, uma revisão de tarefa após cada implementação, e uma revisão final de todo o branch ao final. Cada subagente recebe apenas o contexto da sua tarefa — sem histórico acumulado da sessão.

**Usar quando:**
- Você tem um plano de implementação finalizado
- As tarefas são majoritariamente independentes
- Quer permanecer na sessão atual (vs. sessões paralelas)

---

## Progress Ledger — Formato Padrão

O ledger é o que sobrevive à compactação de contexto. Sem ele, tarefas completas podem ser re-despachadas por engano.

**Localização fixa:** `.superpowers/sdd/progress.md` na raiz do repositório

**Formato:**
```markdown
# Progress Ledger
Plano: [nome do arquivo de plano]
Iniciado: YYYY-MM-DD HH:MM
Branch: [nome do branch]
Merge base: [commit hash completo — resultado de `git merge-base main HEAD`]

## Tarefas

- [x] Tarefa 1: completa (commits abc1234..def5678, revisão limpa)
- [x] Tarefa 2: completa (commits def5678..ghi9012, revisão limpa, 1 minor registrado)
- [ ] Tarefa 3: pendente
- [ ] Tarefa 4: pendente

## Minors Registrados (para revisão final)
- Tarefa 2: variável `x` poderia ter nome mais descritivo (arquivo: src/utils.js:45)

## Bloqueios / Decisões Pendentes
- [registrar aqui qualquer item que precisou de decisão humana]
```

**Comandos de ledger:**
```bash
# Ler ledger após compactação
cat "$(git rev-parse --show-toplevel)/.superpowers/sdd/progress.md"

# Registrar tarefa completa (substitua N e os hashes)
echo "- [x] Tarefa N: completa (commits BASE..HEAD, revisão limpa)" >> .superpowers/sdd/progress.md
```

**Nunca use HEAD~1 como BASE** — em tarefas com múltiplos commits isso trunca silenciosamente. Sempre grave o hash antes de despachar o implementador.

---

## Seleção de Modelo

Escolha o modelo mais simples que consegue fazer a tarefa — não o mais capaz por padrão:

| Tipo de tarefa | Modelo |
|---------------|--------|
| Tarefa mecânica (1-2 arquivos, spec completa) | Mais barato disponível |
| Tarefa de integração (múltiplos arquivos, coordenação) | Modelo padrão |
| Arquitetura, design, decisões ambíguas | Modelo mais capaz |
| Revisão de tarefa | Modelo padrão (mínimo) |
| Revisão final do branch | Modelo mais capaz |

**Regra prática:** Turnos custam mais que tokens. Um modelo barato que precisa de 3x mais turnos em tarefa complexa sai mais caro.

---

## Handoffs via Arquivo

Nunca cole artefatos diretamente no prompt do subagente. Passe caminhos de arquivo:

| Artefato | Caminho | Gerado por |
|----------|---------|-----------|
| Brief da tarefa | `.superpowers/sdd/task-N-brief.md` | Você (antes do dispatch) |
| Relatório do implementador | `.superpowers/sdd/task-N-report.md` | Instrua o subagente a criar |
| Pacote de revisão (diff) | `.superpowers/sdd/task-N-review.diff` | `git diff BASE HEAD > arquivo` |

---

## Ciclo por Tarefa

### Pré-voo (uma vez, antes da Tarefa 1)

Escaneie o plano completo por:
- Tarefas que se contradizem
- Requisitos que o rubric de revisão trataria como defeito
- Violações de mandato (ex: teste que não assert nada)

Apresente todos os achados em **uma única pergunta** ao humano antes de começar.

### Por Tarefa

**1. Gravar BASE antes de despachar:**
```bash
BASE=$(git rev-parse HEAD)
echo "Base para Tarefa N: $BASE"
```

**2. Criar brief da tarefa:**
```markdown
# Task N Brief: [nome da tarefa]

## Objetivo
[descrição clara do que deve ser implementado]

## Arquivos a criar/modificar
- [lista]

## Interfaces de tarefas anteriores
[exports/tipos/contratos que este implementador deve respeitar]

## Critérios de aceite
- [ ] critério 1
- [ ] critério 2

## Restrições
[o que NÃO fazer]

## Arquivo de relatório
Ao concluir, escreva seu relatório em: .superpowers/sdd/task-N-report.md
```

**3. Despachar implementador:**

O implementador deve responder com um status:
- `DONE` — tarefa completa, relatório escrito
- `DONE_WITH_CONCERNS` — completa mas com ressalvas (descrever)
- `NEEDS_CONTEXT` — precisa de informação adicional (especificar o quê)
- `BLOCKED` — não consegue avançar (especificar razão)

Para `NEEDS_CONTEXT`: fornecer a informação e redespatchar.
Para `BLOCKED`: avaliar se modelo mais capaz resolve, ou dividir a tarefa.

**4. Gerar pacote de revisão:**
```bash
git diff $BASE HEAD > .superpowers/sdd/task-N-review.diff
```

**5. Despachar revisor de tarefa com dois vereditos obrigatórios:**

O revisor deve retornar explicitamente:
- **Conformidade com spec:** ✅ conforme / ❌ não conforme
- **Qualidade de código:** aprovado / precisa de ajuste

Para itens "não consigo verificar": você resolve usando contexto do plano antes de marcar completo.

**6. Loop de correção (se necessário):**

Para achados Críticos ou Importantes:
- Despachar subagente de correção com lista completa de achados
- Exigir que o relatório de correção inclua: arquivos de teste cobrindo o problema, comando executado, output
- Re-revisar antes de marcar completo

**7. Registrar no ledger:**
```bash
HEAD=$(git rev-parse HEAD | cut -c1-7)
BASE_SHORT=$(echo $BASE | cut -c1-7)
# Editar progress.md: marcar tarefa N como completa com os hashes
```

---

## Revisão Final do Branch

Após todas as tarefas:

```bash
MERGE_BASE=$(git merge-base main HEAD)
git diff $MERGE_BASE HEAD > .superpowers/sdd/final-review.diff
```

Despachar revisor final no **modelo mais capaz** com:
- O diff completo
- A lista de minors registrados no ledger
- Os critérios de aceite do plano original

Se achados emergirem: despachar **um único** subagente de correção com a lista completa (não um por achado).

Após revisão limpa: usar `finishing-a-development-branch`.

---

## Sobrevivência à Compactação de Contexto

O Claude compacta o contexto automaticamente em sessões longas. A memória da conversa desaparece. O código e os commits persistem; o ledger também.

**Ao retomar após compactação:**

1. Ler o ledger:
   ```bash
   cat .superpowers/sdd/progress.md
   ```
2. Verificar o log do git para confirmar o que foi commitado:
   ```bash
   git log --oneline $(git merge-base main HEAD)..HEAD
   ```
3. Retomar na primeira tarefa marcada como pendente
4. Nunca re-despachar tarefa marcada como completa no ledger

**Preparação antes de sessão longa (preventivo):**

Antes de começar uma série de tarefas longas, crie o ledger e grave o merge base. Se a compactação vier, você tem o ponto de retomada.

---

## Regras Absolutas

- Nunca trabalhe direto em main/master sem consentimento explícito
- Nunca pule um dos dois vereditos (spec E qualidade — ambos obrigatórios)
- Nunca avance com achados Críticos ou Importantes não resolvidos
- Nunca despache múltiplos implementadores em paralelo
- Nunca faça o subagente ler o plano inteiro (use o brief)
- Nunca instrua o revisor sobre o que flagrar ou pré-classifique severidade para ele
- Nunca despache revisor sem o arquivo diff
- Nunca re-despache tarefa marcada como completa no ledger — cheque sempre após compactação

---

## Tratamento de Achados do Revisor

Se um achado conflita com uma decisão do plano: apresente **ambos** (achado + texto do plano) ao humano para arbitragem. Nunca pré-julgue nem ignore.

Para achados Minors: registre no ledger na seção "Minors Registrados" e aponte o revisor final para essa lista no pré-merge.
