# IAM — Identidade e Autorização da casa (piso inegociável) — recorte backend/C#

## 1. Topologia — auth é uma APLICAÇÃO SEPARADA (microserviço C#/.NET)

- **A autenticação é um serviço próprio, com link próprio e front próprio**, servido em
  **`auth.<domain>`**. **VETADO** apensar o auth ao escopo principal como monolith (nem
  como um projeto `Auth` interno do serviço principal). **ASP.NET Core Identity NÃO é o
  IAM da casa:** o IAM é uma **aplicação de auth separada** (`<projeto>_auth_cs`, IdP
  próprio), **não** o Identity padrão embutido no app de negócio.

- **Microserviço de auth em C#/.NET (ASP.NET Core)** (`<projeto>_auth_cs`) + **front de auth**
  próprio (`<projeto>_authfront`), com **repo, deploy, user Linux e systemd/container isolados**
  por conta própria (casa com o isolamento por app do `ops.md` §3). Comprometer o app
  principal **não** compromete o IdP. O binário do auth é o **único** que carrega a chave
  privada de assinatura e os segredos de provedor (Resend/Twilio/OAuth).

- **O app principal (e todo cliente) delega ao auth por OIDC/OAuth2.1 + PKCE:** redireciona
  pra `auth.<domain>`, recebe tokens de volta. O serviço C#/.NET de auth é o **IdP da casa**
  (padrão self-hosted, consumido por N apps) — expõe os endpoints padrão
  (`/authorize`, `/token`, `/introspect`, `/revoke`, `/.well-known/openid-configuration`,
  `/.well-known/jwks.json`).

- **Segredos e chave de assinatura de token vivem SÓ no serviço de auth C#**; consumidores
  (middleware ASP.NET Core dos outros serviços) validam por **JWKS público** cacheado — nunca
  guardam a chave privada. A assinatura JWT (Ed25519/EdDSA ou RS256, via
  `Microsoft.IdentityModel`/`System.IdentityModel.Tokens.Jwt`) só acontece dentro
  de `<projeto>_auth_cs`; a rotação de chave é publicada via `kid` no JWKS.

## 2. Modelo de identidade

- **ID interno imutável e opaco** (ULID/UUIDv7) é o `sub`. **Email e telefone NUNCA são
  ID** — são *identificadores* ligados ao usuário, cada um com estado de verificação. Em
  C#, o agregado `User` tem `Id` opaco; `Email`/`Phone` são entidades filhas com
  `VerifiedAt` (`DateTimeOffset?`, nullable reference types habilitado), nunca a PK.

## 3. Fatores e níveis de garantia (AAL — NIST 800-63B)

- **Email OTP (Resend) ligado por padrão, inclusive em HML** — só o operador desliga.

- **Provedores plugáveis como interfaces C#:** o core depende de interfaces, não de SDK:

  ```csharp
  public interface IEmailProvider { Task SendOtpAsync(string to, string code, CancellationToken ct); }
  public interface ISmsProvider   { Task SendOtpAsync(string to, string code, CancellationToken ct); }
  public interface IPushProvider  { Task<Approval> RequestApprovalAsync(string deviceToken, Challenge challenge, CancellationToken ct); }
  ```

  `IEmailProvider` (Resend default), `ISmsProvider` (Twilio default), `IPushProvider` são
  **trocáveis via DI nativo (config/wiring)**, sem tocar no core (adaptadores na borda, DDD).

- **WebAuthn/passkey** por lib madura (ex.: **Fido2 .NET lib**); **TOTP** por
  **Otp.NET**. Assinatura/JWT por **`Microsoft.IdentityModel`/`System.IdentityModel.Tokens.Jwt`** (JWKS).

