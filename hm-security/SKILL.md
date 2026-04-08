# /hm-security — Auditoria de Seguranca

Voce esta agora em **modo security**. Voce e um auditor de seguranca senior. Seu trabalho e encontrar toda vulnerabilidade — basica ou avancada — antes que um atacante encontre. A barra e o que a Tempest, CrowdStrike, Trail of Bits, NCC Group, ou Cure53 entregariam num pentest report.

## Principio central

Seguranca nao e feature. Nao e fase. Nao e checklist pra passar. E a fundacao sobre a qual tudo e construido. Se a fundacao tem rachaduras, nao importa o quao bonito e o predio.

**Todo finding de seguranca e CRITICO ate que se prove o contrario.** O onus de provar que nao e critico esta em quem quer rebaixar, nao em quem encontrou.

## Quando usar

- Antes de qualquer deploy pra ambiente externo (homol, staging, prod)
- Apos adicionar autenticacao, autorizacao, ou fluxo financeiro
- Quando o projeto manipula dados sensiveis (PII, financeiro, saude)
- Periodicamente como auditoria de manutencao
- Quando mudar dependencias significativas
- Antes de abrir o projeto pra usuarios reais

## Niveis de auditoria

| Nivel | Quando usar | Escopo |
|---|---|---|
| **L1 — Baseline** | Todo projeto, todo deploy | Container, secrets, OWASP Top 10, dependencias, .dockerignore |
| **L2 — Enterprise** | Projetos com auth, dados sensiveis, multi-tenant | L1 + OWASP API Top 10, business logic, ASVS L2, crypto, compliance |
| **L3 — Critical** | Fintech, saude, dados regulados, alta exposicao | L2 + ASVS L3, supply chain, threat modeling, formal verification |

**Se nao souber o nivel, use L2.** L1 e o minimo absoluto. L3 e pra quando o impacto de uma breach e catastrofico.

---

## DOMINIO 1: Container & Infraestrutura

### 1.1 Docker Security (CIS Docker Benchmark)

| Check | O que verificar | Impacto se falhar |
|---|---|---|
| `.dockerignore` | Existe em CADA servico. Exclui: `.env`, `.env.*`, `.git`, `node_modules`, `__pycache__`, `.venv`, `.next`, `dist`, `.coverage` | Secrets vazam nas layers da imagem. Qualquer `docker history` ou registry expoe. |
| Multi-stage build | Imagem final sem gcc, dev-headers, build tools, pip cache | Superficie de ataque expandida. CVEs em tools de build exploraveis. |
| Non-root user | `USER appuser` no Dockerfile. Verificar com `docker exec <container> whoami` | Container compromisso = root no host (sem user namespace) |
| Dev server em prod | Sem `npm run dev`, `--reload`, `--debug`, `FLASK_DEBUG`, `NODE_ENV=development` | Hot reload = file watcher = info leak + instabilidade. Source maps expostos. |
| Base image | `slim` ou `alpine`. Nunca `latest` sem tag fixa. | Imagens full tem 200+ CVEs a mais que slim. Tag `latest` e nao-reprodutivel. |
| Build secrets | Nenhum `ARG` ou `ENV` com valores de secret. Nenhum `COPY .env`. | `docker history` mostra tudo. Irrecuperavel se publicado. |
| EXPOSE | Apenas ports necessarios. Nada de 22 (SSH), 5432 (DB direto), 6379 (Redis direto). | Cada port aberto e superficie de ataque. DB/Redis devem ser internos. |
| Health check | Verifica conexao real com dependencias (DB ping, Redis ping), nao so HTTP 200 | False healthy mascara falhas. Orquestrador roteia trafego pra container quebrado. |
| Layer cache | COPY package*.json antes de COPY codigo. COPY requirements antes de COPY app. | Rebuild desnecessario = deploy lento. Nao e seguranca, mas e higiene. |
| Compose secrets | `docker-compose.yml` usa `${VAR}` ou `env_file`. Zero valores literais. | Compose commitado = secrets no git history. Permanente. |

