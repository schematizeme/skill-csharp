---
name: schematize-csharp
metadata:
  version: 0.2.0
description: Padrões normativos de engenharia da casa no recorte C#/.NET (ASP.NET Core, EF Core, .NET LTS) — arquitetura/DDD, segurança, IAM, testes/pentest, dados, observabilidade, deploy, archive. Use SEMPRE que for projetar, gerar, revisar ou refatorar backend, API, serviço, worker, schema, migration, infra, CI/CD, teste ou deploy em C#/.NET — mesmo sem citar "padrão" —, e ao escolher stack (C# entra por fit + ADR no rol Go/Rust/Elixir/C#/Zig/Ruby), modelar eventos/banco, desenhar auth, configurar observabilidade ou produzir ADR/runbook/archive. Pisos: segredo nunca no cliente; sem SQL concatenado (EF/Dapper parametrizado); nullable reference types ligado; auth server-side com JWT validado por inteiro; IAM como app separada; efeito externo (e-mail/SMS/push) NUNCA sai de não-produção — sink por default, guard deny-by-default, cap por execução, domínio de teste em rota nula; archive obrigatório. Frontend (incl. Blazor) delega à schematize-web; segurança ofensiva à schematize-pentest.
---

# Padrões de Engenharia da Casa — C# / .NET

Conjunto normativo que rege como software em **C# (.NET)** é projetado, construído, testado e operado aqui. Esta skill é o **recorte C#** dos Padrões de Engenharia da Casa: mantém **todos os pisos agnósticos** (segurança, IAM, testes de verdade, ops/DoD/archive) e os concretiza no ecossistema .NET — **ASP.NET Core**, **EF Core** e o **.NET LTS corrente** — a versão exata vive só no anexo volátil (`references/stack-versoes.md`, com data de verificação), nunca aqui. Toda entrega — humana ou assistida por IA — segue os mesmos padrões de clareza, segurança e observabilidade, sem reinventar nem cortar caminho.

**Versão:** skill `schematize-csharp` v0.2.0. Changelog em `CHANGELOG.md`.

## Precedência e herança (leia antes de divergir)

Esta skill é o **recorte C#/.NET** da base. Duas regras governam a relação, e elas resolvem sozinhas
quase toda dúvida de "onde está escrito o quê":

1. **Onde esta skill divergir da base, a BASE MANDA.** `schematize-engineering` é a normativa; aqui
   mora a **especialização** — o mecanismo, a lib, a sintaxe, o gate da linguagem. Divergência de
   *piso* entre este arquivo e a base é **defeito desta skill**, não uma variante local aceitável.
   Achou uma? É item de correção, não licença. *(Foi assim que o `argon2id-only` da casa virou
   "argon2id ou PBKDF2" em uma skill só, e o rol de 6 linguagens virou "só Go e Rust" em três.)*
2. **O que não está repetido aqui é HERDADO, não dispensado.** A ausência de um piso neste repo
   nunca significa que ele não vale — significa que ele não muda de forma nesta linguagem. Em
   especial, valem integralmente, sem cópia local:
   - **§28 Archive** — `<projeto>/<projeto>_archive/` é **repositório git próprio, PRIVADO e
     obrigatório**, criticidade 0 (`schematize-archive`; ADR-0005 para a planta canônica).
   - **§39 Índice/MAPA** — enumeração exaustiva (uma entrada por unidade chamável, `M == N`) e o
     **grafo com arestas em ASCII (`A -> B`), NUNCA a seta unicode** — o parser do app lê ASCII.
   - **§35 Definition of Done** e a lista de anti-padrões **§37** (citada por **título**, nunca por
     número: a numeração dos itens diverge entre skills).
   - **IAM** (`schematize-engineering` → `references/iam.md`): identidade ≠ email, ≥2 fatores, ReBAC multi-tenant,
     **alcançabilidade do 2º fator** (o fator de recuperação tem de ser alcançável quando o
     principal cai — senão o 2FA vira bug de bootstrap que tranca o dono para fora), os parâmetros
     mínimos de argon2id, sessão longa e logout irreversível.
   - **Rol sancionado** — Go, Rust, Elixir, C#, Zig, Ruby, por **fit + ADR**
     (`schematize-engineering` → `references/linguagens.md`). Esta skill é **uma** delas, não a
     régua das outras.
   - **Efeito externo** nunca sai de não-produção (`schematize-engineering` →
     `references/efeitos-externos.md`; gate em `scripts/check-external-effects.sh`, distribuído
     aqui — ADR-0008).

## Comandos (Claude Code)

Digite `/cs-help` pra ver todos. Em resumo:

