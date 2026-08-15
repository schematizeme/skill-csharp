# Segurança, Auth, Multi-tenancy, LGPD e Frontend

> Parte da skill **schematize-csharp**. As referências cruzadas (§N) apontam para seções do corpo completo — todas presentes no conjunto de references desta skill.

## Índice
- 13. Segurança
- 14. Autenticação e Autorização
- 15. Multi-tenancy
- 32. LGPD e Dados Pessoais
- 38. Frontend / NextJS / SPA — Regras Específicas

---

## 13. Segurança

**MUST — pipeline (fail on high/critical)**
- Dependabot ou Renovate
- SAST (Semgrep / CodeQL)
- SCA (dependências + licenças)
- Container scan (Trivy / Grype)
- Secret scan (Gitleaks)
- Commit signing (GPG / SSH / Sigstore)
- SBOM no build (CycloneDX ou SPDX)

**MUST — runtime**
- Containers não-root, read-only filesystem.
- Imagem base `mcr.microsoft.com/dotnet/aspnet` chiseled/distroless (ou chainguard) quando viável.
- Multi-stage build com `dotnet publish` (restore/build/publish num estágio, runtime enxuto no final).
- Healthcheck na imagem.

**MUST — segredos**
- Nunca em código, nunca em env file commitado, **nunca em `appsettings.json` commitado nem no bundle publicado**.
- Storage: **user-secrets** (dev), Azure Key Vault / AWS/GCP Secret Manager, Vault, sealed-secrets, ou env injetada em runtime.
- **Data Protection API** (`IDataProtectionProvider`) para proteger payloads sensíveis em repouso/trânsito interno (chaves persistidas fora do container, protegidas por Key Vault/HSM em produção).
- Rotação documentada.

**MUST — acesso a dados**
- SQL **sempre parametrizado**: **EF Core** (LINQ / `FromSqlInterpolated` com parâmetros) ou **Dapper** com parâmetros nomeados (`@id`). String interpolation/concatenação montando query (`$"... WHERE id = {id}"` cru, `FromSqlRaw` com string montada) é VETADO (§37).
- Nunca expor `DbContext`/entidade diretamente na borda HTTP — DTO/projeção explícita (evita over-posting / mass assignment).

**MUST — transporte e CSRF**
- **HTTPS obrigatório** (`UseHttpsRedirection`) + **HSTS** (`UseHsts`) em produção.
- **Antiforgery** (`AddAntiforgery` + validação) em toda mutação servida a browser com cookie de sessão (CSRF).
- **CORS explícito** (`AddCors` com origens declaradas) — nunca `AllowAnyOrigin` em rota autenticada.

**Licenças permitidas:** MIT, Apache 2.0, BSD, MPL 2.0, ISC.
**Bloqueadas sem ADR:** GPL/AGPL, SSPL, proprietárias.

### 13.4 Segredos no Cliente / Frontend / NextJS

> O backend .NET **expõe API** e não serve UI própria: o frontend delega ao **schematize-web** (Next/Astro). As regras abaixo valem para qualquer cliente que consome essa API.

**VETADO — sem exceção, sem ADR**

- **Qualquer segredo no bundle que vai pro browser.** API key privada, secret de JWT, senha de banco, service-role key (Supabase/Firebase admin), token de provedor de pagamento, chave de terceiro — **nada** disso entra em código que o cliente baixa. O navegador não guarda segredo. Ponto.
- Prefixar segredo com `NEXT_PUBLIC_` (ou equivalente `VITE_`, `REACT_APP_`, `PUBLIC_`). Esse prefixo **expõe a variável publicamente por definição** — usar só para valor que pode estar num outdoor.
- Chamar API de terceiro com chave secreta **direto do browser**. Toda chamada que usa segredo passa por um **BFF / route handler / server action** server-side (§38).
- Guardar token de sessão em `localStorage`/`sessionStorage`. Sessão vai em **cookie `HttpOnly` + `Secure` + `SameSite`** (§38, §22.3 hardening).
- Confiar em validação de auth/role feita no client (`if (user.isAdmin)` no React) como controle de acesso. Isso é UX, não segurança — a decisão é **sempre server-side** (§15, §37).

> "Coloca a senha no NextJS pra funcionar" não é uma solução, é um vazamento agendado. Se o cliente pode ver, o atacante já viu.