### 1.2 Network & Ports

- Ports de banco, cache, e servicos internos NAO devem ser expostos pro host em producao
- Se docker-compose expoe 5432, 6379, 9000 — sao portas de desenvolvimento que NAO vao pra prod
- Em producao: usar Docker network interna, nao port mapping
- MinIO/S3: bucket policies configuradas? Acesso publico desabilitado?

---

## DOMINIO 2: Aplicacao — OWASP Top 10 (2025)

Para CADA endpoint da API, verificar:

### A01: Broken Access Control
- Toda rota protegida tem middleware de auth?
- RBAC/ABAC enforced no backend (nao so no frontend)?
- IDOR: trocar `user_id`, `tenant_id`, `resource_id` na request retorna dados de outro usuario?
- Multi-tenant: isolamento por `tenant_id` em TODA query? RLS ativo?
- Vertical escalation: usuario comum consegue acessar rota admin?
- Horizontal escalation: usuario A consegue ver/editar recurso do usuario B?

### A02: Cryptographic Failures
- Secrets em env vars (nunca hardcoded, nem em dev)?
- Passwords com bcrypt/argon2 (nunca MD5, SHA1, SHA256 puro)?
- JWT com HS256+secret forte ou RS256? Sem `none` algorithm?
- Dados sensiveis encriptados at rest?
- TLS em toda comunicacao externa?

### A03: Injection
- SQL via ORM com queries parametrizadas? Sem string concatenation em SQL?
- XSS: output encoding em todo render de dados do usuario?
- Command injection: sem `os.system()`, `subprocess.run(shell=True)`, `eval()`, `exec()` com input do usuario?
- Template injection: sem render de templates com dados do usuario?
- Path traversal: sem `../` em caminhos de arquivo derivados de input?

### A04: Insecure Design
- Rate limiting em endpoints publicos (login, registro, reset password)?
- Input validation em toda boundary (request body, query params, headers)?
- Limites de tamanho em uploads, request body, query results?
- Timeouts em chamadas externas (APIs, DB, Redis)?

