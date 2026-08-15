---
name: schematize-csharp
metadata:
  version: 0.1.0
description: Padrões normativos de engenharia da casa no recorte C# / .NET (ASP.NET Core, EF Core, .NET LTS 8/9) — arquitetura/DDD, segurança, IAM, testes/pentest, dados, observabilidade, deploy, archive. Use SEMPRE que for projetar, gerar, revisar ou refatorar backend, API, serviço, worker, schema, migration, infra, CI/CD, teste ou deploy em C#/.NET — mesmo que o usuário não cite "padrão". Aplique também ao decidir arquitetura, escolher stack (C# entra por fit + ADR no rol sancionado Go/Rust/Elixir/C#/Zig/Ruby), modelar eventos/banco, desenhar auth/identidade, escrever testes/pentest, configurar observabilidade, ou produzir ADR/runbook/archive. Contém pisos inegociáveis (segredo nunca no cliente, sem SQL por concatenação/interpolação — EF/Dapper parametrizado, nullable reference types habilitado, auth server-side, JWT validado por inteiro, IAM como app separada, archive obrigatório) que vetam atalhos inseguros. Frontend (incl. Blazor/UI) delega ao schematize-web; segurança ofensiva na schematize-pentest.
---

# Padrões de Engenharia da Casa — C# / .NET

Conjunto normativo que rege como software em **C# (.NET)** é projetado, construído, testado e operado aqui. Esta skill é o **recorte C#** dos Padrões de Engenharia da Casa: mantém **todos os pisos agnósticos** (segurança, IAM, testes de verdade, ops/DoD/archive) e os concretiza no ecossistema .NET — **ASP.NET Core**, **EF Core**, **.NET LTS atual (8/9)**. Toda entrega — humana ou assistida por IA — segue os mesmos padrões de clareza, segurança e observabilidade, sem reinventar nem cortar caminho.

**Versão:** skill `schematize-csharp` v0.1.0. Changelog em `CHANGELOG.md`.

## Comandos (Claude Code)

Digite `/cs-help` pra ver todos. Em resumo:

| Comando | O que faz |
|---|---|
| `/cs-help` | lista todos os comandos do schematize-csharp |
| `/cs-load` | carrega à força TODO o corpo normativo no contexto e passa a aplicá-lo como regra inegociável |
| `/cs-claude` | cria ou **atualiza (sobrescreve)** o `CLAUDE.md` da raiz com a versão atual da skill (backup se houver customização) |
| `/cs-cc` | context compact: gera context.md + checklist.md no archive e roda `/compact` |
| `/cs-handoff` | gera o handoff (context.md + checklist.md) **sem** compactar — pra fim de sessão |
| `/cs-qa` | fluxo de Q.A. plan-first (§22.9): planeja, gera MD, pede aprovação, roda |
| `/cs-review` | roda o gate da DoD/§37 no diff (arquivo >750 bloqueia / >300 úteis flag, membro público sem `///`, nullable/analyzer, índice, macaquices) |
| `/cs-iam` | força/audita/scaffolda o IAM (identidade≠email, ≥2 fatores, ReBAC multi-tenant, sessão longa/logout irreversível) como microserviço C#/.NET separado em `auth.<domain>`, ou porta um auth legado |
| `/cs-index` | (re)gera o índice de microfunções (§39) a partir dos doc-comments XML `///` |
| `/cs-ops` | audita/scaffolda o `<projeto>_ops` (interface única): fluxo de ambientes, instalação paralela (`nproc`), independência |

Os comandos ficam em `assets/commands/` e são instalados em `.claude/commands/`.

## Como usar esta skill

1. Identifique o domínio da tarefa e **leia o(s) reference(s) relevante(s)** antes de produzir código ou decisão. Não trabalhe de memória — os detalhes (versões, limites, convenções) estão nos arquivos.
2. **Sempre** aplique os pisos inegociáveis abaixo, independente do reference carregado.
3. Ao terminar, valide contra a Definition of Done (`references/entrega.md`, §35) e **gere o archive** (§28, `references/operacao.md`).

Mapa de references — leia o que casa com a tarefa:

