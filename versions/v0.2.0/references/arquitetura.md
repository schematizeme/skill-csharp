# Arquitetura, Camadas, Repositórios e Linguagens

> Parte da skill **schematize-csharp**. As referências cruzadas (§N) apontam para seções do corpo completo — todas presentes no conjunto de references desta skill.

## Índice
- 2. Estrutura de Repositórios
- 3. Linguagens
- 4. Arquitetura
- 5. Estrutura de Pastas
- 6. Complexidade e Tamanho
- 7. Dependências Internas e Shared Libraries
- 8. CQRS e Padrões de Aplicação

---

## 2. Estrutura de Repositórios

**MUST**
- Um repositório = uma aplicação ou um bounded context.
- Comunicação entre serviços via HTTP, gRPC, eventos ou mensageria — nunca via banco compartilhado.
- Cada serviço é dono do seu schema.
- **Nome do repositório:** `<projeto>_<contexto>[_<lang>]` em snake_case minúsculo. `<projeto>` = slug do produto/organização; `<contexto>` = a aplicação/bounded context daquele repo (`api`, `worker`, `front`, `backoffice`, `gateway`…); `_<lang>` é sufixo **opcional** pra desambiguar linguagem (`_cs` C#/.NET, `_go` Go, `_rs` Rust, `_ts` TypeScript). Como um repo = um contexto, o nome espelha isso. Ex.: `loja_api_cs`, `loja_front`, `loja_worker_cs`.
- **Independência de runtime (cada serviço é entidade à parte):** todo serviço **sobe e opera sozinho**. A indisponibilidade de qualquer outro serviço **nunca** impede o boot nem derruba este — depender de outro serviço para *iniciar/funcionar* é VETADO (nada de "o `ledger` não sobe se o `core` estiver fora"). Dependente ausente vira **degradação graciosa** (fallback, resposta parcial, enfileira e segue), nunca crash em cascata. Como não perder o dado quando a chamada falha: `references/dados-eventos.md` (§18).
- **`<projeto>_ops` (control plane de desenvolvimento):** todo sistema multi-repo tem um repo **`<projeto>_ops`** — a ferramenta de operação do workspace, rodada por dev/agente e **fora do runtime do produto**. Faz bootstrap/instalação, update, manutenção, troubleshooting e roda os testes unitários/debug **através de todos os repos** (clona, sobe/para, migra, semeia e testa cada serviço). Não é microserviço nem é deployado com o produto; é essencial pra tocar um sistema de múltiplos repositórios. Como toda ferramenta, sobe com **observabilidade integrada** (Grafana/LGTM+, ver `references/observabilidade.md` §16).
- **Contenção no workspace (nunca sair da pasta do projeto):** o **diretório de projeto atual é o workspace**; toda aplicação/repo do sistema nasce e mora **dentro dele**. Vai criar uma aplicação nova? Crie uma **pasta pra ela dentro da pasta atual** (`./<projeto>_<contexto>/`) e trabalhe lá — **nunca** largue arquivos soltos no root pra depois **subir de diretório** (`cd ..`, `../`) e criar os outros repos fora. Num sistema multi-repo os repos são **irmãos dentro do mesmo workspace** (clonados ali pelo `<projeto>_ops`), não espalhados pela máquina. **VETADO** criar/ler/escrever fora do workspace: diretório-pai, `~`, `~/Documents`, `~/Downloads`, `/tmp` do usuário, Área de Trabalho. O agente **não sai da pasta do projeto** — nem pra vasculhar, nem pra criar — a menos que o usuário peça explicitamente.

**VETADO**
- **Aplicação monolítica que acopla múltiplos bounded contexts num só deploy/processo.** Não se cogita "começar monolito e quebrar depois" sem ADR explícito de plano e prazo de quebra. Misturar domínios de negócio "pra entregar rápido" é dívida disfarçada de produtividade.
- **Monólito distribuído** — o pior dos dois mundos: serviços separados fisicamente, mas acoplados por banco compartilhado, shared lib de domínio (§7) ou chamadas síncronas em cascata sem fronteira. Tão proibido quanto o monólito clássico.
- **Big ball of mud** — código sem fronteira de contexto, onde tudo importa tudo.

**SHOULD**
- Evitar mais de um domínio de negócio no mesmo repositório.
- Leitura cross-service por réplica read-only só com ADR e contrato documentado.

**MAY**
- *Modular monolith* (módulos com fronteira de contexto rígida, schema separado, comunicação por interface interna) **somente com ADR** que justifique estágio do produto e contenha o plano de extração. É exceção registrada, não default. Nunca usar como atalho para colar domínios.

> Não existe "MVP monolítico que vira microserviço depois" sem o ADR que prova que o depois tem data. Sem isso, "depois" é "nunca", e "nunca" é um big ball of mud em produção.

---

---

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

**MUST**
- Versão exata em uso fica no Anexo A / `references/stack-versoes.md` (alvo: **.NET LTS atual, 8/9**).
- Não misturar linguagens dentro do **mesmo bounded context** sem ADR.
- **Todo serviço backend novo nasce no rol sancionado, com ADR de fit.** C# entra pelo encaixe acima — não por gosto, e não como "porque já sei C#" sem o fit.
- **Nova linguagem fora do rol** exige **ADR de exceção** aprovado.

**SHOULD**
- Frameworks são bem-vindos; abstrações mágicas não. Critério: consigo entender o stack trace? (ASP.NET Core e EF Core passam; "mágica" que esconde a query/o middleware, não.)

### 3.1 Fora do rol — Node-backend e PHP (legado/saída, não reabrem)

- **Node como serviço backend** e **PHP** **não recebem serviço novo.** O que existe é **legado**: fica como está até ser tocado e **migra para uma linguagem do rol** (aqui, quando o fit é .NET, para **C#**) medido **por funcionalidade do módulo**, não por linha.
- **Modelo da métrica.** Um módulo tem N funcionalidades. O quanto uma mudança "pesa" é a fração alterada/criada sobre o total. Ex.: módulo com 10 funcionalidades — refatorar 3 (ou criar ~4 novas) ≈ 30%.
- **Gatilho de extração (~30%):** ao atingir ~30% das funcionalidades do módulo, **não cresça o legado** — extraia para um **serviço/projeto C# à parte** (novo `.csproj`/bounded context) e incorpore o comportamento na nova base.
- **Virada dos 50%:** quando ~50% já estiver extraído/substituído, **migra-se o resto de uma vez** — encerra o módulo legado.
- **Ajuste pontual não porta.** Mudança pequena (abaixo do gatilho) é feita no próprio legado.
- Toda migração registra ADR (§27) e segue o DDD híbrido/coexistência (§4.X, §36): flag de coexistência, sem big-bang. Detalhe da saída do Node em `schematize-node`.

> O anti-padrão de linguagem **não** é "usou C# em vez de Go" — é **abrir serviço backend novo fora do rol sancionado, ou dentro dele sem ADR de fit**. O ganho marginal de tooling não reabre Node/PHP como backend.

---

---

## 4. Arquitetura

**MUST — todos os projetos**
- Separação explícita de camadas: `domain`, `application`, `infrastructure`, `interface`.
- Inversão de dependência: domínio não conhece infra.
- Domínio não importa frameworks nem ORM.

**SHOULD — projetos com regra de negócio relevante**
- DDD tático (agregados, value objects, eventos de domínio).
- Arquitetura hexagonal (ports & adapters).

**MAY — CRUDs simples**
- Manter as 4 camadas, dispensar táticas DDD pesadas.

### Dependências permitidas

```
interface       → application
application     → domain
infrastructure  → domain, application
```

### Dependências proibidas

```
domain          → qualquer outra camada
domain          → frameworks, ORM, libs de IO
application     → interface
```

### Anti-Corruption Layer

**MUST** em integrações com sistemas externos: adapter dedicado em `infrastructure/external/` que traduz o modelo externo para o modelo de domínio. **Nunca** expor DTOs externos diretamente no domínio.

### 4.X DDD híbrido durante transição

Projetos legados onde código já existe sem separação de camadas **podem** adotar DDD progressivamente em vez de big-bang. Regras:

**MUST**
- Toda nova feature/refactor em código tocado segue o layout completo (`domain/`, `application/`, `infrastructure/`, `interface/`) — não introduzir mais código "flat".
- Ao mover/quebrar arquivo legado, organize já em folders DDD mesmo que internamente alguma classe ainda misture responsabilidades (ex.: service em `application/` ainda chamando SQL direto). Estrutura primeiro, inversão depois.
- Cada PR que toca arquivo híbrido **deve** mover ao menos um pedaço pra direção certa (ex.: extrair entidade pra `domain/`, mover query pra `infrastructure/repositories/`).
- ADR registrando o débito e o plano de remoção: `<project>/docs/adr/<n>-ddd-migration-<contexto>.md`.

**SHOULD**
- Manter teste de cobertura por camada (§22) durante a transição — domain começa com 0%, sobe a cada PR.
- Em C#, a **própria árvore de `ProjectReference`** é o guard: o `.csproj` de `Domain` não referencia framework/ORM nem os outros projetos; violar isso não compila. Onde o legado ainda é um projeto único, use um **guard test** (ex.: teste de arquitetura com `NetArchTest`/análise de `using`) que **rejeita dependências proibidas** (mesmo com whitelist de exceções legadas):
  - `Domain` não usa `Microsoft.AspNetCore.*`, `Microsoft.EntityFrameworkCore`, `System.Data.*`, nem os namespaces de `Infrastructure`/`Application`/`Api`.
  - `Application` não depende de `Api` (interface).

**MAY**
- Marcar arquivos híbridos com comment `// @ddd-hybrid` pra busca fácil e cleanup priorizado.

---

---

## 5. Estrutura de Pastas

### C# (.NET) — projetos por camada dentro da solution

Uma **solution** (`.sln`) por bounded context; **um projeto (`.csproj`) por camada** (referência de projeto força a direção de dependência — o compilador vira o guard de camada). `Directory.Build.props` + `Directory.Packages.props` na raiz.

```
src/
├── <Ctx>.Domain/            # entities, value-objects, domain services/events, interfaces de repositório
├── <Ctx>.Application/        # use-cases, DTOs, commands/queries (MediatR opcional)
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

### Node.js

```
src/
├── domain/           # entities, value-objects, services, events, repositories (interfaces)
├── application/      # use-cases, dto, commands, queries
├── infrastructure/   # persistence, messaging, external, observability
├── interface/        # http, grpc, cli
├── shared/
└── config/
tests/
```

### Go

```
cmd/<app-name>/
internal/
├── domain/
├── application/
├── infrastructure/
├── interface/
├── shared/
└── config/
tests/
```

### Rust

```
src/
├── domain/
├── application/
├── infrastructure/
├── interface/
├── shared/
└── config/
tests/
```

---

---

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

---

---

## 7. Dependências Internas e Shared Libraries

**MUST**
- Shared libraries são **mínimas** e com escopo claramente delimitado.
- Permitido como shared: observabilidade, autenticação/auth, primitives de infraestrutura, SDKs internos, logging, configuração.

**MUST NOT**
- Criar `commons` / `core-lib` / `platform-utils` genéricos.
- Compartilhar **lógica de domínio** entre bounded contexts.
- Compartilhar entidades de domínio. Cada contexto modela o seu.

> O caminho mais rápido pra um monólito distribuído é uma shared lib chamada `commons`.

**SHOULD**
- Shared libs versionadas com SemVer próprio.
- Breaking changes em shared lib exigem ADR.

---

---

## 8. CQRS e Padrões de Aplicação

- **Commands**: alteram estado, retornam id ou void.
- **Queries**: nunca alteram estado, otimizadas para leitura, podem usar projeções.
- CQRS **não exige** event sourcing.

---

---