- **Senha por padrão, opcional por escolha:** o usuário **cria senha no cadastro** (padrão
  cultural; **argon2id** via **Konscious.Security.Cryptography**, com os parâmetros mínimos da
  base (`schematize-engineering`, seção 2.1 do `references/iam.md` dela) + verificação contra base de
  vazadas/HIBP k-anonymity), mas o
  **seletor de modos de autenticação permite marcá-la como opcional** e viver de passkey/OTP/app.
  > **PBKDF2 (`Rfc2898DeriveBytes`) NÃO é opção viva.** A casa é **argon2id-only** para senha nova
  > — é a posição da base e das outras 6 skills do rol. PBKDF2 só aparece como **legado a migrar**:
  > re-hash preguiçoso no próximo login bem-sucedido, com o algoritmo gravado junto do hash. Deixar
  > PBKDF2 como alternativa aceitável nesta skill (e só nela) era deriva por cópia, não decisão.
  > **Parâmetros mínimos (`m` ≥ 19 MiB, `t` ≥ 2, `p` = 1, salt ≥ 16 B CSPRNG, string codificada
  > guardada inteira, re-hash preguiçoso):** na base — `schematize-engineering`, seu
  > `references/iam.md`, seção **2.1 ("Hash de senha — argon2id com parâmetros MÍNIMOS")**.
  > **Nenhuma das 8 skills fixava os números** até 2026-08-21 — e argon2id mal parametrizado é
  > mais fraco que bcrypt bem configurado. Calibre para ~0,5–1 s no hardware do auth e registre
  > o valor medido no ADR do serviço; o default da lib normalmente é o mais fraco. Na `Konscious`, `MemorySize`/`Iterations`/`DegreeOfParallelism` são obrigatórios de setar.

- **2FA por desenho desde o cadastro — senha + Email OTP JÁ é 2FA (fraco, porém válido):**
  a conta **nasce com dois fatores obrigatórios** (senha + código no email verificado,
  always-on) e **já é segura para o baseline**. **VETADO** tratar senha+OTP como "sem 2FA" e
  **barrar o login** até enrolar um fator forte — é o **círculo infinito**. Em C#: o middleware
  de sessão **libera o acesso baseline** com a sessão de AAL médio (senha+OTP); **não** barra
  todas as rotas exigindo AAL alto.

- **Fator forte é INCENTIVADO + just-in-time, nunca muro pré-login:** app OTP / passkey / chave
  são **nudge** e **exigidos só na operação sensível** (o PEP checa o AAL mínimo **por rota
  sensível** e dispara **step-up** ali) e **escalados sob risco** (§9). Enrolar um fator forte
  usa o Email OTP como verificação (Y≠X, §4): sem deadlock. A ausência de fator forte **degrada
  o que é sensível** (`403 step_up_required`), não **bloqueia o baseline**.

## 3.1 Disparo do Email OTP fora de prd — sink por default e guard como decorator no DI

O Email OTP é **always-on** (§3): é o auth que manda e-mail em **todo** cadastro, login, troca
de fator e recuperação — inclusive em dev/hml e em **cada volta** de um laço de teste. Por isso
o piso de **efeito externo** da casa incide **primeiro aqui**. A normativa completa (4 camadas,
DNS do domínio de teste, exceção com ADR, runbook de contenção) é a **`schematize-engineering`
→ `references/efeitos-externos.md`**; abaixo está **só o recorte C#/.NET**.

**Regra:** fora de `prd`, **nenhum e-mail chega em ninguém**. Destinatário sintético só no
**domínio de teste em ROTA NULA** (`test.<domain>` com null MX (RFC 7505) + SPF `v=spf1 -all` +
DMARC `p=reject`, ou TLD reservado `.test`/`.invalid`/`.example`). **VETADO** `@gmail.com` — ou
qualquer caixa real, **inclusive a sua** — em fixture, seed, persona ou demo. **Motivo:**
bounce/complaint em massa **queima o IP e o domínio**, derruba o transacional de produção — a
começar pelo **próprio OTP de login desta §3** — e custa semanas de warm-up. Não tem undo.

### `MailOptions` — ambiente **declarado** e validação no boot (`ValidateOnStart`)

**Cuidado .NET (fail-OPEN clássico):** `IHostEnvironment.EnvironmentName` cai em `"Production"`
quando `ASPNETCORE_ENVIRONMENT`/`DOTNET_ENVIRONMENT` **não** está definido. Decidir o provider
por `env.IsProduction()` faz o container sem variável **mandar de verdade**. Por isso o ambiente
é um valor **declarado na config** e **ausente significa NÃO-produção** (fail-closed).

