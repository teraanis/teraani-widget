---
name: security-hardening
description: Use when auditing a website or app for security vulnerabilities and fixing them. Covers OWASP Top 10, exposed credentials, HTTP headers, input validation, and dependency vulnerabilities.
---

# Security Hardening

## Princípio Central

**Nenhuma declaração de "seguro" sem evidência. Sempre audite, categorize, corrija e verifique — nesta ordem.**

Segurança não é uma checagem única. É um ciclo: encontrar → classificar por severidade → corrigir → confirmar que a correção funciona → registrar o que foi verificado.

---

## Quando usar esta skill

- Antes de qualquer deploy em produção
- Após adicionar autenticação, formulários, upload de arquivos, ou integrações externas
- Após o hook de stop detectar mudanças de código (`/security-review`)
- Ao receber um relatório de vulnerabilidade externo

---

## Fase 1: Auditoria — O que verificar

### 1.1 Injeção (XSS / SQL / Command Injection)

**XSS (Cross-Site Scripting):**
```
Buscar: innerHTML, document.write, eval(), setTimeout(string)
Buscar: dados do usuário inseridos diretamente no DOM sem sanitização
Verificar: todos os inputs de formulário, parâmetros de URL, headers refletidos
```

**SQL Injection:**
```
Buscar: queries com concatenação de string + variável de usuário
Padrão perigoso: "SELECT * FROM users WHERE id = " + userId
Padrão seguro: queries parametrizadas / prepared statements
```

**Command Injection:**
```
Buscar: exec(), system(), child_process com input do usuário
Regra: nunca passe dados de usuário diretamente para comandos shell
```

### 1.2 Autenticação e Sessão

- Senhas com hash fraco (MD5, SHA1 simples) → deve usar bcrypt/argon2
- Tokens de sessão previsíveis ou curtos demais (< 128 bits)
- Ausência de expiração de sessão
- Cookies sem flags `HttpOnly`, `Secure`, `SameSite`
- Reset de senha por link sem expiração ou token de uso único

### 1.3 Controle de Acesso

- Rotas autenticadas acessíveis sem token (testar diretamente via curl/fetch)
- IDs sequenciais expostos em URLs (`/user/123`) sem verificação de propriedade
- Endpoints admin sem verificação de papel/role
- Dados de outros usuários retornados por falta de filtro no backend

### 1.4 Dados Sensíveis Expostos

```
Buscar nos arquivos: API_KEY, SECRET, PASSWORD, TOKEN fora de .env
Buscar em git history: git log -S "password" --all
Verificar: .env commitado acidentalmente
Verificar: console.log() com dados de usuário em produção
Verificar: stack traces detalhados expostos ao cliente
```

### 1.5 Headers de Segurança HTTP

Verificar presença dos seguintes headers nas respostas:
```
Content-Security-Policy      → bloqueia XSS inline
X-Frame-Options              → bloqueia clickjacking
X-Content-Type-Options       → bloqueia MIME sniffing
Strict-Transport-Security    → força HTTPS
Referrer-Policy              → controla vazamento de URL
Permissions-Policy           → restringe APIs do browser
```

**Como verificar:**
```bash
curl -I https://seu-site.com | grep -i "content-security\|x-frame\|x-content\|strict-transport"
```

### 1.6 CSRF (Cross-Site Request Forgery)

- Formulários que alteram estado sem token CSRF
- APIs que aceitam requests de qualquer origem sem verificar `Origin`/`Referer`
- Cookies de sessão sem `SameSite=Strict` ou `SameSite=Lax`

### 1.7 Dependências Vulneráveis

```bash
# Node.js
npm audit

# Python
pip-audit

# Verificação manual: checar versões no package.json/requirements.txt
# contra advisories conhecidos
```

### 1.8 Upload de Arquivos

- Validação de tipo apenas no cliente (bypassável) → validar no servidor
- Sem limite de tamanho de arquivo
- Arquivos salvos com nome original do usuário (path traversal)
- Arquivos executáveis aceitos (`.php`, `.sh`, `.exe`)
- Arquivos servidos diretamente do servidor sem Content-Disposition

---

## Fase 2: Classificação por Severidade

Categorize cada vulnerabilidade encontrada antes de corrigir:

| Severidade | Critério | Ação |
|------------|----------|------|
| **Crítica** | Execução remota, vazamento de dados em massa, bypass de autenticação | Corrigir imediatamente, não fazer deploy |
| **Alta** | XSS armazenado, IDOR, credenciais expostas | Corrigir antes do próximo deploy |
| **Média** | Missing headers, CSRF em ações de baixo risco, senhas fracas | Corrigir no ciclo atual |
| **Baixa** | Informações de versão expostas, headers informativos | Registrar, corrigir quando conveniente |

**Regra:** Nunca faça deploy com achados Críticos ou Altos não resolvidos.

---

## Fase 3: Correção

### Padrões de correção seguros

**XSS — sanitização de output:**
```javascript
// ERRADO
element.innerHTML = userInput;

// CERTO
element.textContent = userInput;
// ou com biblioteca de sanitização:
element.innerHTML = DOMPurify.sanitize(userInput);
```

**Headers de segurança (HTML meta tags como fallback):**
```html
<meta http-equiv="Content-Security-Policy" content="default-src 'self'">
<meta http-equiv="X-Content-Type-Options" content="nosniff">
```

**Cookies seguros:**
```javascript
document.cookie = "session=abc; HttpOnly; Secure; SameSite=Strict; Path=/";
```

**Credenciais — remover do código:**
```bash
# Se commitado acidentalmente:
git filter-branch ou BFG Repo Cleaner
# Rotacionar a credencial imediatamente (o histórico não é suficiente)
```

---

## Fase 4: Verificação

Cada correção exige verificação antes de ser declarada resolvida:

1. **XSS:** Tentar injetar `<script>alert(1)</script>` no campo corrigido — deve aparecer como texto, não executar
2. **Headers:** Re-executar o `curl -I` e confirmar presença
3. **Credenciais:** Confirmar que `git log -S "valor-da-chave" --all` não retorna nada
4. **Dependências:** Re-executar `npm audit` e confirmar zero vulnerabilidades críticas/altas
5. **CORS/CSRF:** Testar com origem diferente via fetch no console do browser

**Frase proibida:** "A correção deve funcionar." Só declare corrigido após confirmar com comando ou teste.

---

## Registro de Auditoria

Ao final, registre:
```
Data: YYYY-MM-DD
Arquivos auditados: [lista]
Achados: N críticos, N altos, N médios, N baixos
Corrigidos: [lista com evidência]
Pendentes: [lista com justificativa]
Próxima auditoria recomendada: [trigger ou data]
```

---

## Red Flags — Nunca Ignore

- `eval()` com qualquer dado externo
- Credencial hardcoded em qualquer arquivo rastreado por git
- `Access-Control-Allow-Origin: *` em APIs autenticadas
- Upload sem validação server-side
- Query SQL com concatenação de string de usuário
- Token de sessão passado via URL (aparece em logs)
