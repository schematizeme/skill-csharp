# Testes — recorte C#/.NET

> **PONTEIRO, não cópia.** A **disciplina de teste** da casa é da **`schematize-qa`**: a pirâmide,
> teste de COMPORTAMENTO (não "renderizou"), o "verde de verdade" (smoke com asserção de conteúdo +
> assertion negativa + self-check que força uma falha conhecida), cobertura útil, a11y, regressão
> visual, contrato/dados, **flaky** (quarentena com prazo e dono), o fluxo **plan-first**
> (`/qa-plan` → `/qa-run`) e os **gates de CI que travam o merge**. Leia
> `schematize-qa` → `references/estrategia.md`, `references/categorias.md`,
> `references/execucao.md` e `references/flaky.md`.
>
> **Segurança ofensiva** (rejeição rota a rota, injeção/coerção, IDOR/BOLA, cross-tenant) é a
> **`schematize-pentest`** — não é Q.A. e não mora aqui.
>
> Aqui fica **só o que muda em C#/.NET**: o runner, a sintaxe, e as armadilhas do dialeto.
>
> *(Este arquivo e a antiga reference *testes-execucao* eram, juntos, ~450 linhas por skill — 66% já
> duplicado na `schematize-qa`, 23% que pertence à `schematize-pentest` e ~2% idiomático de
> verdade. Deriva por cópia foi o achado da Classe C/D da vistoria de 2026-08-21.)*

## O runner e o comando

```bash
dotnet test --logger "trx;LogFileName=test.trx"
dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=cobertura
dotnet test --filter "Category!=Integration"
```

## O que muda de forma em C#/.NET

- **xUnit é o default da casa.** `[Fact]` para caso único, `[Theory]` + `[InlineData]`/
  `[MemberData]` para tabela. Uma classe de teste por SUT; o construtor é o `setup` e
  `IAsyncLifetime` cobre o async.
- **`async void` em teste é veneno**: a exceção some e o teste passa. Sempre `async Task`.
- **`ConfigureAwait(false)` não pertence ao teste** — ele existe para biblioteca; no teste
  atrapalha o contexto de sincronização.
- **`WebApplicationFactory<TEntryPoint>`** para testar a API de ponta a ponta em memória, com o DI
  real: é onde o guard de efeito externo (`references/iam.md` §3.1) é exercitado de verdade,
  trocando o provider por sink no `ConfigureTestServices`.
- **EF Core: `InMemory` NÃO é banco.** Ele aceita SQL que o Postgres recusa e ignora constraint.
  Teste de repositório usa **Testcontainers** com o banco real; `InMemory` só para lógica pura.
- **`FluentAssertions`** para asserção legível; **`Verify`** para snapshot.
- **Property-based:** `FsCheck`. **Mutation:** Stryker.NET.

## Onde divergir da base, a base manda

O piso é o mesmo: teste é **visto falhar no vermelho** antes de valer; cobertura é **contrato**
(não se baixa a régua para passar o CI); **teste nunca dispara efeito externo real** — endereço no
domínio de teste em rota nula, provider = sink, cap por execução, e a caixa se confere **lendo do
sink** (`references/iam.md` §3.1 desta skill; normativa em `schematize-engineering` →
`references/efeitos-externos.md`); e **gate não se desliga "por enquanto"**.
