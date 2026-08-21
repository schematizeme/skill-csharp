---
description: schematize-csharp — carrega à força TODO o corpo normativo (DDD/arquitetura, clean code, segurança, dados, testes, operação) no contexto e passa a aplicá-lo no projeto atual como regra inegociável
---

Carregue **à força** e passe a aplicar **integralmente** os Padrões de Engenharia da Casa (skill `schematize-csharp`) neste projeto. A partir de agora, nesta sessão, isto **não é opcional**.

1. **Leia agora, na íntegra, TODOS os arquivos** de references da skill — não trabalhe de memória, abra cada arquivo. O caminho é `.claude/skills/schematize-csharp/references/*.md` (instalação no projeto) ou `~/.claude/skills/schematize-csharp/references/*.md` (instalação global). Com destaque para:
   - `padroes-codigo.md` — **clean code**: arquivo ≤750 linhas (~500 úteis + ~250 comentário; flag >300 úteis, ~400 obs), uma função/unidade por arquivo, XML doc-comment (`///`) obrigatório em membro público (motivo/comportamento/I-O), `MAPA.md`.
   - `arquitetura.md` — **DDD**, camadas, repositórios, bounded contexts, anti-monólito, shared libs, CQRS, escolha de linguagem.
   - `stack-versoes.md` — versões suportadas do runtime (.NET LTS/STS, C# lang version), EOL, `nullable reference types` habilitado, `<TreatWarningsAsErrors>`, target frameworks.
   - `async-concorrencia.md` — `async`/`await`, `Task`/`ValueTask`, `CancellationToken` propagado, `IAsyncEnumerable`, `ConfigureAwait`, `Channel<T>`, evitar `async void`/`.Result`/`.Wait()` (deadlock), `IHostedService`/`BackgroundService`.
   - `seguranca.md` — auth/JWT, multi-tenancy, LGPD, segredo nunca no cliente, SQL parametrizado (EF Core / Dapper com parâmetros, nunca interpolação).
   - `iam.md` — **IAM da casa (recorte C#/.NET)**: auth como microserviço **C#/.NET (ASP.NET Core)** separado (`auth.<domain>`), ID≠email, ≥2 fatores (passkey Fido2 .NET/Resend/Twilio), ReBAC multi-tenant deny-default (PEP=middleware/policy ASP.NET Core), sessão 7d/90d, logout irreversível, migração de legado prioridade 0.
   - `dados-eventos.md` — eventos/mensageria, banco (EF Core / migrations), cache, APIs, resiliência (Polly), jobs.
   - `cadeia-suprimentos.md` — lockfile (`packages.lock.json`), SBOM, scan que trava, imagem mínima/pinada/assinada, SLSA.
   - `testes.md` — test kit (xUnit), "verde de verdade", pentest, Q.A. plan-first.
   - `observabilidade.md` — healthchecks, performance, FinOps, **OpenTelemetry .NET** (traces/metrics/logs).
   - `operacao.md` + `entrega.md` — config, deploy/K8s, git/PR, runbooks, ADR, **archive**, DoD, índice.
   - `ops.md` — **control plane `<projeto>_ops`**: fluxo dev→local→github→hml→prd (nada direto no servidor), ops como interface única (100%, autônomo), instalação paralela=`nproc`, independência=invariante (prioridade máxima).
   - `anti-padroes.md` — a lista completa de anti-padrões vetados (§37).
   - `contexto-claude-code.md` — gestão de contexto/handoff em sessões longas.

2. **Confirme ao usuário** que leu, com **1 linha por arquivo** resumindo o piso central de cada um.

3. Deste ponto em diante, **aplique estes padrões como regra inegociável** em toda decisão, geração e revisão de código deste projeto — arquitetura/DDD, clean code, segurança, testes e archive. Em conflito entre "fazer rápido" e o padrão, **o padrão vence**.

4. **Atualize o `CLAUDE.md` da raiz** do repositório com a versão atual de `assets/CLAUDE.md` da skill — **sobrescreva mesmo se já existir** (rodar não pode deixar a versão antiga). Se o `CLAUDE.md` atual tiver customização local (seções fora do template da skill), salve backup `./CLAUDE.md.bak` e reaplique as customizações por cima do template novo. Se não existir, crie. É o mesmo que o comando `/cs-claude`. Confirme a versão aplicada.
