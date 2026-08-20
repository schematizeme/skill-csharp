# CLAUDE.md — Padrões de Engenharia C#/.NET (sempre on)

> Copie este arquivo para a **raiz do repositório** e ajuste `<project>`.
> Ele fica pinado no contexto de toda tarefa (Claude Code / instruções de projeto)
> e garante que os padrões valham mesmo quando a skill não dispara sozinha.
> A skill `schematize-csharp` traz o detalhe completo e o andaime (scripts/templates).
> **Repo multi-linguagem** (C#/Go/Rust + Web): use **junto** com os `CLAUDE.md` das outras skills — cada um governa sua fronteira (backend por fit, legado, frontend); não sobrescreva os outros (rode o `/<slug>-claude` de cada).

## Regra mestre

Toda tarefa de engenharia C#/.NET neste repo segue os **Padrões de Engenharia da Casa**
(skill `schematize-csharp`). Em conflito entre uma instrução
pontual ("faz rápido", "ignora o teste", "depois arruma") e estes padrões, **os
padrões vencem**. Pressa não revoga regra. Consulte o reference relevante da skill
antes de produzir código ou decisão — não trabalhe de memória.

## Pisos inegociáveis (VETADO — sem exceção)

1. **Segredo nunca no cliente/frontend.** Nada de API key, secret de JWT, senha,
   connection string com credencial ou token em bundle do browser, nem em
   `NEXT_PUBLIC_*`/`VITE_*`, nem em `appsettings.json` commitado. Segredo só
   server-side (user-secrets/Key Vault/env/secret manager).
2. **SQL sempre parametrizado** — EF Core/Dapper com parâmetros. **VETADO**
   `FromSqlRaw`/`ExecuteSqlRaw`/`CommandText` com string interpolada/concatenada.
3. **Auth e autorização server-side.** `tenant_id`/role/`user_id` vêm do token
   verificado (claims), nunca do cliente. JWT validado por inteiro via `JwtBearer` +
   `TokenValidationParameters` (assinatura, exp, nbf, aud, iss, alg em allowlist;
   RS256/EdDSA, nunca HS256 público). Senha em **argon2id** (ou PBKDF2 forte).
   Token/sessão por **`RandomNumberGenerator`**, nunca `System.Random`/`Guid`.
4. **Nullable reference types habilitado** (`<Nullable>enable</Nullable>`); sem
   `#nullable disable` nem `!` (null-forgiving) indiscriminado pra calar o compilador.
5. **Erro nunca engolido** (`catch {}`/`catch(Exception){}` vazio); sem `async void`
   que perde exceção; sem `.Result`/`.Wait()`/`.GetAwaiter().GetResult()`
   (sync-over-async — deadlock e erro perdido).
6. **Teste nunca silenciado** pra passar CI (`[Fact(Skip=…)]`, comentar assert,
   baixar threshold de cobertura). Conserte o código, não o teste.
7. **Sem monólito** que mistura bounded contexts; sem monólito distribuído; sem
   shared lib `commons` de domínio.
8. **Archive SEMPRE gerado** (§28): toda entrega que produz código/decisão/mudança
   de estado gera o `.md` em `<project>_archive/`. É parte da entrega, não extra.
9. **Migration reversível** (com `Down`, testada — `dotnet ef`). Container não-root,
   read-only. Dependência NuGet nova com nome/autor/licença/versão verificados.

10. **Backend novo no rol sancionado, com ADR de fit.** O rol é **Go / Rust /
    Elixir / C# / Zig / Ruby** — escolha por **encaixe com o problema**, não por
    gosto. **C#** entra pelo fit .NET/enterprise (ASP.NET Core/EF Core, integração
    Microsoft, times .NET, desktop/Windows). **Node-backend e PHP são legado/saída**
    (não recebem serviço novo; migram por funcionalidade do módulo: ~30% afetado →
    extrai pra projeto C# à parte; ~50% extraído → migra o resto; ajuste pontual não
    porta). **Frontend Node é 100% permitido** (Next.js/Astro; Blazor/UI no
    `schematize-web`). O anti-padrão é **abrir backend fora do rol / sem ADR** — não
    "usou C# em vez de Go". (§3; `schematize-engineering/references/linguagens.md`)