| Tarefa | Reference |
|---|---|
| **Limites de código (arquivo ≤750: ~500 úteis + ~250 comentário; flag >300 úteis), uma unidade/arquivo, `///` XML doc, MAPA** | `references/padroes-codigo.md` |
| Arquitetura, camadas (projeto por camada/DDD), repositórios, anti-monólito, **rol de linguagens + fit de C#**, CQRS | `references/arquitetura.md` |
| **Stack/versões: .NET LTS 8/9, `dotnet` CLI, ASP.NET Core, EF Core, nullable/analyzers, Central Package Management + lockfile, Polly** | `references/stack-versoes.md` |
| **Async/concorrência: async/await de ponta a ponta, `CancellationToken`, `Task`/`ValueTask`, `Channel<T>`, backpressure, sem sync-over-async/`async void`, shutdown gracioso** | `references/async-concorrencia.md` |
| Eventos/mensageria, banco, cache, APIs, resiliência, jobs, migrations reversíveis (`dotnet ef`) | `references/dados-eventos.md` |
| Segurança, auth, JWT (`JwtBearer` completo), multi-tenancy, LGPD, EF parametrizado, secrets, Data Protection, **frontend/segredos** | `references/seguranca.md` |
| **IAM (identidade+autorização): auth como microserviço C#/.NET separado (`auth.<domain>`), ID≠email, ≥2 fatores/passkey/Resend/Twilio, ReBAC multi-tenant, sessão longa/logout irreversível, migração de legado** | `references/iam.md` |
| **Cadeia de suprimentos: `packages.lock.json`/CPM, SBOM, scan que trava, imagem mínima/pinada/assinada, SLSA, segredo no build** | `references/cadeia-suprimentos.md` |
| Testes — test kit, saída machine-readable, categorias de teste (§22.1–22.3), xUnit/coverlet/FsCheck | `references/testes.md` |
| Testes — padrão de script, seeds, CI, pentest, Q.A. plan-first, Makefile (§22.4–23) | `references/testes-execucao.md` |
| Observabilidade (OpenTelemetry .NET + `ILogger`), healthchecks, performance, FinOps | `references/observabilidade.md` |
| Config, deploy/K8s, git/PR, ownership, runbooks/incidentes, ADR, **archive** (§20–28) | `references/operacao.md` |
| **Ops (control plane): fluxo dev→local→github→hml→prd (nada direto no servidor), ops como interface única (100%, autônomo), instalação paralela=`nproc`, independência=invariante** | `references/ops.md` |
| Templates, feature flags, IA assistida, DoD, evolução, índice de funcionalidades (§29+) | `references/entrega.md` |
| Filosofia, aplicação universal e a lista completa de anti-padrões vetados | `references/anti-padroes.md` |
| Gestão de contexto em sessões longas no Claude Code (handoff, hooks) | `references/contexto-claude-code.md` |

## Pisos inegociáveis (VETADO — sem ADR de exceção)

Estes nunca são violados, nem "pra funcionar", nem "pra ir mais rápido". A lista completa com veto + caminho certo está em `references/anti-padroes.md` (§37). Os que mais aparecem em código gerado às pressas:

- **Segredo nunca no cliente.** Nada de API key, secret de JWT, senha de banco, connection string com credencial ou token em bundle do browser, nem em `NEXT_PUBLIC_*`/`VITE_*`, nem em `appsettings.json` commitado. Segredo só server-side (user-secrets/Key Vault/env/secret manager). Detalhe em `references/seguranca.md`.
- **SQL sempre parametrizado.** EF Core/Dapper com parâmetros; **VETADO** `FromSqlRaw`/`ExecuteSqlRaw`/`CommandText` com string interpolada ou concatenada. Concatenar input em query é injeção esperando acontecer.
- **Nullable reference types habilitado** (`<Nullable>enable</Nullable>`) — piso de linguagem. Sem `#nullable disable` nem `!` (null-forgiving) indiscriminado pra calar o compilador.
- **Auth e autorização server-side.** `tenant_id`/role/`user_id` vêm do token verificado (claims), nunca do body/header do cliente. Validação no front é UX, não controle.
- **JWT validado por inteiro** via `JwtBearer` + `TokenValidationParameters` completos (assinatura, `exp`, `nbf`, `aud`, `iss`, `alg` em allowlist — RS256/EdDSA, nunca HS256 em fluxo público). Senha em **argon2id** (ou PBKDF2 forte). Token/id de sessão por **`RandomNumberGenerator`**, nunca `System.Random`/`Guid`.
- **Erro nunca engolido** (`catch {}`/`catch(Exception){}` vazio); sem `async void` que perde exceção; sem `.Result`/`.Wait()` (sync-over-async) — ver `references/async-concorrencia.md`.
- **Teste nunca silenciado** pra passar CI (`[Fact(Skip=…)]`, comentar assert, baixar threshold de cobertura). Conserta o código, não o teste.
- **Sem monólito que mistura bounded contexts**, sem monólito distribuído, sem shared lib `commons` de domínio. Detalhe em `references/arquitetura.md`.
- **Archive SEMPRE gerado.** Toda entrega que produz código/decisão/mudança de estado gera o `.md` de archive (§28) — é parte da entrega, não extra. Pular é violação direta. Templates em `assets/`.
- **Migration reversível** (com `Down`, testada com rollback — `dotnet ef`). Container não-root, read-only. Dependência NuGet nova com nome/autor/licença/versão verificados (typosquatting é real).
- **Pisos de código (`references/padroes-codigo.md`):** arquivos **≤ 750 linhas** (~250 comentário + ~500 útil; acima → quebrar por coesão), **flag em > 300 linhas úteis**, **uma unidade lógica por arquivo**, **todo membro público com doc XML `///`** (motivo/comportamento/entradas/saídas/efeitos), **`MAPA.md`** atualizado no mesmo PR — em **`<projeto>_archive/index/`, nunca no root** — e **índice de microfunções** regenerado (`/cs-index`). **Todo MD gerado mora no archive**, root limpo (§28).
- **Backend novo no rol sancionado, com ADR de fit.** O rol é **Go / Rust / Elixir / C# / Zig / Ruby** — escolhido por **encaixe com o problema** (§3), não por gosto. C# entra pelo fit .NET/enterprise (ASP.NET Core/EF Core, integração Microsoft, times .NET, desktop/Windows). **Node-backend e PHP são legado/saída** (não recebem serviço novo; migram por funcionalidade do módulo). **Frontend Node é 100% permitido** (Next.js/Astro; Blazor/UI no `schematize-web`). O anti-padrão é **abrir backend fora do rol / sem ADR**, não "usou C# em vez de Go". Detalhe em `references/arquitetura.md` (§3) e `schematize-engineering/references/linguagens.md`.
- **Fluxo de ambientes e ops (`references/ops.md`).** Toda mudança segue **dev local → teste local → GitHub → hml → prd**; **VETADO editar código direto no servidor**. **100%** das operações no servidor passam pela **ferramenta do `<projeto>_ops`** — nunca à mão; o ops é **autônomo**. **Instalação sempre paralela** = `nproc`; **falha no paralelo = serviços não independentes → corrigir a independência é prioridade máxima**.
- **Deploy destrutivo por seed + isolamento por usuário (`references/ops.md` §2–§3).** O ops provisiona em **`/<app>/`** clonando os repos dentro; **`/<app>/.env` é o seeder global**. **Todo redeploy é destrutivo na aplicação** (apaga e recria clone zerado do seed), **preservando os dados** (migration reversível). **Cada serviço roda como user Linux próprio em systemd unit hardened.**
- **IAM por desenho (`references/iam.md`).** Todo projeto começa com identidade+autorização robustas, e o **auth é app SEPARADA** — **microserviço C#/.NET** `<projeto>_auth_cs` + front próprio em `auth.<domain>`, isolados; nunca monolith. **ASP.NET Core Identity embutido no app NÃO é o IAM da casa.** Apps delegam por OIDC/PKCE e validam por JWKS público. **ID interno imutável (ULID/UUIDv7) — email/telefone nunca é ID**. **≥2 fatores sempre** (passkey/WebAuthn núcleo, TOTP/push, email OTP Resend always-on, Twilio; providers como interfaces C# via DI; senha argon2id+HIBP por padrão mas opcional); **recuperação ≥ login**. **Multi-tenant RBAC/ABAC granular via ReBAC** (OpenFGA/SpiceDB; deny-default, PDP=Check API / PEP=middleware/policy ASP.NET Core, server-side, token fino). **Multi-dispositivo, sessão 7d/90d, logout irreversível.** **Migrar auth legado = prioridade 0.** Scaffold/auditoria por **`/cs-iam`**; testes cross-tenant/priv-esc na `schematize-pentest`.

> Regra de bolso: se a justificativa começa com "só pra funcionar", "depois eu arrumo" ou "é mais rápido assim" e o resultado mexe em segredo, auth, dado ou registro — é um anti-padrão vetado. Pare e faça certo.