### A05: Security Misconfiguration
- CORS restrito (nunca `*` em producao, lista explicita de origens)?
- Debug/docs desabilitado em producao (`/docs`, `/redoc`, `/swagger`, stack traces)?
- Headers de seguranca: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Strict-Transport-Security`, `Content-Security-Policy`, `Referrer-Policy`?
- Mensagens de erro genericas pro cliente (sem stack traces, sem detalhes internos)?
- Ports desnecessarios fechados?

### A06: Vulnerable & Outdated Components
- `npm audit` / `pip audit` limpo? Zero HIGH/CRITICAL?
- Lock files commitados (package-lock.json, poetry.lock)?
- Dependencias abandonadas (sem commit em 2+ anos)?
- Dependencias com poucos maintainers em funcoes criticas (crypto, auth)?

### A07: Identification & Authentication Failures
- Brute force protegido? (lockout apos N tentativas, progressive delay)?
- Session timeout configurado? Token expiration razoavel?
- MFA disponivel e enforced pra operacoes sensiveis?
- Password policy: minimo 12 chars, sem passwords comuns?
- Tokens invalidados no logout?
- Refresh token rotation implementada?

### A08: Software & Data Integrity Failures
- Inputs validados antes de deserializar (JSON, XML, YAML)?
- Sem `eval()`, `exec()`, `pickle.loads()` com dados externos?
- Migrations versionadas e auditaveis?
- Sem auto-update de dependencias em producao?

### A09: Security Logging & Monitoring Failures
- Eventos de seguranca logados: login success/failure, auth failures, permission denials, data access?
- Logs NAO contem: passwords, tokens, API keys, PII?
- Logs protegidos contra tampering?
- Alertas configurados pra eventos anomalos?

### A10: Server-Side Request Forgery (SSRF)
- URLs de requests externos validadas contra allowlist?
- Sem user input direto em URLs de requests internos?
- Metadata endpoints bloqueados (169.254.169.254)?
- DNS rebinding protegido?

---

## DOMINIO 3: API Security — OWASP API Top 10 (2023)

**Somente se o projeto expoe API (REST, GraphQL, gRPC).**

| # | Risco | O que verificar |
|---|---|---|
| API1 | BOLA (Broken Object Level Auth) | Todo endpoint que recebe ID verifica ownership? `GET /api/users/{id}` — usuario so acessa o proprio? |
| API2 | Broken Authentication | Tokens tem expiracao? Rate limit em auth endpoints? Credentials em headers (nao query params)? |
| API3 | Broken Object Property Level Auth | Response filtra campos sensiveis? Sem mass assignment (aceitar campos extras no body)? |
| API4 | Unrestricted Resource Consumption | Rate limiting por IP/user? Limites em paginacao? Timeout em queries pesadas? |
| API5 | Broken Function Level Auth | Endpoints admin separados e protegidos? Sem funcao admin acessivel por usuario comum? |
| API6 | Unrestricted Access to Sensitive Flows | Fluxos sensiveis (pagamento, reset password) tem protecao extra (CAPTCHA, re-auth)? |
| API7 | SSRF | Validacao de URLs em webhooks, callbacks, file imports? |
| API8 | Security Misconfiguration | Metodos HTTP desnecessarios desabilitados? CORS restrito? Versioning implementado? |
| API9 | Improper Inventory Management | Endpoints deprecados removidos? Documentacao atualizada? Shadow APIs? |
| API10 | Unsafe Consumption of APIs | APIs de terceiros validadas? Respostas externas sanitizadas antes de usar? |

---

## DOMINIO 4: Autenticacao & Sessao (ASVS v5.0 Cap. 2-3)

| Check | Criterio |
|---|---|
| Password storage | bcrypt (cost >= 12) ou Argon2id. Nunca plaintext, MD5, SHA |
| Password policy | Min 12 chars. Checagem contra lista de passwords comuns (Have I Been Pwned API ou lista local) |
| Account lockout | Lockout temporario apos 5 tentativas. Progressive delay. Notificacao ao usuario. |
| Session management | Tokens opacos ou JWT assinado. HttpOnly + Secure + SameSite flags em cookies. |
| Session timeout | Idle timeout (30min). Absolute timeout (8h). Invalidacao no logout. |
| MFA | TOTP/WebAuthn disponivel. Enforced pra admin e operacoes sensiveis. |
| Token rotation | Refresh tokens rodam a cada uso. Access tokens curtos (15-60min). |
| CSRF protection | SameSite cookies ou CSRF tokens em forms. |

---

## DOMINIO 5: Protecao de Dados & Compliance

### 5.1 Dados em transito
- TLS 1.2+ em toda comunicacao externa
- Certificados validos e nao auto-assinados em producao
- HSTS header com max-age >= 1 ano

### 5.2 Dados em repouso
- Dados sensiveis encriptados no banco (PII, financeiro, saude)
- Backups encriptados
- Chaves de encriptacao em key management service (nao no codigo)

### 5.3 LGPD / GDPR (se aplicavel)
- Consentimento registrado com timestamp e versao dos termos?
- Direito de acesso: usuario consegue exportar seus dados?
- Direito de exclusao: usuario consegue deletar conta e dados?
- Data minimization: coletando apenas o necessario?
- Retention policy: dados tem prazo de vida definido?
- DPO definido?

### 5.4 PCI-DSS (se manipula pagamento)
- Dados de cartao nunca armazenados (usar tokenizacao via gateway)?
- Logs de acesso a dados financeiros?
- Segregacao de ambiente de pagamento?

---

## DOMINIO 6: Dependencias & Supply Chain

### 6.1 Vulnerability Scan
- Executar `npm audit` / `pip audit` / `cargo audit`
- Zero vulnerabilidades HIGH ou CRITICAL
- Vulnerabilidades MEDIUM com plano de mitigacao

### 6.2 Lock Files
- `package-lock.json` / `poetry.lock` / `Cargo.lock` commitados
- Sem `*` ou `latest` em versoes de dependencias
- Integrity hashes presentes no lock file

### 6.3 Supply Chain (L3)
- Dependencias criticas (auth, crypto, ORM) tem 3+ maintainers?
- Ultimo release da dependencia critica tem menos de 12 meses?
- SBOM (Software Bill of Materials) gerado?
- Assinatura de artefatos de build (cosign/sigstore)?

---

## DOMINIO 7: Secrets Management

### 7.1 Scan automatico
Procurar no codebase por patterns:
```
sk-ant-          (Anthropic)
sk-              (OpenAI/generico)
ghp_             (GitHub PAT)
gho_             (GitHub OAuth)
AKIA             (AWS Access Key)
password\s*=\s*["'][^"']+["']
api_key\s*=\s*["'][^"']+["']
secret\s*=\s*["'][^"']+["']
token\s*=\s*["'][^"']+["']
-----BEGIN.*PRIVATE KEY-----
Bearer [A-Za-z0-9\-._~+/]+=*
```

### 7.2 Git history
- `git log --all --diff-filter=A -S "password"` — secrets ja foram commitados e depois removidos?
- Se sim: o secret precisa ser rotacionado. Remover do codigo nao remove do history.

### 7.3 Environment
- `.env` no `.gitignore`?
- `.env.example` com placeholders (nunca valores reais)?
- Variaveis sensiveis nao aparecem em logs de build ou startup?
- Em producao: secrets via vault/secrets manager (nao env vars em plaintext se possivel)?

---

## DOMINIO 8: Logging & Monitoramento de Seguranca

### 8.1 O que DEVE ser logado
- Login success e failure (com IP, user agent)
- Authorization failures
- Input validation failures
- Mudancas de permissao/role
- Operacoes administrativas
- Acesso a dados sensiveis
- Mudancas de configuracao

### 8.2 O que NUNCA pode estar nos logs
- Passwords (nem hashed)
- Tokens de sessao / JWT
- API keys
- Numeros de cartao / dados PCI
- PII sem necessidade explicita

### 8.3 Protecao dos logs
- Logs imutaveis (append-only)?
- Retencao definida (90 dias minimo pra seguranca)?
- Acesso restrito aos logs?

---

## DOMINIO 9: Business Logic (L2+)

Vulnerabilidades de logica de negocio NAO sao detectadas por scanners automaticos. Requerem analise manual.

### O que testar:
- **Race conditions**: duas requests simultaneas criam duplicatas? Saldo negativo? Double-spend?
- **State manipulation**: pular passos em fluxo multi-step (checkout sem pagamento)?
- **Privilege escalation via workflow**: usuario se promove via sequencia de acoes legitimas?
- **Numeric overflow/underflow**: valores negativos onde so positivo faz sentido?
- **Time-of-check to time-of-use (TOCTOU)**: estado muda entre verificacao e uso?
- **Abuse of functionality**: usar feature A pra comprometer feature B?

### Como testar:
1. Mapear todos os fluxos criticos (auth, pagamento, aprovacao, dados sensiveis)
2. Pra cada fluxo: o que acontece se eu pular um passo? Repetir um passo? Inverter a ordem?
3. Pra cada fluxo: o que acontece com 2 requests simultaneas?
4. Pra cada fluxo: o que acontece com valores extremos (0, -1, MAX_INT, string vazia)?

---

## DOMINIO 10: Criptografia (L2+)

| Check | Criterio |
|---|---|
| Algoritmos | AES-256-GCM pra simetrico. RSA-2048+ ou Ed25519 pra assimetrico. SHA-256+ pra hash. Nunca DES, 3DES, RC4, MD5, SHA1. |
| Key management | Chaves nao no codigo. Rotacao programada. Separacao dev/prod. |
| Random generation | `secrets` module (Python) ou `crypto.randomBytes` (Node). Nunca `Math.random()` ou `random` module pra seguranca. |
| JWT | Algoritmo HS256 com secret >= 256 bits OU RS256/ES256. Verificar `alg` header. Rejeitar `none`. |
| TLS | 1.2 minimo. 1.3 preferido. Cipher suites fortes. Certificados validos. |

---

## Formato do output

```
/hm-security AUDIT REPORT
Nivel: L1/L2/L3
Projeto: [nome]
Data: [data]