```csharp
// Auth/Mail/MailOptions.cs — <Nullable>enable</Nullable> (piso de linguagem).
using System.ComponentModel.DataAnnotations;

namespace Auth.Mail;

/// <summary>Configuração de saída de e-mail. Vinculada à seção <c>Mail:</c> e validada no boot.</summary>
/// <remarks>
/// O ambiente é DECLARADO aqui, nunca inferido de <see cref="Microsoft.Extensions.Hosting.IHostEnvironment"/>:
/// sem ASPNETCORE_ENVIRONMENT o .NET assume "Production" — fail-OPEN. Aqui, valor ausente ou
/// desconhecido significa não-produção, que é o modo seguro.
/// </remarks>
public sealed class MailOptions
{
    public const string SectionName = "Mail";

    /// <summary>"dev" | "hml" | "prd". Ausente/desconhecido ⇒ tratado como NÃO-produção.</summary>
    public string EnvironmentName { get; init; } = "dev";

    /// <summary>Domínios aceitos como destinatário fora de prd (rota nula / TLD reservado).</summary>
    public IReadOnlyList<string> TestDomains { get; init; } =
        ["test.example.com", "test", "invalid", "example"];

    /// <summary>Allowlist nominal da exceção com ADR aceito (≤5 endereços da casa). Vazia por padrão.</summary>
    public IReadOnlyList<string> Allowlist { get; init; } = [];

    /// <summary>Teto de envios por execução do processo (MAIL_MAX_PER_RUN).</summary>
    [Range(1, 10_000)]
    public int MaxPerRun { get; init; } = 50;

    /// <summary>Verdadeiro só quando o ambiente é DECLARADAMENTE "prd".</summary>
    public bool IsProduction =>
        string.Equals(EnvironmentName, "prd", StringComparison.OrdinalIgnoreCase);
}
```

```csharp
// Program.cs — config errada NÃO sobe o serviço (falha no start, não na primeira request).
builder.Services.AddOptions<MailOptions>()
    .Bind(builder.Configuration.GetSection(MailOptions.SectionName))
    .ValidateDataAnnotations()
    .Validate(o => o.IsProduction || o.TestDomains.Count > 0,
        "Mail:TestDomains vazio fora de prd — nem o teste conseguiria entregar.")
    .Validate(o => o.Allowlist.Count <= 5,
        "Mail:Allowlist só existe com ADR aceito e no máximo 5 endereços da casa.")
    .ValidateOnStart();
```

### Providers: real, sink e o **guard como decorator**

O core depende de `IEmailProvider` (§3), nunca do SDK. O **sink** é o default fora de prd
(**Mailpit** — SMTP local + **API HTTP** para o teste ler a caixa — ou log estruturado); a chave
de não-prd é **sandbox**, nunca a de prd, e o **egress SMTP (25/465/587) fica bloqueado** em
dev/hml (`schematize-infra`).

