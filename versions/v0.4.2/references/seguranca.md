# Segurança, Auth, Multi-tenancy, LGPD e Frontend


> **PONTEIRO, não cópia.** A normativa deste tema é da base: **`schematize-engineering`** →
> `references/seguranca.md`. Leia lá primeiro; aqui fica **só o que muda em C#/.NET**.
>
> **Onde este arquivo divergir da base, a BASE MANDA** (`SKILL.md` §"Precedência e herança").
> Em 2026-08-21 os blocos idênticos à base foram **podados mecanicamente** (`tools/podar-clone.mjs`),
> e é por isso que a numeração **salta**: o número é o da base, e o item ausente aqui é o que **não
> muda nesta linguagem** — procure-o lá.

## 13. Segurança

- Imagem base `mcr.microsoft.com/dotnet/aspnet` chiseled/distroless (ou chainguard) quando viável.

- Multi-stage build com `dotnet publish` (restore/build/publish num estágio, runtime enxuto no final).

- Nunca em código, nunca em env file commitado, **nunca em `appsettings.json` commitado nem no bundle publicado**.

- Storage: **user-secrets** (dev), Azure Key Vault / AWS/GCP Secret Manager, Vault, sealed-secrets, ou env injetada em runtime.

- **Data Protection API** (`IDataProtectionProvider`) para proteger payloads sensíveis em repouso/trânsito interno (chaves persistidas fora do container, protegidas por Key Vault/HSM em produção).

**MUST — acesso a dados**

- SQL **sempre parametrizado**: **EF Core** (LINQ / `FromSqlInterpolated` com parâmetros) ou **Dapper** com parâmetros nomeados (`@id`). String interpolation/concatenação montando query (`$"... WHERE id = {id}"` cru, `FromSqlRaw` com string montada) é VETADO (§37).

- Nunca expor `DbContext`/entidade diretamente na borda HTTP — DTO/projeção explícita (evita over-posting / mass assignment).

**MUST — transporte e CSRF**

- **HTTPS obrigatório** (`UseHttpsRedirection`) + **HSTS** (`UseHsts`) em produção.

- **Antiforgery** (`AddAntiforgery` + validação) em toda mutação servida a browser com cookie de sessão (CSRF).

- **CORS explícito** (`AddCors` com origens declaradas) — nunca `AllowAnyOrigin` em rota autenticada.

### 13.4 Segredos no Cliente / Frontend / NextJS

> O backend .NET **expõe API** e não serve UI própria: o frontend delega ao **schematize-web** (Next/Astro). As regras abaixo valem para qualquer cliente que consome essa API.

- Guardar token de sessão em `localStorage`/`sessionStorage`. Sessão vai em **cookie `HttpOnly` + `Secure` + `SameSite`** (§38, a `schematize-qa` (`references/categorias.md` §§5 e 10) hardening).

## 14. Autenticação e Autorização

- Validação **completa** do JWT em toda request via **`JwtBearer` + `TokenValidationParameters`** com tudo ligado: `ValidateIssuerSigningKey`, `ValidateLifetime` (`exp`/`nbf`), `ValidateAudience` (`aud`), `ValidateIssuer` (`iss`) e `ValidAlgorithms` como allowlist explícita. Decodar o payload e confiar sem validar é VETADO (§37). **Use `JsonWebTokenHandler`** (namespace `Microsoft.IdentityModel.JsonWebTokens`), que é o handler default do ASP.NET Core desde a 8: o **`JwtSecurityTokenHandler` é legado** — mais lento, e é ele que mapeia os claims para os nomes longos do WS-Federation (`http://schemas.xmlsoap.org/...`), o que faz `User.FindFirst("sub")` voltar `null` e leva alguém a "resolver" lendo o claim errado. `JsonWebTokenHandler.ValidateTokenAsync` preserva os nomes originais.

- RBAC com permissões granulares (policies/`AuthorizationHandler`); ABAC quando necessário.

- **ASP.NET Core Identity não é o IAM da casa** — o IAM é uma app separada (ver `references/iam.md` §1, *"Topologia — auth é uma APLICAÇÃO SEPARADA"*). Identity pode servir de biblioteca de primitivas, nunca como a fronteira de autenticação do sistema.

