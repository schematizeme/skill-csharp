# schematize-csharp

> Padrões normativos de engenharia da casa no recorte **C# / .NET** — ASP.NET Core, EF Core, .NET LTS (8/9). Arquitetura, segurança, IAM, testes/pentest, dados, observabilidade, deploy e archive.

Pacote de **skill normativa para [Claude Code](https://claude.com/claude-code)**.
Parte do catálogo **schematize skills** ([skills.schematize.me](https://skills.schematize.me)).

## Instalar

### Última versão (recomendado)

A partir de um clone do repositório:

```bash
git clone https://github.com/schematizeme/skill-csharp.git
cd skill-csharp && ./install.sh          # instala no projeto atual (diretório corrente)
# ./install.sh /caminho/do/projeto          # ou aponte para outro projeto
```

Ou baixe o `.zip` da última release e descompacte direto em `.claude/skills/`:

```bash
curl -L -o schematize-csharp.zip \
  https://github.com/schematizeme/skill-csharp/releases/latest/download/skill-csharp.zip
unzip schematize-csharp.zip -d .claude/skills/
```

### Uma versão específica

Cada versão tem três formas de obter: **(1)** um Release com `.zip` para baixar,
**(2)** uma pasta navegável em `versions/`, e **(3)** uma tag git.

| Versão | Data | Download (.zip) | Pasta navegável | Notas |
|---|---|---|---|---|
| **0.1.0** | 2026-08-15 | [release](https://github.com/schematizeme/skill-csharp/releases/download/v0.1.0/skill-csharp.zip) | [versions/v0.1.0/](versions/v0.1.0) | [CHANGELOG](CHANGELOG.md) |

```bash
# clonar uma versão exata pela tag:
git clone --branch v0.1.0 https://github.com/schematizeme/skill-csharp.git
```

> Todas as versões aparecem na página de **[Releases](https://github.com/schematizeme/skill-csharp/releases)**.

## Comandos

Todos prefixados por `cs-` — **sem conflito** com as outras skills na mesma máquina.

| Comando | O que faz |
|---|---|
| `/cs-help` | lista todos os comandos do schematize-csharp |
| `/cs-load` | carrega à força todo o corpo normativo no contexto |
| `/cs-claude` | cria/atualiza o `CLAUDE.md` da raiz com a versão da skill |
| `/cs-cc` | context compact (archive + `/compact`) |
| `/cs-handoff` | gera o handoff sem compactar |
| `/cs-qa` | Q.A. plan-first |
| `/cs-review` | gate da Definition of Done / anti-padrões no diff |
| `/cs-iam` | força/audita/scaffolda o IAM (microserviço C#/.NET separado) |
| `/cs-index` | (re)gera o índice de microfunções dos doc-comments `///` |
| `/cs-ops` | audita/scaffolda o `<projeto>_ops` |

Digite `/cs-help` dentro do Claude Code para ver a lista completa.

## Conteúdo da skill

- `SKILL.md` — porta de entrada e pisos inegociáveis.
- `references/` — corpo normativo fatiado por domínio (leia o que casa com a tarefa).
- `assets/` — templates (ADR/TASK/RUNBOOK/…), comandos, `CLAUDE.md`, CI, lint (.NET), hooks.
- `scripts/` — andaime de testes, índice e gestão de contexto.
- `skill.toml` — manifesto da skill (slug, nome, versão, descrições).

## Skills irmãs

- [skill-engineering](https://github.com/schematizeme/skill-engineering) — base agnóstica de engenharia.
- [skill-go](https://github.com/schematizeme/skill-go) — backend Go.
- [skill-rust](https://github.com/schematizeme/skill-rust) — backend Rust.
- [skill-web](https://github.com/schematizeme/skill-web) — frontend / SEO / performance (Blazor/UI incluídos).
- [skill-pentest](https://github.com/schematizeme/skill-pentest) — teste de segurança.

Podem ficar habilitadas ao mesmo tempo: os comandos são namespaced por skill
(`cs-*`, `go-*`, `rust-*`, `web-*`). A linguagem de backend se escolhe por **fit + ADR**
(rol sancionado Go/Rust/Elixir/C#/Zig/Ruby).

## Licença

[MIT](LICENSE) © 2026 schematizeme.
