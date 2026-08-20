# Changelog — schematize-csharp

## [0.2.0] — 2026-08-20
Piso "efeito externo NUNCA sai de não-produção" no recorte C#/.NET.
### Adicionado
- **`SKILL.md`**: novo piso inegociável — efeito externo (e-mail/SMS/push/webhook/cobrança) não acontece fora de `prd`; sink por default, guard como decorator no DI, ambiente declarado em `IOptions` com `ValidateOnStart`, cap por execução, domínio de teste em rota nula. Nova linha no mapa de references.
- **`references/iam.md` §3.1** — *Disparo do Email OTP fora de prd*: `MailOptions` (ambiente **declarado**, `TestDomains`, `Allowlist`, `MaxPerRun`) com `AddOptions().ValidateDataAnnotations().ValidateOnStart()`; `SinkEmailProvider` default fora de prd e `ResendEmailProvider` como typed client só em prd; **`GuardedEmailProvider`** decorando `IEmailProvider` (`services.Decorate<>` do Scrutor ou factory), com `ExternalRecipientBlockedException`/`MailCapExceededException` tipadas, `MailRunQuota` singleton com `Interlocked` e match de domínio fail-closed; seleção por ambiente na composição do `Program.cs`; suite xUnit que **espera a exceção** e prova que o inner não recebeu nada. Novo item no checklist de DoD do IAM.
- **Alerta de fail-OPEN do .NET**: `IHostEnvironment.IsProduction()` retorna `true` quando `ASPNETCORE_ENVIRONMENT`/`DOTNET_ENVIRONMENT` não está definido — por isso o ambiente do guard vem da config declarada, e ausência significa não-produção.
- **`references/testes-execucao.md` §22.5** — seeds/personas com e-mail só no domínio de teste, veto a caixa real com gate `grep` no CI, e o teste que vê o vermelho do guard.
- **`references/anti-padroes.md`** — nova seção *Efeitos externos* com o item **47** (mandar de verdade fora de prd) e o caminho certo.
- **`assets/CLAUDE.md`** — piso **18** (efeito externo fora de não-produção, recorte .NET/DI).
### Mudado
- **`description`** do frontmatter passa a listar o piso de efeito externo entre os inegociáveis.

## [0.1.2] — 2026-08-18
Correção da contradição do muro pré-login de IAM (alinha ao `iam.md` da schematize-engineering).
### Mudado
- **/cs-iam**: removido o "2º fator forte obrigatório antes do acesso pleno" e o "força 2º fator no 1º login" — o muro pré-login / deadlock de bootstrap VETADO pela norma. Agora senha+Email OTP = 2FA baseline; fator forte é nudge + step-up just-in-time.


Formato: [Keep a Changelog]; versionamento: SemVer.

## [0.1.1] — 2026-08-18
Q.A. repointado para a skill dedicada **schematize-qa**.
### Mudado
- **`/cs-qa` virou wrapper fino** da **schematize-qa** (`/qa-plan` → `/qa-run`) no recorte C#/.NET (`dotnet test`/xUnit). Referências ao antigo **§22.9** removidas de `SKILL.md`, `references/testes*.md`, `assets/CLAUDE.md` e `/cs-help`.

## [0.1.0] — 2026-08-15
Primeira release — recorte **C# / .NET** dos padrões normativos de engenharia da casa.

### Adicionado
- Corpo normativo completo especializado para C#/.NET em `references/` (16 arquivos):
  arquitetura/DDD (projeto por camada, rol de linguagens + fit de C#), `stack-versoes.md`
  (.NET LTS 8/9, `dotnet` CLI, ASP.NET Core, EF Core, nullable/analyzers, Central Package
  Management + lockfile, Polly), `async-concorrencia.md` (async/await de ponta a ponta,
  `CancellationToken`, `Task`/`ValueTask`, `Channel<T>`/backpressure, sem sync-over-async/
  `async void`, shutdown gracioso), segurança (EF parametrizado, `JwtBearer` completo,
  argon2id/PBKDF2, `RandomNumberGenerator`, Data Protection, secrets), IAM (auth como
  microserviço C#/.NET separado — `<projeto>_auth_cs`; ASP.NET Core Identity ≠ IAM da casa),
  testes (xUnit/coverlet/FsCheck/Stryker.NET/Testcontainers), dados/eventos, cadeia de
  suprimentos, observabilidade (OpenTelemetry .NET + `ILogger`), operação, ops, entrega,
  anti-padrões, padrões de código, contexto.
- Comandos `/cs-help`, `/cs-load`, `/cs-claude`, `/cs-cc`, `/cs-handoff`, `/cs-qa`,
  `/cs-review`, `/cs-iam`, `/cs-index`, `/cs-ops` (prefixo `cs-`, sem conflito com as skills irmãs).
- Andaime: `scripts/` (`lib.sh`, `test-skeleton.sh`, `smoke-selfcheck.sh`, `simulated/run.py`,
  `check-diff.sh` com macaquices C#, `build-index.mjs` reconhecendo `///` XML e assinaturas C#,
  `archive-secret-scan.sh`, hooks de contexto).
- Assets: `CLAUDE.md` sempre-on especializado, templates (ADR/TASK/CHAT/PR/RUNBOOK/INDEX_*/MAPA),
  `settings.claude.example.json`, CI .NET (`ci/github-actions-ci.yml`), lint C#
  (`lint/Directory.Build.props` + `lint/.editorconfig` com analyzers Roslyn/nullable/warnings-as-errors),
  pre-commit (`hooks/.pre-commit-config.yaml` com `dotnet format`/build/gitleaks).

### Normativo coberto
- Rol sancionado de linguagens (Go/Rust/Elixir/C#/Zig/Ruby por fit + ADR §3); C# posicionado
  no ecossistema .NET/enterprise; Node-backend e PHP como legado/saída; frontend Node permitido
  (Blazor/UI delega ao `schematize-web`). Base agnóstica em `schematize-engineering`.
- Nullable reference types habilitado, arquivos ≤750 linhas + micro-funções, `///` XML doc em
  todo membro público (§6), índice de funcionalidades como fonte da verdade (§39).
- Q.A. plan-first (skill schematize-qa); handoff de contexto (§34.1); archive obrigatório (§28).
- Anti-padrões vetados (§37); testes/pentest com gate de "verde de verdade" (§22).
- IAM por desenho como app separada; ops/deploy destrutivo por seed; observabilidade LGTM.