- Hash de senha: **argon2id** (via `Konscious.Security.Cryptography`) — a casa é **argon2id-only**
  para senha nova, com os parâmetros mínimos da base
  (`schematize-engineering` -> `references/iam.md` §2.1). **PBKDF2 é legado a migrar**, e se ele
  aparecer em código antigo, use a **API estática** `Rfc2898DeriveBytes.Pbkdf2(...)`: o **construtor**
  está obsoleto como **`SYSLIB0041`** e, com o `TreatWarningsAsErrors` que esta skill exige
  (`stack-versoes.md`), o código que segue o conselho antigo **nem compila** — a skill prescrevia
  algo que ela mesma proíbe compilar. MD5/SHA1/sem-salt/plaintext são VETADOS (§37).

- Tokens, ids de sessão e códigos de reset gerados por **CSPRNG** (`System.Security.Cryptography.RandomNumberGenerator`) — **nunca `System.Random`/`Random`** (§37).

## 14.1 Rate limiting — piso, e no framework (não numa gem)

Endpoint público sem limite é DoS barato e, em fluxo de auth, é **amplificador de envio** (cada
tentativa vira um e-mail/SMS — `iam.md` §3.1). No .NET isso não precisa de dependência: o
**middleware `AddRateLimiter`** faz parte do framework desde o .NET 7.

```csharp
builder.Services.AddRateLimiter(o =>
{
    // 429 explícito com Retry-After: silêncio ou 500 aqui vira "a API caiu" no suporte.
    o.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
    o.OnRejected = async (ctx, ct) =>
    {
        if (ctx.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retry))
            ctx.HttpContext.Response.Headers.RetryAfter = ((int)retry.TotalSeconds).ToString();
        await ctx.HttpContext.Response.WriteAsync("rate limit", ct);
    };

    // Auth: janela CURTA por (rota + identificador + IP). O particionador é o ponto —
    // limitar "a rota" globalmente pune todo mundo por causa de um atacante.
    o.AddPolicy("auth", ctx => RateLimitPartition.GetFixedWindowLimiter(
        partitionKey: $"{ctx.Request.Path}|{ctx.Request.Headers["X-Forwarded-For"]}|{ctx.User.Identity?.Name ?? "anon"}",
        _ => new FixedWindowRateLimiterOptions { PermitLimit = 5, Window = TimeSpan.FromMinutes(1) }));

    // API geral: token bucket permite rajada curta sem liberar sustentado.
    o.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(ctx =>
        RateLimitPartition.GetTokenBucketLimiter(ctx.Connection.RemoteIpAddress?.ToString() ?? "sem-ip",
            _ => new TokenBucketRateLimiterOptions
            {
                TokenLimit = 100, TokensPerPeriod = 100,
                ReplenishmentPeriod = TimeSpan.FromMinutes(1), QueueLimit = 0,
            }));
});
app.UseRateLimiter();
```

**MUST**
- **Endpoint de auth** (login, cadastro, OTP, reset, reenvio, convite) tem limite **por
  destinatário + IP + conta**, não só por IP: limitar só por IP não segura o ataque distribuído e
  pune quem está atrás de NAT.
- **Atrás de proxy**, configure `ForwardedHeaders` **com a lista de proxies confiáveis** — senão o
  `X-Forwarded-For` é escolhido pelo atacante e a partição vira inútil (pior: vira um jeito de
  esgotar a memória do limitador criando partições infinitas).
- **429 com `Retry-After`**, sempre. E o limite **aparece na métrica** — rate limit que ninguém
  observa é indistinguível de indisponibilidade (`observabilidade.md`).
- **O limite do processo não é o limite do sistema:** com N réplicas, cada uma tem o seu. Onde o
  número precisa ser global (OTP, cobrança), o contador vai para o store compartilhado.

## 38. Frontend / NextJS / SPA — Regras Específicas

> O backend .NET **expõe API**; a UI é responsabilidade do **schematize-web** (Next/Astro, TypeScript strict). As regras de fronteira client/server abaixo governam esse frontend e o contrato que a API .NET oferece a ele.

- CSP, `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy` e `frame-ancestors` configurados (hardening; ver a `schematize-qa`, `references/categorias.md` §§5 e 10).

- Confiar em `redirect`/`next` param sem allowlist (open redirect — a `schematize-qa` (`references/categorias.md` §§5 e 10)).