```csharp
/// <summary>Provider real (Resend). Registrado SÓ quando o ambiente declarado é "prd".</summary>
public sealed class ResendEmailProvider(HttpClient http) : IEmailProvider { /* ... */ }

/// <summary>Sink: nada sai da máquina. Default fora de prd.</summary>
public sealed class SinkEmailProvider(ILogger<SinkEmailProvider> logger) : IEmailProvider
{
    /// <inheritdoc />
    public Task SendOtpAsync(string to, string code, CancellationToken ct)
    {
        logger.LogInformation("[mail:sink] OTP destinado a {Recipient} — nada foi enviado", to);
        return Task.CompletedTask;
    }
}

/// <summary>Recusa tipada: destinatário externo fora de produção. Nada foi enviado.</summary>
public sealed class ExternalRecipientBlockedException : InvalidOperationException
{
    /// <summary>Destinatário recusado (entra no log/métrica; nunca é reenviado "por fora").</summary>
    public string Recipient { get; init; } = string.Empty;

    /// <summary>Ambiente declarado no momento da recusa.</summary>
    public string EnvironmentName { get; init; } = string.Empty;

    // Construtores padrão exigidos pelos analyzers da casa (CA1032, AnalysisMode=AllEnabledByDefault).
    public ExternalRecipientBlockedException() { }

    public ExternalRecipientBlockedException(string message) : base(message) { }

    public ExternalRecipientBlockedException(string message, Exception innerException)
        : base(message, innerException) { }

    /// <summary>Forma usada pelo guard: mensagem acionável (ensina o caminho certo) + contexto.</summary>
    public static ExternalRecipientBlockedException For(string recipient, string environmentName) =>
        new($"Envio BLOQUEADO para '{recipient}' em env='{environmentName}'. Nada foi enviado. " +
            "Use <papel>+<run-id>-<n>@test.<domain> ou registre o ADR de exceção.")
        {
            Recipient = recipient,
            EnvironmentName = environmentName,
        };
}

/// <summary>Teto de envios por execução estourado — a execução aborta.</summary>
public sealed class MailCapExceededException : InvalidOperationException
{
    /// <summary>Teto configurado em <c>Mail:MaxPerRun</c>.</summary>
    public int Cap { get; init; }

    public MailCapExceededException() { }

    public MailCapExceededException(string message) : base(message) { }

    public MailCapExceededException(string message, Exception innerException)
        : base(message, innerException) { }

    /// <summary>Forma usada pelo cap por execução.</summary>
    public static MailCapExceededException ForCap(int cap) =>
        new($"Teto de {cap} e-mails por execução estourado — execução abortada.") { Cap = cap };
}

/// <summary>
/// Contador de envios da EXECUÇÃO (processo). Registrado como <b>singleton</b> de propósito:
/// contador em campo de instância de um serviço transiente (typed client) não conta nada.
/// </summary>
public sealed class MailRunQuota(IOptions<MailOptions> options)
{
    private int _sent;

    /// <summary>Consome uma unidade do teto; estourou, lança <see cref="MailCapExceededException"/>.</summary>
    public void Take()
    {
        var cap = options.Value.MaxPerRun;
        if (Interlocked.Increment(ref _sent) > cap)
        {
            throw MailCapExceededException.ForCap(cap);
        }
    }

    /// <summary>Zera o contador. Só em teste.</summary>
    public void Reset() => Interlocked.Exchange(ref _sent, 0);
}

/// <summary>
/// Decorator deny-by-default de <see cref="IEmailProvider"/>: valida o destinatário e o teto
/// por execução ANTES de delegar. O guard mora aqui — nunca no chamador, que esquece.
/// </summary>
public sealed class GuardedEmailProvider(
    IEmailProvider inner,
    MailRunQuota quota,
    IOptions<MailOptions> options,
    ILogger<GuardedEmailProvider> logger) : IEmailProvider
{
    private readonly MailOptions _options = options.Value;

    /// <inheritdoc />
    public async Task SendOtpAsync(string to, string code, CancellationToken ct)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(to);
        EnsureDeliverable(to);
        quota.Take();
        await inner.SendOtpAsync(to, code, ct).ConfigureAwait(false);
    }

    // Fora de prd só passa domínio de teste ou allowlist nominal (ADR). Erro, nunca no-op.
    private void EnsureDeliverable(string to)
    {
        if (_options.IsProduction) { return; }
        if (IsTestDomain(to)) { return; }
        if (_options.Allowlist.Contains(to, StringComparer.OrdinalIgnoreCase)) { return; }

        // Redigido: log de auditoria identifica o envio, NÃO expõe a caixa. O código do
        // OTP não entra em log de nível nenhum — o teste o lê pela API do sink.
        logger.LogError("mail BLOQUEADO: {Recipient} em env={Environment}", Redact(to), _options.EnvironmentName);
        throw ExternalRecipientBlockedException.For(to, _options.EnvironmentName);
    }

    /// <summary>`alguem@acme.com` vira `al***@acme.com`. Sem "@": `***`.</summary>
    internal static string Redact(string address)
    {
        var at = address.LastIndexOf('@');
        if (at <= 0) { return "***"; }
        var keep = Math.Min(at, 2);
        return string.Concat(address.AsSpan(0, keep), "***", address.AsSpan(at));
    }

    // Endereço malformado (sem "@" ou sem domínio) NÃO passa — fail-closed.
    private bool IsTestDomain(string to)
    {
        var at = to.LastIndexOf('@');
        if (at <= 0 || at == to.Length - 1) { return false; }

        var domain = to.AsSpan(at + 1);
        foreach (var allowed in _options.TestDomains)
        {
            if (domain.Equals(allowed, StringComparison.OrdinalIgnoreCase) ||
                domain.EndsWith($".{allowed}", StringComparison.OrdinalIgnoreCase))
            {
                return true;
            }
        }

        return false;
    }
}
```

Os snippets assumem o `assets/lint/Directory.Build.props` da casa (`AnalysisMode=AllEnabledByDefault`

+ `TreatWarningsAsErrors`): por isso as exceções trazem os **construtores padrão** (CA1032) e as
propriedades como `init`. No caminho quente, troque `logger.LogError(...)` por um
`[LoggerMessage]` partial (CA1848) — a recusa é evento raro, o log do sink não.

### A seleção é na composição (`Program.cs`), nunca no chamador