---

---

## 14. Autenticação e Autorização

- OAuth2 / OIDC.
- JWT assinado (RS256 ou EdDSA; **nunca HS256** em fluxo público).
- Validação **completa** do JWT em toda request via **`JwtBearer` + `TokenValidationParameters`** com tudo ligado: `ValidateIssuerSigningKey`, `ValidateLifetime` (`exp`/`nbf`), `ValidateAudience` (`aud`), `ValidateIssuer` (`iss`) e `ValidAlgorithms` como allowlist explícita. Decodar o payload (`JwtSecurityTokenHandler.ReadJwtToken`) e confiar sem validar é VETADO (§37).
- Refresh token rotativo com detecção de reuso.
- RBAC com permissões granulares (policies/`AuthorizationHandler`); ABAC quando necessário.
- **ASP.NET Core Identity não é o IAM da casa** — o IAM é uma app separada (ver `iam.md`, §14 do corpo). Identity pode servir de biblioteca de primitivas, nunca como a fronteira de autenticação do sistema.
- Hash de senha: **argon2id** (via `Konscious.Security.Cryptography`) ou **PBKDF2** (`Rfc2898DeriveBytes` com iterações altas e sal por usuário). MD5/SHA1/sem-salt/plaintext são VETADOS (§37).
- Tokens, ids de sessão e códigos de reset gerados por **CSPRNG** (`System.Security.Cryptography.RandomNumberGenerator`) — **nunca `System.Random`/`Random`** (§37).

---

---

## 15. Multi-tenancy

**MUST — quando aplicável (SaaS, plataformas)**
- Isolamento de tenant explícito em todas as camadas.
- `tenant_id` propagado em contexto, logs e traces.
- Autorização validada **server-side** em toda operação. **Nunca** confiar em `tenant_id` (ou role, ou user_id) enviado pelo cliente sem verificação contra o token. Derivar sempre do token, server-side.
- Queries com `tenant_id` no `WHERE`, sempre. Considerar Row Level Security no Postgres.

**SHOULD**
- Testes de cross-tenant leak em CI.
- Métricas e logs particionáveis por tenant.

---

---

## 32. LGPD e Dados Pessoais

**MUST**
- Classificação de dados: público, interno, confidencial, pessoal, pessoal sensível.
- PII nunca em logs (ver §16.1).
- Política de retenção documentada por tipo de dado.
- Processo para exercício de direitos (acesso, correção, eliminação, portabilidade).
- Criptografia em trânsito (TLS 1.2+) e em repouso para dados pessoais.
- DPIA para tratamentos de alto risco.
- PII **nunca** em query string / URL (acaba em log, histórico, referer) — §37.

---

---

## 38. Frontend / NextJS / SPA — Regras Específicas

> O backend .NET **expõe API**; a UI é responsabilidade do **schematize-web** (Next/Astro, TypeScript strict). As regras de fronteira client/server abaixo governam esse frontend e o contrato que a API .NET oferece a ele.

**MUST**
- **Fronteira clara client/server.** Tudo que toca segredo, banco, ou terceiro com credencial roda **server-side** (route handler, server action, BFF, server component que não serializa segredo pra props).
- Sessão em **cookie `HttpOnly` + `Secure` + `SameSite=Lax|Strict`**. Token de auth **nunca** em `localStorage`/`sessionStorage` (XSS lê tudo lá).
- Variáveis públicas (`NEXT_PUBLIC_*` etc.) contêm **apenas** dado não-sensível (URL de API pública, id de analytics público). Tratar esse prefixo como "vai pro outdoor".
- Validação de input no client é **UX**; a validação que importa é a do servidor (§12). Autorização idem é server-side (§15).
- CSP, `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy` e `frame-ancestors` configurados (§22.3 hardening).
- Chamada a API de terceiro com chave secreta passa por proxy server-side. O browser nunca segura a chave.

**VETADO**
- Service-role key / admin SDK (Supabase, Firebase, etc.) no código do client.
- `dangerouslySetInnerHTML` com conteúdo não sanitizado (XSS).
- Confiar em `redirect`/`next` param sem allowlist (open redirect — §22.3).

> "Bota a senha no NextJS" não existe como solução. Existe como CVE. O front pede ao servidor; o servidor guarda o segredo.

---

---