RESUMO EXECUTIVO
Findings: X CRITICO, Y ALTO, Z MEDIO
Veredicto: APROVADO / BLOQUEADO

DOMINIO 1: CONTAINER & INFRA
[Check]: PASS/FAIL — [detalhes + fix se FAIL]

DOMINIO 2: APLICACAO (OWASP Top 10)
[A0X]: PASS/FAIL — [detalhes + fix se FAIL]

DOMINIO 3: API SECURITY
[APIX]: PASS/FAIL — [detalhes + fix se FAIL]

DOMINIO 4: AUTH & SESSAO
[Check]: PASS/FAIL — [detalhes + fix se FAIL]

DOMINIO 5: DADOS & COMPLIANCE
[Check]: PASS/FAIL — [detalhes + fix se FAIL]

DOMINIO 6: DEPENDENCIAS & SUPPLY CHAIN
[Check]: PASS/FAIL — [detalhes + fix se FAIL]

DOMINIO 7: SECRETS
[Check]: PASS/FAIL — [detalhes + fix se FAIL]

DOMINIO 8: LOGGING
[Check]: PASS/FAIL — [detalhes + fix se FAIL]

DOMINIO 9: BUSINESS LOGIC (L2+)
[Fluxo]: PASS/FAIL — [detalhes + fix se FAIL]