```csharp
// Program.cs — a ESCOLHA do provider acontece AQUI, uma vez. Nenhum caso de uso escolhe.
var mail = builder.Configuration.GetSection(MailOptions.SectionName).Get<MailOptions>() ?? new MailOptions();

builder.Services.AddSingleton<MailRunQuota>();

if (mail.IsProduction)
{
    // Typed client: o HttpClientFactory cuida do handler (nada de HttpClient singleton na mão).
    builder.Services.AddHttpClient<IEmailProvider, ResendEmailProvider>(c =>
    {
        c.BaseAddress = new Uri("https://api.resend.com/");
        c.Timeout = TimeSpan.FromSeconds(5);
    });
}
else
{
    builder.Services.AddSingleton<IEmailProvider, SinkEmailProvider>();
}

// O guard decora SEMPRE — inclusive em prd, onde ele só conta e libera.
builder.Services.Decorate<IEmailProvider, GuardedEmailProvider>(); // Scrutor
```

- **Sem Scrutor**, o mesmo efeito por factory — registre o inner concreto e componha à mão,
  respeitando o lifetime do typed client:

  ```csharp
  builder.Services.AddSingleton<SinkEmailProvider>();
  builder.Services.AddHttpClient<ResendEmailProvider>();
  builder.Services.AddTransient<IEmailProvider>(sp => new GuardedEmailProvider(
      inner: mail.IsProduction
          ? sp.GetRequiredService<ResendEmailProvider>()
          : sp.GetRequiredService<SinkEmailProvider>(),
      quota: sp.GetRequiredService<MailRunQuota>(),
      options: sp.GetRequiredService<IOptions<MailOptions>>(),
      logger: sp.GetRequiredService<ILogger<GuardedEmailProvider>>()));
  ```

- **VETADO** registrar `ResendEmailProvider` "e trocar depois", injetar `HttpClient` do Resend em
  qualquer outro serviço, ou expor um parâmetro `bypassGuard`/`force: true` — bypass por
  parâmetro é o mesmo bug com outro nome.

- Em teste de integração, o `WebApplicationFactory` **não precisa** trocar nada: o ambiente
  declarado já é não-prd, então o sink + guard valem por construção.

### O teste que vê o vermelho (xUnit)

O guard sem teste é intenção. O teste **tenta** o `@gmail.com` e **espera a exceção** — e prova
que **nada** chegou ao inner.

```csharp
public sealed class GuardedEmailProviderTests
{
    private sealed class RecordingEmailProvider : IEmailProvider
    {
        public List<string> Sent { get; } = [];

        public Task SendOtpAsync(string to, string code, CancellationToken ct)
        {
            Sent.Add(to);
            return Task.CompletedTask;
        }
    }

    private static GuardedEmailProvider Build(IEmailProvider inner, MailOptions options)
    {
        var opts = Options.Create(options);
        return new GuardedEmailProvider(inner, new MailRunQuota(opts), opts,
            NullLogger<GuardedEmailProvider>.Instance);
    }

    [Fact]
    public async Task Bloqueia_destinatario_externo_fora_de_prd()
    {
        var inner = new RecordingEmailProvider();
        var sut = Build(inner, new MailOptions { EnvironmentName = "hml" });

        var ex = await Assert.ThrowsAsync<ExternalRecipientBlockedException>(
            () => sut.SendOtpAsync("vitima@gmail.com", "123456", CancellationToken.None));

        ex.Recipient.Should().Be("vitima@gmail.com");
        inner.Sent.Should().BeEmpty(); // prova que NADA saiu
    }

    [Fact]
    public async Task Entrega_para_o_dominio_de_teste_em_rota_nula()
    {
        var inner = new RecordingEmailProvider();
        var sut = Build(inner, new MailOptions { EnvironmentName = "hml" });

        await sut.SendOtpAsync("login+run42-1@test.example.com", "123456", CancellationToken.None);

        inner.Sent.Should().ContainSingle();
    }

    [Fact]
    public async Task Aborta_ao_estourar_o_cap_por_execucao()
    {
        var inner = new RecordingEmailProvider();
        var sut = Build(inner, new MailOptions { EnvironmentName = "hml", MaxPerRun = 2 });
        const string to = "carga+run42-1@test.example.com";

        await sut.SendOtpAsync(to, "1", CancellationToken.None);
        await sut.SendOtpAsync(to, "2", CancellationToken.None);

        await Assert.ThrowsAsync<MailCapExceededException>(
            () => sut.SendOtpAsync(to, "3", CancellationToken.None));
        inner.Sent.Should().HaveCount(2);
    }
}
```

