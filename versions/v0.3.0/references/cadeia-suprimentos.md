# Cadeia de Suprimentos de Software (supply chain)


> **PONTEIRO, não cópia.** A normativa deste tema é da base: **`schematize-engineering`** →
> `references/cadeia-suprimentos.md`. Leia lá primeiro; aqui fica **só o que muda em C#/.NET**.
>
> **Onde este arquivo divergir da base, a BASE MANDA** (`SKILL.md` §"Precedência e herança").
> Em 2026-08-21 os blocos idênticos à base foram **podados mecanicamente** (`tools/podar-clone.mjs`),
> e é por isso que a numeração **salta**: o número é o da base, e o item ausente aqui é o que **não
> muda nesta linguagem** — procure-o lá.

## S1. Dependências

- **Lockfile commitado** (`packages.lock.json` via Central Package Management +
  `RestorePackagesWithLockFile` no .NET; `Cargo.lock` para binário, `go.sum`,
  `package-lock`), build **offline/reproduzível** a partir dele (`dotnet restore
  --locked-mode`). Sem range frouxo em dependência sensível.

- **Verificar nome e licença** de toda dependência nova: typosquatting é real
  (nome quase igual ao popular — vale para NuGet também); licença na allowlist
  (MIT/Apache-2.0/BSD/MPL-2.0/ISC ok; GPL/AGPL/SSPL/proprietária só com ADR).

- Atualização automatizada (Dependabot/Renovate — Dependabot cobre NuGet) com CI verde como gate.

- `cargo vendor`/proxy de módulos (feed NuGet interno) para builds críticos não dependerem de upstream no ar.

## S2. SBOM e vulnerabilidades

- **Gerar SBOM** (CycloneDX/SPDX) no build e **versioná-lo junto ao artefato** —
  é o inventário do que foi de fato embarcado. `dotnet CycloneDX` / `cargo cyclonedx`
  / `syft` servem.

- **Scan de vulnerabilidade no CI, travando o merge** em `high`/`critical` sem ADR
  de aceite: `dotnet list package --vulnerable --include-transitive` (.NET),
  `cargo audit`/`cargo deny` (Rust), `govulncheck` (Go), `npm audit`/SCA (Node).
  Scan também na imagem (`grype`/`trivy`).

## S3. Imagem de container

- **Base mínima e pinada por digest** (`@sha256:...`), não por tag móvel (`latest`/
  `1`). Distroless/Alpine/scratch quando viável; em .NET, `mcr.microsoft.com/dotnet/*`
  chiseled/distroless (assinada Sigstore/cosign, pinada por digest); menos pacote,
  menos CVE.