| Comando | O que faz |
|---|---|
| `/cs-help` | lista todos os comandos do schematize-csharp |
| `/cs-load` | carrega à força TODO o corpo normativo no contexto e passa a aplicá-lo como regra inegociável |
| `/cs-claude` | cria ou **atualiza (sobrescreve)** o `CLAUDE.md` da raiz com a versão atual da skill (backup se houver customização) |
| `/cs-cc` | context compact: gera context.md + checklist.md no archive e roda `/compact` |
| `/cs-handoff` | gera o handoff (context.md + checklist.md) **sem** compactar — pra fim de sessão |
| `/cs-qa` | Q.A. no recorte C#/.NET — wrapper da skill **schematize-qa** (`/qa-plan` → `/qa-run`) |
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
| **Stack/versões: .NET LTS corrente, `dotnet` CLI, ASP.NET Core, EF Core, nullable/analyzers, Central Package Management + lockfile, Polly** | `references/stack-versoes.md` |
| **Async/concorrência: async/await de ponta a ponta, `CancellationToken`, `Task`/`ValueTask`, `Channel<T>`, backpressure, sem sync-over-async/`async void`, shutdown gracioso** | `references/async-concorrencia.md` |
| Eventos/mensageria, banco, cache, APIs, resiliência, jobs, migrations reversíveis (`dotnet ef`) | `references/dados-eventos.md` |
| Segurança, auth, JWT (`JwtBearer` completo), multi-tenancy, LGPD, EF parametrizado, secrets, Data Protection, **frontend/segredos** | `references/seguranca.md` |
| **Efeito externo fora de prd (e-mail/SMS/push): sink por default, guard como decorator no DI, `IOptions` com `ValidateOnStart`, cap com `Interlocked`, domínio de teste em rota nula** | `references/iam.md` (§3.1) + `schematize-engineering/references/efeitos-externos.md` |
| **IAM (identidade+autorização): auth como microserviço C#/.NET separado (`auth.<domain>`), ID≠email, ≥2 fatores/passkey/Resend/Twilio, ReBAC multi-tenant, sessão longa/logout irreversível, migração de legado** | `references/iam.md` |
| **Cadeia de suprimentos: `packages.lock.json`/CPM, SBOM, scan que trava, imagem mínima/pinada/assinada, SLSA, segredo no build** | `references/cadeia-suprimentos.md` |
| Testes — o recorte C#/.NET (runner, sintaxe, armadilhas do dialeto). **A disciplina é da `schematize-qa`.** | `references/testes.md` |
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
- **Efeito externo NUNCA sai de não-produção (`references/iam.md` §3.1; normativa em `schematize-engineering/references/efeitos-externos.md`).** E-mail (o **Email OTP always-on** é o disparador nº 1), SMS/voz, push, webhook de terceiro e cobrança **não acontecem de verdade** fora de `prd`. **(a)** Endereço sintético só no **domínio de teste em ROTA NULA** (`test.<domain>` com null MX + SPF `-all` + DMARC `p=reject`, ou `.test`/`.invalid`/`.example`) — **VETADO** `@gmail.com`, domínio de terceiro, e-mail de pessoa real (inclusive o seu) e o domínio de produção em fixture/seed/persona/demo. **(b)** `IEmailProvider` com **`SinkEmailProvider` por default** fora de prd e o **guard como decorator no DI** (`services.Decorate<IEmailProvider, GuardedEmailProvider>()` ou factory na composição — **nunca** o chamador escolhendo): destinatário fora do domínio de teste ⇒ **`ExternalRecipientBlockedException`**, erro, nunca warning/no-op. **Ambiente é DECLARADO** em `IOptions<MailOptions>` com `ValidateOnStart` — **jamais** `IHostEnvironment.IsProduction()`, que retorna `true` quando `ASPNETCORE_ENVIRONMENT` não está definido (fail-OPEN); ausente ⇒ não-prd. **(c)** **Cap por execução** (`MAIL_MAX_PER_RUN`, default 50) com `Interlocked` em singleton + abort. **(d)** Chave **sandbox** fora de prd, egress SMTP bloqueado em dev/hml. Entregar de verdade fora de prd exige **as cinco**: ADR + allowlist ≤5 + cap + janela + subdomínio separado. **Motivo:** bounce/complaint em massa **queima IP e domínio**, derruba o transacional de prd (inclusive o **OTP de login**) e custa semanas de warm-up — com utilidade zero.

> Regra de bolso: se a justificativa começa com "só pra funcionar", "depois eu arrumo" ou "é mais rápido assim" e o resultado mexe em segredo, auth, dado ou registro — é um anti-padrão vetado. Pare e faça certo.

## Testes — o que conta como "verde de verdade"

Detalhe em `references/testes.md` (o recorte C#/.NET) — a **disciplina** de teste é da `schematize-qa`, e a segurança ofensiva da `schematize-pentest`.

- **Smoke não pode ser teatro:** assertar shape do body (não só status 200), assertion negativa (sem stack trace/placeholder), e um **self-check que força uma falha conhecida** pra provar que o runner consegue reportar FAIL. Smoke que nunca falha está cego.
- **Unit agressivo:** caminho de erro obrigatório, casos hostis (tipo errado, unicode, null byte, boundary), property-based (FsCheck) e mutation testing (Stryker.NET) no domínio crítico. Cobertura de linha é piso, não meta.
- **Pentest prova rejeição, rota por rota, campo por campo:** nunca 500 por input hostil, nunca coerção silenciosa de tipo, nunca eco sem escape, nunca vazamento cross-tenant. Princípios em a `schematize-pentest`).
- **`simulated` (teste emulado):** cruza rotas × personas × injections e prova que **100% das rotas** do inventário estão acessíveis pra quem deve e bloqueadas pra quem não deve. Rota fantasma/morta quebra o run.
- **Q.A. mora na skill dedicada `schematize-qa`** (`/qa-plan` → `/qa-run`): planeja tudo, gera o MD de passo a passo, **pede aprovação antes de rodar** e trava nos gates. `/cs-qa` é o wrapper no recorte C#/.NET (`dotnet test`/xUnit). Nada de Q.A. roda às cegas.

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
