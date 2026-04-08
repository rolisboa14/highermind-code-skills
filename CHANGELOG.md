# Changelog

Todas as mudanças notáveis neste projeto serão documentadas aqui.

Formato baseado em [Keep a Changelog](https://keepachangelog.com/pt-BR/1.1.0/).

---

## [2.1.0] — 2026-04-08

Security-first evolution. Motivado por falhas reais no Orion Finance (Dockerfile com `npm run dev`, sem `.dockerignore`, `--reload` no entrypoint) que passaram pela v2.

### Adicionado

#### Nova skill: `/hm-security` — Auditoria de seguranca dedicada
- Skill world-class baseada em OWASP ASVS 5.0, CIS Benchmarks, SLSA, e metodologias de Tempest/CrowdStrike/Trail of Bits
- 3 niveis de auditoria: L1 (baseline), L2 (enterprise), L3 (critical systems)
- 8 dominios: Container, Aplicacao (OWASP Top 10 + API Top 10), Auth, Dados, Dependencias, Infra, Logging, Crypto
- Secrets scan automatico com patterns
- Supply chain audit (lock files, SBOM, provenance)
- Business logic testing (race conditions, IDOR, privilege escalation)
- Compliance mapping (LGPD, GDPR, PCI-DSS)
- Severidades padronizadas: CRITICO bloqueia, sem excecao

#### Skills existentes — Security gate integrado
- `/hm-deploy`: Security Gate como secao 0 (bloqueante antes de tudo)
- `/hm-engineer`: OWASP Top 10 + Container Security como primeira auditoria
- `/hm-init`: Seguranca obrigatoria desde o commit zero (.dockerignore, multi-stage, non-root)
- `/hm-qa`: Security Audit como primeira etapa antes de testes

### Mudado

- Seguranca movida de "camada de auditoria" para "gate bloqueante" em todas as skills
- Output de todas as skills agora tem secao de seguranca no topo
- Tabela de ports atualizada com todos os projetos ativos

---

## [2.0.0] — 2026-04-03

Evolução completa das skills baseada em aprendizados reais de 5+ projetos construídos com o framework (higher-mind-os, Scout, HM Finance, Orion, higher-mind-community).

### Adicionado

#### Nova skill: `/hm-deploy`
- Validação completa de infraestrutura, containers e reprodutibilidade
- Checklist de Docker (subida, rebuild, dados sagrados)
- Validação de environment e secrets
- Checklist de database e migrations
- Health checks e monitoramento
- Teste de reprodutibilidade (clone limpo)
- Segurança de deploy (ports, CORS, HTTPS, secrets)
- Tabela de ports dos projetos Higher Mind pra evitar colisões

#### `/hm-init` — Framework de decisão de stack
- Tabela de critérios ponderados pra avaliação de stack (fit, performance, custo, maturidade, ecossistema, DX, hiring)
- Anti-patterns de escolha explícitos
- Seção de arquitetura agent-first como default (quando aplicável)
- Infraestrutura local com Docker Compose desde o dia 1
- Restrições de custo como parte do design (API calls, hosting, bandwidth)
- Documentação obrigatória via ARCHITECTURE.md
- Princípio "dados são sagrados" desde o primeiro docker-compose.yml

#### `/hm-engineer` — Padrão senior e novas camadas de auditoria
- Baseline de engenheiro senior inegociável (zero bare except, zero any types, zero fire-and-forget, zero secrets hardcoded, zero queries sem limit)
- Nova camada: **Custo x Performance** (API calls justificadas, contexto mínimo em LLMs, token usage consciente)
- Nova camada: **Dados sagrados** (nenhuma operação destrutiva sem confirmação, volumes nomeados, migrations não-destrutivas)
- Nova camada: **Infraestrutura** (Docker rebuild vs restart, migrations automáticas, health checks, ports)
- Expansão de Performance: I/O paralelo, memoização
- Expansão de Arquitetura: validação de agent loops (max iterations, token limits, timeout)
- Regra: dados em risco é sempre CRÍTICO

#### `/hm-designer` — Agent-first UI e pixel perfect
- Filosofia agent-first: UI = visibilidade + override, não input principal
- Referência A24 adicionada às referências estéticas
- Seção **Pixel perfect**: zero tolerância a desalinhamentos, quebras, cortes
- Padrões técnicos: full-width layout, shimmer (não spinner), dark-first, inline styles quando framework não coopera, transições 200-300ms
- Novos anti-patterns: formulários onde agente deveria executar, spinners genéricos, layout centralizado em telas grandes
- Regra: desalinhamento arquitetural (formulário vs agente) é finding

#### `/hm-qa` — Infraestrutura, agente e custo como teste
- Nova seção: **Verificação de infraestrutura** (containers, migrations, ports, volumes, rebuild, .env)
- Nova seção: **Verificação de agente** (tool loops, alucinação de tools, token usage, custo por interação)
- Nova seção: **Integridade de dados** (persistência, migrations não-destrutivas, backups, operações destrutivas)
- Nova seção: **Check de custo** (API calls por fluxo, contexto mínimo, calls redundantes, custo por usuário/mês)
- Output expandido com seções de Infraestrutura, Agente, Integridade de Dados e Custo

### Mudado

- Contagem de skills: 4 → 5 (adição de `/hm-deploy`)
- SKILL.md parent atualizado pra refletir 5 skills
- Todas as skills agora consideram agent-first como paradigma (quando aplicável)
- Performance expandida de "não ter N+1" pra incluir custo de APIs externas e token management

---

## [1.0.0] — 2026-03-12

Release inicial.

### Skills
- `/hm-init` — Início de projeto com melhores ferramentas e estrutura
- `/hm-engineer` — Validação de código em todas as camadas
- `/hm-designer` — Validação de interface contra o mais alto padrão
- `/hm-qa` — Quality assurance completo

### Infraestrutura
- Setup script com symlinks automáticos
- CLAUDE.md.template como ponto de partida
- Instalação global e por projeto