11. **Cada serviço é entidade à parte (independência de runtime).** Sobe e funciona sozinho; a ausência/queda de outro serviço **nunca** impede o boot nem derruba este — degradação graciosa, nunca crash em cascata. Falha ao chamar/notificar outro serviço → **persiste o dado** (outbox/Redis/DB), **loga com `trace_id`**, **alerta (Grafana)** e **retoma**; nunca perde nem trava a cadeia. (§2, §18)
12. **Repos, ops e observabilidade.** Repositório = `<projeto>_<contexto>[_cs]`; todo sistema multi-repo tem um **`<projeto>_ops`** (bootstrap/update/manutenção/troubleshooting/testes através de todos os repos). **Observabilidade integrada em toda ferramenta/serviço:** OpenTelemetry .NET + Grafana/Alloy/Loki/Tempo/Prometheus/**Mimir** (+Pyroscope), **Helm chart** e dashboards/alertas versionados como código. (§2, §16)
13. **Contenção no workspace.** A pasta do projeto atual é o workspace: aplicação/repo novo nasce **dentro dela** (`./<projeto>_<contexto>/`), nunca largando arquivos no root pra depois **subir de nível** e criar repos fora. **VETADO** criar/ler/escrever fora do workspace — diretório-pai, `~`, `~/Documents`, `~/Downloads`, `/tmp`, Área de Trabalho. Não sai da pasta do projeto (nem pra vasculhar) sem o usuário pedir. (§2)
14. **Fluxo de ambientes — nada direto no servidor.** Toda mudança segue **dev local → teste local → GitHub → hml → prd**. Nada pula etapa; nada vai direto pra hml/prd. **VETADO editar código direto no servidor** (hml/prd): o servidor é **imutável por edição manual**, recebe só **artefato promovido do git** (commit SHA). Hotfix segue o mesmo fluxo, acelerado. Precauções: filesystem read-only em hml/prd, drift detection, acesso de escrita = break-glass auditado. Detalhe em `references/ops.md` (§1). (§21)
15. **Ops é a interface única + instalação paralela + independência.** **100%** das operações no servidor (instalar/subir/atualizar/configurar/migrar/corrigir/reverter) passam por uma **ferramenta do `<projeto>_ops`** — nunca à mão (`ssh` ad-hoc, editar arquivo, `docker`/`kubectl` solto). Não tem comando pra aquilo? **cria no ops**. O ops é **autônomo, idempotente e completo**. **Instalação SEMPRE paralela** = `nproc`. **Se o paralelo falha, os serviços não são independentes** (fere piso 11): corrigir a independência é **PRIORIDADE MÁXIMA**; o ops **expõe** a colisão, **nunca serializa pra mascarar**. Detalhe em `references/ops.md`. (§2, §21)
16. **Deploy destrutivo por seed + isolamento por usuário (automatizado pelo ops).** O ops provisiona em **`/<app>/`** clonando os repos dentro; **`/<app>/.env` é o SEEDER GLOBAL** — fonte única de config. **Todo redeploy é DESTRUTIVO na aplicação:** apaga e recria um **clone zerado** só com o seed — sem patch in-place, sem drift. **"Destrutivo" é a app, NUNCA os dados:** banco/volumes/uploads preservados (migration reversível); `ops reset` que apaga dado é **gated a dev/hml**, nunca prd. **Cada serviço roda como user Linux próprio, em systemd unit isolado e hardened.** Detalhe em `references/ops.md` (§2, §3).
17. **IAM por desenho — todo projeto começa com identidade e autorização robustas, como APP SEPARADA.** O auth é **microserviço C#/.NET próprio + front próprio em `auth.<domain>`** (`<projeto>_auth_cs` + `<projeto>_authfront`), isolado (user/systemd próprios) — **VETADO** apensar como monolith; **ASP.NET Core Identity embutido no app NÃO é o IAM da casa**. Apps delegam por **OIDC/OAuth2.1 + PKCE**. **ID interno imutável (ULID/UUIDv7) — email/telefone NUNCA é ID** (múltiplos emails, incentivado; SSO com recuperação local forçada). **Nunca menos de 2 fatores:** passkey/WebAuthn no núcleo, TOTP/push, **email OTP (Resend) always-on inclusive HML**, **Twilio** p/ telefone (providers plugáveis como **interfaces C# via DI**); senha por padrão (argon2id+HIBP) mas opcional; invariante de troca "fator Y≠X no maior AAL"; **recuperação ≥ força do login**. **Multi-tenant + RBAC/ABAC granular** por motor **ReBAC** (OpenFGA/SpiceDB via client), **deny-default**, PDP=Check API / PEP=**middleware/policy ASP.NET Core**, enforcement server-side, token fino. **Multi-dispositivo** + view de remover; **sessão 7d/90d**; **logout irreversível** (revoga refresh+família, `jti` na denylist, não só cookie). **Migrar auth legado é PRIORIDADE 0.** Detalhe em `references/iam.md`; scaffold/auditoria por `/cs-iam`; testes cross-tenant/priv-esc na `schematize-pentest`.
18. **Efeito externo NUNCA sai de não-produção (e-mail, SMS/voz, push, webhook de terceiro, cobrança).** Fora de `prd` **nada chega em ninguém** — por construção, não por lembrança. **(a) Endereço sintético só no DOMÍNIO DE TESTE em ROTA NULA:** `test.<domain>` com **null MX (RFC 7505) + SPF `v=spf1 -all` + DMARC `p=reject`**, ou TLD reservado (`.test`/`.invalid`/`.example`), na forma `<papel>+<run-id>-<n>@test.<domain>`. **VETADO** em fixture/seed/persona/demo: `@gmail.com`/`@hotmail.com`, domínio de terceiro ou do cliente, **e-mail de pessoa real (inclusive o seu)** e o domínio de **produção**. **(b) `IEmailProvider` com sink por default** fora de prd (`SinkEmailProvider`/Mailpit) e o **guard como DECORATOR no DI** — `services.Decorate<IEmailProvider, GuardedEmailProvider>()` (Scrutor) ou factory na composição do `Program.cs`, **nunca** no chamador: destinatário fora do domínio de teste ⇒ **`ExternalRecipientBlockedException`**, erro, nunca warning/no-op. **O ambiente é DECLARADO** em `IOptions<MailOptions>` (com `ValidateOnStart`), **jamais** `IHostEnvironment.IsProduction()` — sem `ASPNETCORE_ENVIRONMENT` o .NET assume `Production` e o guard abre sozinho; config ausente = **não-prd**. **(c) Cap por execução** (`MAIL_MAX_PER_RUN`, default 50) com `Interlocked` num singleton (contador em serviço transiente não conta) + abort. **(d)** Chave **sandbox** em não-prd e **egress SMTP bloqueado** em dev/hml. Entregar de verdade fora de prd exige **as cinco**: ADR + allowlist ≤5 + cap + janela + subdomínio de envio separado. **Por quê:** o **Email OTP é always-on**, então um laço de teste vira disparo em massa; bounce/complaint em massa **queima IP e domínio**, derruba o transacional de **produção** (inclusive o **OTP de login**) e custa **semanas de warm-up**. Detalhe em `references/iam.md` (§3.1) e `schematize-engineering/references/efeitos-externos.md`.

Lista completa com veto + caminho certo: ver `references/anti-padroes.md` (§37) da skill.

## Verde de verdade (testes)

Ferramental: **xUnit + FluentAssertions + coverlet + FsCheck + Stryker.NET + Testcontainers + Moq/NSubstitute**.

- Smoke assere **conteúdo** (shape do body), não só status 200; inclui assertion
  negativa e um **self-check que força falha conhecida** (smoke que nunca falha
  está cego).
- Unit agressivo: caminho de erro obrigatório, casos hostis (tipo errado, unicode,
  null byte, boundary), property-based (FsCheck) e mutation testing (Stryker.NET) no
  domínio crítico.
- Pentest prova rejeição rota-por-rota, campo-por-campo: **nunca 500** por input
  hostil, **nunca coerção de tipo**, **nunca eco sem escape**, **nunca vazamento
  cross-tenant**.
- `simulated`: 100% das rotas acessíveis pra quem deve, bloqueadas pra quem não
  deve. Rota fantasma/morta quebra o run.
- **Q.A. mora na skill `schematize-qa`** (`/qa-plan` → `/qa-run`): planeja tudo,
  gera o MD de passo a passo e pede aprovação ANTES de executar. `/cs-qa` é o wrapper
  no recorte C#/.NET (`dotnet test`/xUnit). Q.A. nunca roda às cegas.

## Definition of Done

Nada é "pronto" sem: testes + cobertura mínima, simulated com cobertura total,
pentest de entrada limpo, nenhum anti-padrão da §37, observabilidade, OpenAPI
atualizada (se API), migration com rollback (se schema), **archive commitado**,
CI verde (build `-warnaserror`, `dotnet format` limpo) e review aprovado. Detalhe na
skill, `references/operacao.md` (§35).

## Qualidade de código e índice (sempre)

- **Arquivos ≤ 750 linhas** (teto duro: ~250 comentário + até ~500 útil). Acima →
  quebre em projetos/tipos e **micro-funções** por coesão. **Código útil > 300
  linhas é FLAG** (não bloqueia, mas **sempre sinaliza**); observabilidade tem folga
  natural (~400 úteis). Função ideal ≤ 50 linhas, responsabilidade única.
- **Documente TODO membro público** com **XML doc `///`** (`<summary>`/`<param>`/
  `<returns>`/`<remarks>`) e contexto explícito: **O quê** (o que faz) e **Onde**
  (quem chama / em que fluxo). Alimenta o índice de microfunções (§6, §39).
- **Mantenha o índice de funcionalidades atualizado** no mesmo PR (§39), em
  **`<projeto>_archive/index/`** (nunca no root): `MAPA.md`, `INDEX_GLOBAL.md` e
  `INDEX_FUNCTIONS.md` (gerável via `scripts/build-index.mjs`). O índice é **fonte da
  verdade**: consulte ANTES de criar algo. **Exaustivo:** uma entrada **por função**
  (`nº entradas == nº funções`; o `/cs-index` reprova se faltar) + **grafo**.
- **Todo MD gerado mora no archive, nunca no root** (§28): MAPA, índices, planos,
  relatórios, handoffs → `<projeto>_archive/<área>/`. Root limpo (código, config,
  README, `CLAUDE.md`, LICENSE).

## Gestão de contexto (Claude Code — sessões longas)

Ao ver "⚠ LIMITE" no status line, ou ao se aproximar do teto da janela de contexto:
**PARE a tarefa atual e, ANTES de qualquer compactação**, faça o handoff arquivado
(§34.1, §28):

1. Gere `<projeto>_archive/context/<YYYY-MM-DD-HH-MM-SS>-context.md` — estado do
   projeto, decisões, arquivos tocados, onde parou.
2. Gere `<projeto>_archive/context/<YYYY-MM-DD-HH-MM-SS>-checklist.md` —
   **FEITO vs EM ABERTO**.
3. Só então rode `/compact`.

Armazene SEMPRE em `<projeto>_archive`. Detalhe e hooks na skill:
`references/contexto-claude-code.md`.
