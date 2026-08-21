# Stack e Versões — C# / .NET

> **Verificado em: 2026-08-21** — o LTS corrente é o **.NET 10** (último patch **10.0.11**,
> 11/08/2026), com suporte até nov/2028. **.NET 8 e .NET 9 terminam em 10/11/2026** — menos de
> três meses a partir desta verificação. *(Este anexo dizia ".NET LTS atual — 8 ou 9"; além de
> vencido, era factualmente errado: o **9 é STS**, não LTS.)*
>
> Parte da skill **schematize-csharp**. Define o ferramental, as versões-alvo e o
> piso de toolchain do recorte C# (.NET) do rol sancionado. A escolha da linguagem
> (fit + ADR) está em `references/arquitetura.md` (§3) e no canônico agnóstico
> `schematize-engineering/references/linguagens.md`. Tudo o mais (arquitetura,
> segurança, testes, operação) segue os references comuns.

## 1. Versões-alvo (piso)

- **Runtime/SDK:** **.NET LTS atual — .NET 10** (`net10.0`). Sempre em versão **suportada
  pela Microsoft** (LTS preferida em produção; STS só com plano de upgrade). **EOL
  é dívida ativa:** rodar em versão fora de suporte é violação de cadeia de
  suprimentos (`references/cadeia-suprimentos.md`) — planeje o upgrade **antes** do
  fim de suporte.
- **Fixação da SDK:** `global.json` no repo pinando a banda da SDK (`rollForward`
  controlado) — build reproduzível em máquina e CI.
- **Linguagem:** `<LangVersion>latest</LangVersion>` alinhada ao TFM; use os
  recursos modernos (records, pattern matching, `required`, nullable) sem depender
  de preview em produção.
- **A versão exata em uso mora no Anexo A do projeto** (e no `global.json`).

## 2. Toolchain (piso — trava CI)

- **Build/CLI:** `dotnet` (restore/build/test/publish). Solution `.sln` por bounded
  context; **um `.csproj` por camada** (§ arquitetura §5).
- **Nullable reference types HABILITADO** (`<Nullable>enable</Nullable>`) — **piso
  inegociável** (§ padroes-codigo). Warning de nullable é erro.
- **Warnings como erro:** `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`.
  Não existe "amarelo tolerado".
- **Analyzers Roslyn:** `<EnableNETAnalyzers>true</EnableNETAnalyzers>` +
  `<AnalysisLevel>latest-recommended</AnalysisLevel>` + `.editorconfig` versionado
  (segurança CA, estilo). `<EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>`.
- **Formatação:** `dotnet format --verify-no-changes` no CI.
- **Doc XML:** `<GenerateDocumentationFile>true</GenerateDocumentationFile>` — o
  `///` de todo membro público alimenta o índice de microfunções (§39).
- Andaime pronto em `assets/lint/Directory.Build.props` + `assets/lint/.editorconfig`.

## 3. Frameworks e bibliotecas (defaults da casa)

| Papel | Default | Notas |
|---|---|---|
| **HTTP/API** | **ASP.NET Core** (Minimal APIs ou Controllers) | fronteira clara; DI nativo é o composition root |
| **DI** | **`Microsoft.Extensions.DependencyInjection`** (nativo) | sem container mágico; registro explícito |
| **ORM/dados** | **EF Core** (queries parametrizadas por desenho; migrations reversíveis via `dotnet ef`) | **Dapper** como opção para leitura/perf |
| **CQRS (opcional)** | **MediatR** | commands/queries (§8 arquitetura); não é obrigatório |
| **RPC** | **gRPC** (`Grpc.AspNetCore`) | contrato entre serviços |
| **Resiliência** | **Polly** (retry/circuit-breaker/timeout/bulkhead) | via `IHttpClientFactory` |
| **Config** | `Microsoft.Extensions.Configuration` + **user-secrets**/Key Vault/env | segredo nunca em `appsettings` commitado |
| **Log/telemetria** | **`ILogger`** estruturado (Serilog opção) + **OpenTelemetry .NET SDK** | → stack LGTM da casa |
| **Health** | `AddHealthChecks` (`/health`, `/ready`) | `/ready` valida dependência de verdade |

> Frameworks são bem-vindos; **abstração mágica não**. Critério: consigo ler o
> stack trace e entender o middleware/a query? EF Core e ASP.NET Core passam; o que
> esconde a query gerada ou o pipeline, não.

## 4. Segurança de linguagem/deps (piso — casa com cadeia-suprimentos.md)

- **Central Package Management:** `Directory.Packages.props` +
  `<ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>` — versão
  única por pacote em toda a solution.
- **Lockfile determinístico:** `<RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>`
  gera `packages.lock.json` (commitado); CI roda `dotnet restore --locked-mode`
  (falha se divergir). É **piso** de cadeia de suprimentos.
- **SCA:** `dotnet list package --vulnerable --include-transitive` no CI (fail on
  high/critical) + Dependabot/Renovate no feed NuGet.
- **Typosquatting NuGet é real:** dependência nova com **nome/autor/licença/versão
  verificados** antes de entrar; prefira pacotes com origem confiável e assinada.
- **SBOM** no build (CycloneDX: `dotnet CycloneDX`).
- **`unsafe`/interop nativo só com ADR** e bloco comentado explicando as
  invariantes; `AllowUnsafeBlocks` desligado por padrão.

## 5. Publish e imagem (casa com ops.md/seguranca.md)

- **`dotnet publish -c Release`**; runtime-dependent ou self-contained conforme o
  alvo. **AOT/trimming** só com ADR (avaliar refletir/serializar).
- **Imagem mínima:** base `mcr.microsoft.com/dotnet/aspnet` **chiseled/distroless**
  quando viável, **multi-stage**, **não-root**, filesystem read-only, healthcheck.
  Pinada por digest e assinada (Sigstore/cosign) — `references/cadeia-suprimentos.md`.

## 6. Pisos de código valem igual

Independente da linguagem, valem os limites de `references/padroes-codigo.md`:
arquivos ≤ 750 linhas (~500 de código útil + ~250 de comentário; flag em > 300
úteis), uma função/unidade lógica por arquivo, **todo** membro público com
doc-comment (**XML `///`** em C#) explicando motivo, comportamento esperado,
entradas, saídas e efeitos, e o **`MAPA.md`** atualizado no mesmo PR.

## 7. Coexistência com as outras skills

`schematize-csharp`, `schematize-go`, `schematize-rust` e `schematize-web` podem
estar habilitadas na mesma máquina ao mesmo tempo, sem conflito: cada skill instala
no seu diretório (`.claude/skills/schematize-<slug>/`) e seus comandos são
prefixados pelo slug (`/cs-*`, `/go-*`, `/rust-*`, `/web-*`). Escolha a skill pela
natureza do trabalho — C#/.NET aqui, e a linguagem certa por **fit** (§3). O
frontend (incluindo Blazor/UI) fica no `schematize-web`; a segurança ofensiva na
`schematize-pentest`.
