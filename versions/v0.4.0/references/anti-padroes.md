# Filosofia, Aplicação Universal e Anti-Padrões Vetados


> **PONTEIRO, não cópia.** A normativa deste tema é da base: **`schematize-engineering`** →
> `references/anti-padroes.md`. Leia lá primeiro; aqui fica **só o que muda em C#/.NET**.
>
> **Onde este arquivo divergir da base, a BASE MANDA** (`SKILL.md` §"Precedência e herança").
> Em 2026-08-21 os blocos idênticos à base foram **podados mecanicamente** (`tools/podar-clone.mjs`),
> que é por que a numeração dos itens **salta**: o número é o da base, e o item que não aparece aqui
> é porque **não muda nesta linguagem** — procure-o lá. Manter a cópia era manter a próxima deriva
> (foi assim que o `argon2id-only` da casa virou "ou PBKDF2" numa skill e o rol de 6 linguagens
> virou "só Go e Rust" em três).

## 37. Anti-Padrões Vetados — "Macaquices" que Terminam Rápido e Quebram em Produção

### Injeção e execução

4. **SQL por concatenação de string** com input externo — inclui string interpolation (`$"... {input} ..."`) montando SQL.
   → Prepared statements / query parametrizada, sempre (§10). Em C#, EF Core ou Dapper com parâmetros; nunca interpolação de string em SQL.

### Auth e autorização

11. **`Math.random()`, `System.Random` (ou rand não-cripto) pra token, id de sessão, código de reset, nonce.**
    → CSPRNG: `crypto.randomBytes`, `crypto/rand`, `RandomNumberGenerator` (§14).

### CORS, headers e superfície

12. **`Access-Control-Allow-Origin: *` em rota autenticada** (pior ainda com `allow-credentials`).
    → Allowlist explícita de origens (hardening; ver a `schematize-qa`, `references/categorias.md` §§5 e 10).

13. **Endpoint de debug/admin/management sem auth, ou bind em `0.0.0.0`** expondo porta interna.
    → Bind restrito, auth obrigatória, `/debug` e `/actuator` retornam 404 externamente (ver a `schematize-qa`, `references/categorias.md` §§5 e 10).

### Erros, tipos e qualidade

15. **Catch que engole erro** — `catch {}`, `catch (Exception) {}` engolido, `except: pass`, `_ = err`, `.catch(() => {})`; `async void` (a exceção some) ou `.Result`/`.Wait()` (sync-over-async, trava thread e mascara a falha).
    → Tratar, logar com contexto e `trace_id`, propagar ou degradar de forma consciente. Em C#, `await` de verdade — `async void` só em event handler; nunca `.Result`/`.Wait()`.

16. **`// @ts-ignore`, `any`, `interface{}`, `#nullable disable` pra calar warning, `!` (null-forgiving) indiscriminado, `unwrap()`/`panic`/`!` pra calar o compilador/linter.**
    → Tipar de verdade, tratar o caso de erro. Inline-ignore de regra **de segurança** (`nolint`, `eslint-disable security/*`, `# nosec`) é VETADO sem ADR.

### Testes e cobertura

19. **Baixar o threshold de cobertura ou editar o gate** pra o número fechar.
    → Cobertura é contrato (ver a `schematize-qa`). Sobe escrevendo teste, não mexendo na régua.

### Operação e entrega

31. **Novo serviço backend fora do rol sancionado (Go/Rust/Elixir/C#/Zig/Ruby), ou sem ADR de fit da linguagem; qualquer serviço backend novo em Node ou código novo em PHP.**
    → Backend novo **só no rol sancionado** — C# incluído (ecossistema .NET/enterprise, ASP.NET Core/EF Core) — e com **ADR de fit** justificando a escolha pro problema. Node backend e PHP são **legado/saída**: não recebem serviço novo e migram (§3); Node backend legado segue a regra dos 30% (§3.1). **Frontend em Node** (Next.js/Astro) segue **100% permitido**. Rol sancionado e critérios de fit em `schematize-engineering/references/linguagens.md`.

36. **Operar o servidor por fora do `<projeto>_ops`** — `ssh` + comando ad-hoc, editar arquivo no servidor, `docker`/`kubectl`/`systemctl` na mão, script solto.
    → **100%** de install/update/config/correção passa por comando do ops. Não tem comando? **cria no ops** (`schematize-engineering` -> `references/ops.md` §2).

37. **Instalar/subir o sistema em série** ("um serviço de cada vez", 20 min).
    → Instalação **paralela por padrão** = `nproc` (`schematize-engineering` -> `references/ops.md` §3).

### IAM (identidade e autorização)

43. **Auth apensado ao escopo principal como monolith** (login/2FA/authz dentro do serviço principal, num `internal/auth`, sem serviço/front próprios).
    → Auth é **app separada** em `auth.<domain>`: **microserviço C#** `<projeto>_auth_cs` + `<projeto>_authfront`, isolados; apps delegam por OIDC/PKCE e validam por JWKS público (`references/iam.md` §1).

44. **Email/telefone como ID de usuário** (`user_id = email`, FK por email, login que assume 1 email), ou 2FA/recuperação com 1 fator só (reset por 1 email que pula o 2FA).
    → **ID interno imutável** (ULID/UUIDv7) como `sub`; email/telefone são identificadores N e verificáveis; **≥2 fatores sempre** (passkey/WebAuthn no núcleo); **recuperação ≥ força do login**; senha em **argon2id**+HIBP (`references/iam.md` §2–§4).

45. **Autorização hand-rolled / no cliente / permissão embutida em token longo** — `if role == "admin"` espalhado pelos handlers, checagem só no front, sem multi-tenant, papéis não-granulares.
    → **RBAC/ABAC granular por motor ReBAC** (OpenFGA/SpiceDB via client), **deny-default**, PDP=Check API / PEP=**middleware ASP.NET Core**, **enforcement server-side**, token fino, decisão auditada (`references/iam.md` §5).

46. **Logout que só apaga o cookie** (sessão recuperável por refresh/replay), ou sessão curta que chuta o usuário toda hora sem refresh silencioso.
    → **Logout irreversível** (revoga refresh+família, apaga sessão server-side, `jti` na denylist consultada em toda request); **sessão 7d/90d** com refresh rotativo silencioso e multi-dispositivo (`references/iam.md` §6).

### Efeitos externos (e-mail, SMS, push, webhook, cobrança)

47. **Mandar de verdade fora de produção** — `ResendEmailProvider`/SMTP real registrado em dev/hml, provider escolhido por `IHostEnvironment.IsProduction()` (que dá `true` sem `ASPNETCORE_ENVIRONMENT` — **fail-OPEN**), `@gmail.com` (ou o seu e-mail) em fixture/seed/persona, laço de teste criando N contas com **Email OTP always-on** e nenhum contador no caminho.
    → **Sink por default fora de prd** + **guard como decorator no DI** (`Decorate<IEmailProvider, GuardedEmailProvider>()`, `ExternalRecipientBlockedException`), **ambiente declarado** em `IOptions<MailOptions>` com `ValidateOnStart` (ausente ⇒ não-prd), **cap por execução** com `Interlocked` em singleton, e endereço sintético só em `test.<domain>` com **null MX** (`references/iam.md` §3.1; normativa em `schematize-engineering/references/efeitos-externos.md`). Bounce/complaint em massa **queima IP e domínio** e derruba o OTP de login de **produção** — semanas de warm-up, utilidade zero.