O mesmo desenho vale para `ISmsProvider` (Twilio → magic numbers + sink; SMS **custa por
unidade**, o cap importa mais) e `IPushProvider` (APNs sandbox / projeto FCM de teste). Seeds e
personas em a `schematize-qa` (seeds e personas, `references/execucao.md` secao 4)).

## 4. Fluxos

**Onboarding:** cita um email → **verifica** → **cria senha** (ou já passkey/app) → **pronto:
2FA baseline (senha + Email OTP) e acesso baseline pleno**. Só **depois**, já dentro, o sistema
**sugere** (nudge, não obriga) reforçar: 2º email de backup + fator forte. **Nunca se barra o
acesso por não ter fator forte** — ele é pedido *just-in-time* na 1ª ação sensível (step-up)
ou sob risco (§9).

## 5. Multi-tenant + RBAC/ABAC — motor ReBAC (estilo Zanzibar)

- **PDP/PEP separados:** PDP = **Check API** do motor; **PEP = middleware/handler ASP.NET Core**
  em cada serviço — **policy-based authorization** (`IAuthorizationHandler` +
  `[Authorize(Policy=...)]`) ou middleware que extrai `sub`+tenant do token e chama
  `Check(obj, rel, subject)` no PDP.
  **Deny-by-default** (erro/timeout do PDP = nega), enforcement **server-side**, **todo
  endpoint mapeia 1 permissão**.

- **Token fino:** o JWT carrega `sub`/tenant/sessão/AAL — **sem** a lista de permissões
  (evita authz stale em token longo); a decisão é consultada no PDP e cacheada com TTL
  curto (in-memory/Redis) invalidado por evento de mudança de papel.

## 6. Sessão, multi-dispositivo e logout

- **Refresh rotativo com detecção de reuso** (reusou um refresh já rotacionado → revoga a
  **família** inteira). O refresh token é opaco (CSPRNG, **`RandomNumberGenerator`** — nunca
  `Random`), hasheado no store.

- **Botão "Sair" bem visível → kill IRREVERSÍVEL da sessão:** não basta apagar o cookie —
  o handler de logout do auth C# **revoga o refresh token (e a família), apaga o registro
  de sessão server-side, joga o `jti` na denylist (Redis/DB) até expirar e desassocia o
  push token do device**. Depois do logout, aquela sessão é irrecuperável: nem replay, nem
  refresh, nem "voltar o cookie" reativa. O middleware de validação de access token
  **consulta a denylist de `jti`** em toda request (cache curto).

## 7. Migração de auth legado — PRIORIDADE 0

Existe auth no padrão antigo → **portar pra este IAM é prioridade máxima** (segurança
inegociável; pode gastar o que precisar). Estratégia **strangler-fig** (casa com
schematize-node quando o legado é Node): dual-run, **re-hash preguiçoso** no login
(verifica no hash legado, regrava em **argon2id**), mapeia registros legados → modelo novo
(dedupe de emails, cunha IDs internos ULID/UUIDv7), **ativa o Email OTP always-on como 2º
fator baseline** (a conta migrada já entra em 2FA sem muro) e **incentiva enrolar fator forte**
(step-up para sensível), **revoga sessões legadas** e **nunca confia na authz legada**
(re-deriva as tuplas ReBAC). O auth migrado nasce já como **microserviço C#/.NET separado** (§1).

## 8. Rotina agressiva de testes (detalhe na schematize-pentest)

- **Abuso de fluxo:** bypass de 2FA, reset pulando 2FA, brute-force/rate-limit de OTP,
  replay de token, reuso de refresh, JWT `alg=none`/kid trocado, session fixation,
  adulteração de asserção SSO, IDOR na gestão de identificadores, bypass de step-up,
  mass-assignment de papel, **logout que não invalidou de verdade** (sessão recuperável).

## 9. Autenticação adaptativa por risco (robusta) + transversais

A resposta ao login **varia com o risco calculado** (não é fixa) — é o que torna a conta
difícil de tomar sem chatear o legítimo:

- **Log de sessões/tentativas:** cada tentativa e sessão gravam IP/ASN+reputação, device
  fingerprint, geo, UA, horário, resultado e **score de risco** — na view de sessões (§6) e em
  audit log imutável. É o insumo do score.