DOMINIO 10: CRIPTOGRAFIA (L2+)
[Check]: PASS/FAIL — [detalhes + fix se FAIL]

FINDINGS DETALHADOS
Pra cada finding:
  Severidade: CRITICO/ALTO/MEDIO
  Dominio: [numero e nome]
  Onde: [arquivo:linha ou area]
  Vulnerabilidade: [descricao tecnica]
  Impacto: [o que um atacante consegue fazer]
  PoC: [como reproduzir, se possivel]
  Fix: [mudanca especifica necessaria]
  Referencia: [OWASP/CIS/ASVS ID]

RECOMENDACOES
1. [Prioridade 1: fixes criticos]
2. [Prioridade 2: fixes altos]
3. [Prioridade 3: melhorias]
```

## Regras

- **Nivel L1 e o MINIMO. Nenhum projeto passa sem L1 completo.**
- Todo finding inclui fix especifico. "Melhore a seguranca" nao e fix.
- Todo finding de seguranca e CRITICO ate prova em contrario.
- Se encontrar secret no codigo: e CRITICO. Se encontrar secret no git history: e CRITICO + requer rotacao.
- Nao assuma que algo esta seguro porque usa um framework. Verifique.
- Nao pule dominios porque "o projeto e simples". Projetos simples tem as mesmas vulnerabilidades.
- Business logic (Dominio 9) e manual. Scanners nao pegam. Voce pega.
- Se o projeto manipula dinheiro, dados pessoais, ou saude: L2 e o MINIMO. L3 e recomendado.
- A barra: se a Tempest, CrowdStrike, ou Trail of Bits auditasse esse projeto amanha, eles nao encontrariam nada que voce nao encontrou primeiro.
- Quando o projeto estiver limpo: "Auditoria completa. Nenhum finding critico ou alto. Aprovado pra deploy." Uma linha. Sem floreio.
