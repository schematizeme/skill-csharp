# Changelog — schematize-csharp

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