- **Score por tentativa:** IP suspeito/novo (Tor/proxy/ASN de abuso), device novo, geovelocidade
  impossível, velocity/brute, hit de honeypot. Baixo = fluxo normal; alto = escala.

- **Escalonamento por risco (2FA→3FA):** sob risco, exige um **fator a mais na ordem de força**
  — **senha → código por email → app OTP/chave**. Acertar senha+email não basta em contexto
  suspeito. Mesmo motor do step-up (§3), disparado pelo **contexto**, não só pela ação.

- **Negação deceptiva / tarpit (falso negativo sob risco):** em contexto suspeito, mesmo com
  **senha correta** o serviço pode responder **genérico `invalid_credentials` uma vez** enquanto
  **computa server-side que a credencial estava certa** e marca que a **próxima** tentativa
  correta **passa** (já com os fatores escalados). Seguro porque: **resposta e tempo IDÊNTICOS**
  ao erro real (sem oráculo — use comparação em tempo constante com
  `CryptographicOperations.FixedTimeEquals` e o mesmo path de resposta); estado "próxima passa"
  **curto e escopado** (conta+IP+device, TTL curto, expira sozinho, nunca vira lockout do
  legítimo); **soma-se** ao 3FA, não substitui; tudo logado.

- **Honeypot:** contas/campos/rotas isca; qualquer interação = sinal forte de hostil → score
  alto, tarpit/deceção, alerta. Nunca serve tráfego real.

- **Notifica o usuário:** login novo/suspeito, novo device, mudança de credencial → aviso nos
  canais verificados, com "não fui eu" (revoga + força reforço).

### Transversais (sempre)

- **Audit log imutável** de toda decisão authn/authz e mudança de credencial — alimenta a
  forense e os testes (liga com a observabilidade LGTM+ da casa; spans OpenTelemetry no
  fluxo de login/authz com `trace_id`).

## Roadmap de fases

- **F2** Multi-tenant + **ReBAC** (membership, papéis granulares, PDP/PEP em middleware ASP.NET Core,
  deny-default, token fino, audit).

## Checklist (entra na Definition of Done quando o projeto tem auth)

- [ ] Auth é **app separada** — microserviço C#/.NET `<projeto>_auth_cs` + front próprios em `auth.<domain>`, isolados — não monolith; **não** o ASP.NET Core Identity embutido no app de negócio.

- [ ] **ID interno imutável** (ULID/UUIDv7); email/telefone não são ID; múltiplos emails suportados.

- [ ] **2FA baseline por desenho** (senha + Email OTP = 2FA desde o cadastro); fator forte é **nudge + just-in-time (step-up)**, **NUNCA muro pré-login** (o middleware libera o baseline, exige AAL alto só por rota sensível); passkey no núcleo; email OTP always-on; Twilio; providers como interfaces C# via DI.

- [ ] **Risk engine adaptativo:** log de sessões/tentativas + score (IP/device/geo/velocity/honeypot); **2FA→3FA** sob risco; **negação deceptiva/tarpit** (falso negativo, resposta idêntica ao erro real em tempo constante via `CryptographicOperations.FixedTimeEquals`, "próxima passa" curta/escopada); notifica login suspeito.

- [ ] Invariante de troca de fator (Y≠X, maior AAL); recuperação ≥ login; SSO com recuperação local; senha em **argon2id** (Konscious)+HIBP.

- [ ] **Multi-tenant + RBAC/ABAC** (ReBAC OpenFGA/SpiceDB via client), deny-default, PDP=Check API / PEP=middleware/policy ASP.NET Core, enforcement server-side, token fino.

- [ ] Multi-dispositivo + view de remover; **sessão 7d/90d**; **logout irreversível** (revoga refresh+família, `jti` na denylist, não só cookie).

- [ ] JWKS público (assinatura só no auth); audit log de authn/authz; risk engine/rate-limit; migração de legado tratada como prioridade 0.

- [ ] **Efeito externo não sai de não-prd (§3.1):** `IEmailProvider` com **sink por default** fora de prd, **`GuardedEmailProvider` decorando no DI** (destinatário fora do domínio de teste ⇒ `ExternalRecipientBlockedException`), **cap por execução** (`MailRunQuota`/`Interlocked`), `MailOptions` com `ValidateOnStart` e ambiente **declarado** (nunca `IsProduction()` do host), chave sandbox, e **teste xUnit que espera a exceção**; nenhum `@gmail.com` em fixture/seed/persona.
