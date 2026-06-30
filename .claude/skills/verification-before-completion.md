---
name: verification-before-completion
description: Use before declaring any task complete. Requires evidence before success claims. Covers automated tests, linters, builds, bug fixes, and visual/UI changes where automated verification is impossible.
---

# Verification Before Completion

## Mandato Central

**Evidência antes de declarações. Sempre.**

Nenhuma afirmação de sucesso é permitida sem executar o comando de verificação e confirmar o output real. Isso vale para código, UI, correções de bug, trabalho de subagentes — sem exceção.

---

## O Portão de Verificação

Antes de qualquer declaração de conclusão, execute esta sequência:

1. Identificar qual comando prova a afirmação
2. Executar o comando de forma completa e nova (não reusar output anterior)
3. Ler o output completo e o exit code
4. Confirmar que o output corresponde à afirmação
5. Somente então declarar sucesso, citando a evidência

---

## Verificação por Tipo de Trabalho

### Código com testes automatizados

```
Comando de verificação: npm test / pytest / go test / etc.
Critério: 0 falhas, 0 erros
Evidência exigida: output do test runner com contagem de testes passando
```

**Não basta:** "os testes devem passar" ou "não mudei essa parte"

### Linter / Type check

```
Comando: npm run lint / tsc --noEmit / flake8 / etc.
Critério: 0 erros (warnings podem ser aceitáveis se pré-existentes)
Evidência exigida: output limpo ou diferença explícita do baseline
```

### Build

```
Comando: npm run build / cargo build / etc.
Critério: exit code 0, sem erros de compilação
Evidência exigida: "Build successful" ou equivalente no output
```

### Correção de bug

```
Sequência obrigatória:
1. Escrever teste que reproduz o bug (deve falhar — RED)
2. Aplicar a correção
3. Rodar o teste (deve passar — GREEN)
4. Rodar a suite completa (sem regressões)
Evidência exigida: output mostrando RED antes, GREEN depois
```

### Trabalho de subagente

```
Nunca confie no relatório do subagente sem verificação independente:
- Verificar os commits reais: git log --oneline BASE..HEAD
- Verificar os arquivos alterados: git diff --stat BASE HEAD
- Rodar os testes: não assumir que o subagente rodou
Evidência exigida: sua própria execução dos comandos, não o relato do subagente
```

### Cumprimento de requisitos

```
Método: checklist linha por linha dos requisitos originais
Para cada item:
  - Qual comando ou observação prova que está feito?
  - Execute o comando ou faça a observação agora
  - Marque como completo somente com evidência
Evidência exigida: checklist preenchido com evidência por item
```

---

## Quando Verificação Automatizada é Impossível

Para mudanças visuais (HTML, CSS, UI), deploys, ou integrações externas onde não há teste automatizado disponível:

### Opção A: Verificação visual documentada

Execute um dos seguintes e documente o resultado:

```bash
# Screenshot via Playwright (se disponível)
npx playwright screenshot --url http://localhost:3000 /tmp/before.png
# [fazer mudança]
npx playwright screenshot --url http://localhost:3000 /tmp/after.png

# Ou: abrir no browser e descrever especificamente o que foi observado
```

**Declaração válida:** "Abri no browser, o botão aparece em azul com hover em tom mais escuro, sem texto cortado em mobile 375px."

**Declaração inválida:** "A mudança visual deve estar correta."

### Opção B: Checklist manual explícito

Para cada mudança visual, documente:

```
[ ] Elemento aparece na posição correta em desktop (1280px)
[ ] Elemento aparece corretamente em mobile (375px)
[ ] Nenhum texto cortado ou overflow
[ ] Cores corretas conforme especificação
[ ] Estado de hover/focus funcionando
[ ] Acessibilidade: contraste adequado, labels presentes
```

Execute e marque cada item. Nunca marque sem verificar.

### Opção C: Aceitar limitação explicitamente

Se nenhuma verificação for possível neste momento, declare:

```
"Verificação não foi possível porque [razão específica].
O que foi feito: [descrição exata das mudanças].
Risco residual: [o que pode estar errado sem verificação].
Próximo passo para confirmar: [o que fazer quando possível]."
```

Isso é honesto. O que não é aceitável é declarar "está feito" sem evidência e sem reconhecer a limitação.

---

## Para Deploys e Ambientes Externos

```
Verificação mínima após deploy:
1. URL principal responde: curl -I https://dominio.com → 200
2. Funcionalidade crítica acessível (não apenas "servidor no ar")
3. Nenhum erro óbvio nos logs: verificar logs do servidor
4. Rollback disponível: confirmar que versão anterior pode ser restaurada
```

---

## Frases que Violam Este Protocolo

Qualquer variação das seguintes frases é um red flag — pare e verifique antes de continuar:

- "deve funcionar"
- "provavelmente está correto"
- "parece estar ok"
- "a lógica está certa então..."
- "não mudei essa parte então..."
- "o subagente confirmou que..."
- "satisfeito com o resultado"

---

## Verificação Pós-Compactação de Contexto

Quando o contexto for compactado (conversa longa), o Claude perde memória do que foi verificado. Ao retomar:

1. Não assuma que verificações anteriores ainda valem
2. Re-execute os comandos de verificação para o estado atual do código
3. Consulte o progress ledger (se estiver em subagent-driven-development) para saber o que foi realmente commitado

---

## Checklist de Saída (por tarefa)

Antes de declarar qualquer tarefa completa, responda com evidência:

```
[ ] Testes passando? Comando executado: ___ Output: ___
[ ] Linter limpo? Comando executado: ___ Output: ___
[ ] Build bem-sucedido? Comando executado: ___ Output: ___
[ ] Mudanças visuais verificadas? Método: ___ Observação: ___
[ ] Requisitos cumpridos? Checklist: ___
[ ] Sem regressões em funcionalidades adjacentes? Como verificado: ___
```

Itens sem evidência = itens não completos.
