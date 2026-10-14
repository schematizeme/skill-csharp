# Observabilidade — recorte C#/.NET

> **PONTEIRO, não cópia.** O padrão (OpenTelemetry, Collector, sinais correlacionados, SLO, alertas, dashboards
> como código, stack de referência) é da **`schematize-observability`** (`schematize-observability -> SKILL.md` e
> `references/`); o piso da base está em `schematize-engineering -> references/observabilidade.md`. Aqui fica
> **só o que muda em C#/.NET**. **Onde divergir, a base e a `schematize-observability` mandam**
> (`SKILL.md` §"Precedência e herança"); a numeração salta porque o número é o da base.

## 16. Observabilidade

- **Instrumentação:** **OpenTelemetry .NET SDK** (`OpenTelemetry.Extensions.Hosting`, exportador OTLP,
  instrumentações ASP.NET Core, `HttpClient` e EF Core) — traces, métricas e logs correlacionados por `trace_id`
  (`schematize-observability -> references/padrao-otel.md`). Spans de domínio com `ActivitySource`.

### 16.1 Logs

- `ILogger<T>` com logging estruturado (templates de mensagem, não interpolação de string) ou Serilog com sink JSON;
  escopos (`BeginScope`) carregam `trace_id`/`tenant_id`. Regras de conteúdo: `schematize-observability -> references/logs.md`.

### 16.2 Métricas

- `System.Diagnostics.Metrics` (`Meter`, `Counter<T>`, `Histogram<T>`), exportado pelo OpenTelemetry .NET SDK.
  Cardinalidade como orçamento: `schematize-observability -> references/metricas.md`.

## 17. Healthchecks

- `AddHealthChecks` + `MapHealthChecks` com tags: liveness sem dependência, readiness com as checagens de DB/fila.
  Contrato: `schematize-observability -> references/healthchecks.md`.
