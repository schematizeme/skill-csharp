# observabilidade — recorte C#/.NET

> **PONTEIRO, não cópia.** A normativa deste tema é da base: **`schematize-engineering`** →
> `references/observabilidade.md`. Leia lá primeiro; aqui fica **só o que muda em C#/.NET**.
>
> **Onde este arquivo divergir da base, a BASE MANDA** (`SKILL.md` §"Precedência e herança").
> Este arquivo era um clone com **93%** de conteúdo
> idêntico à base — deriva por cópia foi o achado da Classe C da vistoria de 2026-08-21, e ela
> já tinha atingido piso de segurança (o `argon2id-only` da casa virou "ou PBKDF2" numa skill,
> o rol de 6 linguagens virou "só Go e Rust" em três). Manter uma cópia é manter a próxima deriva.
## 16. Observabilidade

- **Instrumentação:** OpenTelemetry — em .NET, o **OpenTelemetry .NET SDK** (traces, métricas e logs correlacionados por `trace_id`; exemplars ligando métrica↔trace).

### 16.1 Logs

- JSON estruturado (em .NET, `ILogger` com logging estruturado / Serilog + sink JSON).

### 16.2 Métricas

- Instrumentação via `System.Diagnostics.Metrics` (exportada pelo OpenTelemetry .NET SDK).

## 17. Healthchecks

Endpoints obrigatórios (em .NET, expostos via `AddHealthChecks` + mapeamento das rotas):
