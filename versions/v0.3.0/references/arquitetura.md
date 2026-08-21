# Arquitetura, Camadas, Repositórios e Linguagens


> **PONTEIRO, não cópia.** A normativa deste tema é da base: **`schematize-engineering`** →
> `references/arquitetura.md`. Leia lá primeiro; aqui fica **só o que muda em C#/.NET**.
>
> **Onde este arquivo divergir da base, a BASE MANDA** (`SKILL.md` §"Precedência e herança").
> Em 2026-08-21 os blocos idênticos à base foram **podados mecanicamente** (`tools/podar-clone.mjs`),
> que é por que a numeração dos itens **salta**: o número é o da base, e o item que não aparece aqui
> é porque **não muda nesta linguagem** — procure-o lá. Manter a cópia era manter a próxima deriva
> (foi assim que o `argon2id-only` da casa virou "ou PBKDF2" numa skill e o rol de 6 linguagens
> virou "só Go e Rust" em três).

## 2. Estrutura de Repositórios

- **Nome do repositório:** `<projeto>_<contexto>[_<lang>]` em snake_case minúsculo. `<projeto>` = slug do produto/organização; `<contexto>` = a aplicação/bounded context daquele repo (`api`, `worker`, `front`, `backoffice`, `gateway`…); `_<lang>` é sufixo **opcional** pra desambiguar linguagem (`_cs` C#/.NET, `_go` Go, `_rs` Rust, `_ts` TypeScript). Como um repo = um contexto, o nome espelha isso. Ex.: `loja_api_cs`, `loja_front`, `loja_worker_cs`.

- **Independência de runtime (cada serviço é entidade à parte):** todo serviço **sobe e opera sozinho**. A indisponibilidade de qualquer outro serviço **nunca** impede o boot nem derruba este — depender de outro serviço para *iniciar/funcionar* é VETADO (nada de "o `ledger` não sobe se o `core` estiver fora"). Dependente ausente vira **degradação graciosa** (fallback, resposta parcial, enfileira e segue), nunca crash em cascata. Como não perder o dado quando a chamada falha: `schematize-engineering` -> `references/dados-eventos.md` (§18).

## 3. Linguagens

A casa **não tem "a linguagem única"** — tem um **rol sancionado** e um **guia de fit**. A engenharia (pisos: segurança, testes, archive, DoD, IAM, ops, observabilidade) é **agnóstica** e vale igual em todas. A linguagem muda o **como**, não o **o quê**. Esta skill é o **recorte C# (.NET)** do rol. Canônico agnóstico: `schematize-engineering/references/linguagens.md`.

**Backend novo — escolha uma do rol, com ADR (§27) que justifique o fit:**

| Linguagem | Skill irmã | Sufixo de repo | Encaixe (fit) |
|---|---|---|---|
| **Go** | `schematize-go` | `_go` | serviços de rede/API concorrentes, CLIs, tooling de infra. **Default pragmático.** |
| **Rust** | `schematize-rust` | `_rs` | correção/segurança de memória crítica, performance sem GC, componentes sensíveis. **Default quando errar é caro.** |
| **Elixir** | `schematize-elixir` | `_ex` | realtime/alta concorrência tolerante a falha (BEAM/OTP), messaging/pub-sub. |
| **C# (.NET)** | **`schematize-csharp`** | **`_cs`** | **ecossistema .NET/enterprise, integração Microsoft, times .NET, cargas ASP.NET Core/EF Core, desktop/Windows quando aplicável.** |
| **Zig** | `schematize-zig` | `_zig` | baixo nível, performance máxima, interop com C, artefatos pequenos. |
| **Ruby** | `schematize-ruby` | `_rb` | prototipagem rápida, scripts/automação, DX de produto (Rails); manutenção de legado Ruby. |

**Quando C# é a escolha certa (o fit vira ADR):** a organização/time já é **.NET**; há **integração forte com o ecossistema Microsoft** (Azure, AD/Entra, SQL Server, Office/Graph); a carga é típica de **ASP.NET Core** (APIs, workers, gRPC) com **EF Core**; existe base/domínio **desktop ou Windows** a servir; ou o SDK maduro do fornecedor é primariamente .NET. Em empate técnico com Go, registre o porquê no ADR (o default pragmático da casa é Go).

**Frontend — Node é 100% permitido (e só frontend).** **Next.js** é a stack principal; **Astro e outros frameworks consolidados** são permitidos. O server-side do próprio front (route handlers, server actions, BFF) faz parte do frontend e é governado pelo §13.4 e §38 (segredo só server-side). Blazor/MAUI e qualquer UI ficam com o **`schematize-web`**; **aqui é API/backend/serviços** em C#. Isso vale só para frontend — **não** reabre Node como serviço backend.

- Versão exata em uso fica no Anexo A / `references/stack-versoes.md` (alvo: **.NET LTS atual — .NET 10**; verificado em 2026-08-21).

- **Todo serviço backend novo nasce no rol sancionado, com ADR de fit.** C# entra pelo encaixe acima — não por gosto, e não como "porque já sei C#" sem o fit.

- **Nova linguagem fora do rol** exige **ADR de exceção** aprovado.

- Frameworks são bem-vindos; abstrações mágicas não. Critério: consigo entender o stack trace? (ASP.NET Core e EF Core passam; "mágica" que esconde a query/o middleware, não.)

### 3.1 Fora do rol — Node-backend e PHP (legado/saída, não reabrem)

- **Node como serviço backend** e **PHP** **não recebem serviço novo.** O que existe é **legado**: fica como está até ser tocado e **migra para uma linguagem do rol** (aqui, quando o fit é .NET, para **C#**) medido **por funcionalidade do módulo**, não por linha.

- **Modelo da métrica.** Um módulo tem N funcionalidades. O quanto uma mudança "pesa" é a fração alterada/criada sobre o total. Ex.: módulo com 10 funcionalidades — refatorar 3 (ou criar ~4 novas) ≈ 30%.

- **Gatilho de extração (~30%):** ao atingir ~30% das funcionalidades do módulo, **não cresça o legado** — extraia para um **serviço/projeto C# à parte** (novo `.csproj`/bounded context) e incorpore o comportamento na nova base.

- **Virada dos 50%:** quando ~50% já estiver extraído/substituído, **migra-se o resto de uma vez** — encerra o módulo legado.

- **Ajuste pontual não porta.** Mudança pequena (abaixo do gatilho) é feita no próprio legado.

- Toda migração registra ADR (§27) e segue o DDD híbrido/coexistência (§4.X, §36): flag de coexistência, sem big-bang. Detalhe da saída do Node em `schematize-node`.

> O anti-padrão de linguagem **não** é "usou C# em vez de Go" — é **abrir serviço backend novo fora do rol sancionado, ou dentro dele sem ADR de fit**. O ganho marginal de tooling não reabre Node/PHP como backend.

## 4. Arquitetura

### 4.X DDD híbrido durante transição

- Manter teste de cobertura por camada (ver a `schematize-qa`) durante a transição — domain começa com 0%, sobe a cada PR.

- Em C#, a **própria árvore de `ProjectReference`** é o guard: o `.csproj` de `Domain` não referencia framework/ORM nem os outros projetos; violar isso não compila. Onde o legado ainda é um projeto único, use um **guard test** (ex.: teste de arquitetura com `NetArchTest`/análise de `using`) que **rejeita dependências proibidas** (mesmo com whitelist de exceções legadas):
  - `Domain` não usa `Microsoft.AspNetCore.*`, `Microsoft.EntityFrameworkCore`, `System.Data.*`, nem os namespaces de `Infrastructure`/`Application`/`Api`.
  - `Application` não depende de `Api` (interface).

## 5. Estrutura de Pastas

### C# (.NET) — projetos por camada dentro da solution

Uma **solution** (`.sln`) por bounded context; **um projeto (`.csproj`) por camada** (referência de projeto força a direção de dependência — o compilador vira o guard de camada). `Directory.Build.props` + `Directory.Packages.props` na raiz.

```
src/
├── <Ctx>.Domain/            # entities, value-objects, domain services/events, interfaces de repositório
├── <Ctx>.Application/        # use-cases, DTOs, commands/queries (handler próprio no DI; ver stack-versoes.md)
├── <Ctx>.Infrastructure/     # EF Core (DbContext, migrations), messaging, external, observability
├── <Ctx>.Api/                # interface: ASP.NET Core (Minimal APIs/Controllers), DI/composition root
└── <Ctx>.Shared/             # primitives transversais mínimas (§7)
tests/
├── <Ctx>.Domain.Tests/
├── <Ctx>.Application.Tests/
└── <Ctx>.IntegrationTests/   # Testcontainers
```

- **`Domain` não referencia nenhum outro projeto** (o `.csproj` do domínio não tem `ProjectReference` para Application/Infrastructure/Api nem pacote de framework/ORM) — a inversão de dependência é imposta pela própria árvore de referências.

- `Api` é o único que conhece todos (composition root); `Application` referencia só `Domain`; `Infrastructure` referencia `Domain` (+`Application` para implementar portas).

## 6. Complexidade e Tamanho

> **Canônico em `references/padroes-codigo.md`** (arquivo ≤ 750 linhas — ~500 de código útil + ~250 de comentário; flag em > 300 úteis; uma função/unidade por arquivo; doc-comment obrigatório com motivo/comportamento/entradas/saídas/efeitos; e `MAPA.md`). Esta seção é o recorte de arquitetura desses pisos — não duplica a regra, contextualiza.

**MUST — arquivos pequenos, micro-funções**

- **Teto duro: ≤ 750 linhas/arquivo** (~250 de comentário + até ~500 de código útil). Acima disso, o arquivo **deve ser quebrado** — extraia responsabilidades em módulos menores e a lógica em **micro-funções** com nome que explica a intenção. Não existe "arquivo de 1200 linhas porque é coeso": coesão real cabe em arquivos pequenos colaborando.

- **Flag em > 300 linhas de código útil** (não bloqueia, mas **sempre sinaliza**): passou de 300 úteis (ou ~400 em observabilidade), é **indício** de função muito extensa / falta de abstração — registre como dívida e **revise quando as prioridades forem resolvidas**.

- **Funções pequenas e de responsabilidade única.** Ideal ≤ 50 linhas; função grande vira micro-funções compostas. Use case: uma responsabilidade.

- Exceções (não disparam quebra): testes, migrations, código gerado, schemas/fixtures.

**MUST — toda função documentada**

- **TODA função/método/membro público tem comentário** no formato de doc da linguagem (**XML doc `///` em C#**; JSDoc, GoDoc, rustdoc). O comentário declara, no mínimo:
  - **O quê** — o que a função faz, em uma linha.
  - **Onde é usada / prevista** — quem chama, em que fluxo/camada ela foi pensada pra servir (ex.: "usado pelo use-case `CreateOrder`", "handler HTTP de `/v1/checkout`"). Isto dá contexto explícito de propósito e evita função órfã.
  - Parâmetros, retorno e efeitos colaterais relevantes.

- Esse comentário é a **fonte do índice de microfunções** (§39) — escreva pensando que ele será extraído e indexado, não como enfeite.

Convenção mínima (adapte à linguagem):

```
/**
 * O quê: valida o payload de checkout e cria o pedido.
 * Onde:  use-case CreateOrder; chamado pelo handler POST /v1/checkout.
 * Efeitos: persiste em orders, publica catalog.order.created via outbox.
 */
```

**Bloqueio rígido em CI**

- Arquivo de produção > 750 linhas (ou > ~500 de código útil) sem quebra (exceto as exceções acima) → bloqueia; > 300 úteis (~400 obs) → flag registrado.

- Função de produção sem doc-comment → bloqueia.

- Complexidade ciclomática > 15 em função de produção.

- Aninhamento > 4 níveis.

> Linha de código é proxy ruim para complexidade — complexidade ciclomática é a métrica honesta. Mas arquivo gigante e função sem contexto são dívidas óbvias: quebre e documente antes do merge.