## Testes — o que conta como "verde de verdade"

Detalhe completo em `references/testes.md` (§22.1–22.3) e `references/testes-execucao.md` (§22.4–23). Ferramental da casa: **xUnit** (primário) / NUnit, **FluentAssertions**, **coverlet** (cobertura), **FsCheck** (property-based), **Stryker.NET** (mutation), **Testcontainers** (integração), **Moq/NSubstitute** nas bordas. O essencial:

- **Smoke não pode ser teatro:** assertar shape do body (não só status 200), assertion negativa (sem stack trace/placeholder), e um **self-check que força uma falha conhecida** pra provar que o runner consegue reportar FAIL. Smoke que nunca falha está cego.
- **Unit agressivo:** caminho de erro obrigatório, casos hostis (tipo errado, unicode, null byte, boundary), property-based (FsCheck) e mutation testing (Stryker.NET) no domínio crítico. Cobertura de linha é piso, não meta.
- **Pentest prova rejeição, rota por rota, campo por campo:** nunca 500 por input hostil, nunca coerção silenciosa de tipo, nunca eco sem escape, nunca vazamento cross-tenant. Princípios em `references/testes-execucao.md` (§22.8).
- **`simulated` (teste emulado):** cruza rotas × personas × injections e prova que **100% das rotas** do inventário estão acessíveis pra quem deve e bloqueadas pra quem não deve. Rota fantasma/morta quebra o run.
- **Fluxo de Q.A. (plan-first, §22.9):** toda submissão de Q.A. **planeja tudo primeiro**, gera um MD detalhado de passo a passo, e **pede aprovação do usuário antes de executar**. Nada de Q.A. roda às cegas.

## Andaime pronto (scripts e templates)

Não escreva do zero o que já está bundlado:

- `scripts/lib.sh` — helpers de teste (`test_pass`, `test_fail`, `test_skip`, `test_section`, `test_summary`, `http_call`, `assert_http_in`). Todo script de teste usa estes.
- `scripts/test-skeleton.sh` — esqueleto obrigatório de `tests/<mode>/<name>.sh`.
- `scripts/smoke-selfcheck.sh` — o meta-teste anti "verde mentiroso".
- `scripts/simulated/run.py` — scaffold do engine rotas × personas × injections.
- `scripts/check-diff.sh` — gate determinístico da DoD/§37 (tamanho de arquivo, macaquices C#: SQL interpolado, `.Result`/`.Wait()`, `async void`, `#nullable disable`, `System.Random` em segredo).
- `scripts/build-index.mjs` — gera o índice de microfunções dos doc-comments (reconhece `///` XML e assinaturas C#).
- `scripts/hooks/context-monitor.mjs` + `scripts/hooks/precompact-backup.mjs` — gestão de contexto no Claude Code. Ver `references/contexto-claude-code.md` e `assets/settings.claude.example.json`.
- `assets/lint/Directory.Build.props` + `assets/lint/.editorconfig` — pisos de linguagem prontos (nullable, warnings-as-errors, analyzers Roslyn, doc XML, lockfile). `assets/ci/github-actions-ci.yml` — CI .NET de referência. `assets/hooks/.pre-commit-config.yaml` — pre-commit (`dotnet format`/build/gitleaks).
- `assets/ADR.md`, `assets/TASK.md`, `assets/CHAT_ARCHIVE.md`, `assets/PR_TEMPLATE.md`, `assets/RUNBOOK.md` — templates. ADR/TASK/CHAT_ARCHIVE cumprem §27/§28.
- `assets/INDEX_GLOBAL.md` + `assets/INDEX_FUNCTIONS.md` + `assets/MAPA.md` — índice de funcionalidades (§39).
- `assets/CLAUDE.md` — arquivo "sempre on" pra colocar na **raiz do repositório**. Copie e ajuste `<project>`.
- `assets/commands/cs-cc.md` — comando `/cs-cc` (context compact). Copie para `.claude/commands/cs-cc.md`.

## Aplicação sempre-on

Esta skill é puxada quando a tarefa casa com a descrição. Para garantir que os padrões valham em **toda** interação do repo (e não só nas que disparam a skill), copie `assets/CLAUDE.md` para a raiz do projeto. Os dois mecanismos se complementam: o `CLAUDE.md` pina o resumo e aponta pra cá; a skill entrega o detalhe e o andaime.
