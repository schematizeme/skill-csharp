# Async e Concorrência (C# / .NET)

> Parte da skill **schematize-csharp**. Os erros mais caros de um backend .NET não
> aparecem no compilador — aparecem em runtime: deadlock de sync-over-async, thread
> pool faminto, `CancellationToken` ignorado, `async void` que engole exceção, e
> fila sem backpressure que estoura memória. Esta reference é o piso de
> concorrência. Liga com `stack-versoes.md` (ASP.NET Core) e `dados-eventos.md`
> (resiliência, jobs, outbox).

## Índice
- A1. async/await de ponta a ponta — nada de sync-over-async
- A2. `CancellationToken` — propagar sempre
- A3. `Task` vs `ValueTask`, `async void` e exceções
- A4. Estado compartilhado, locks e `await`
- A5. Backpressure, `Channel<T>` e paralelismo limitado
- A6. Shutdown gracioso e `IHostedService`

---

## A1. async/await de ponta a ponta — VETADO sync-over-async

**VETADO**
- **Bloquear em código async:** `.Result`, `.Wait()`, `.GetAwaiter().GetResult()`,
  `Task.WaitAll`/`WaitAny` em caminho de request/produção. Num contexto com
  `SynchronizationContext` (ou sob pressão de thread pool) isso **deadlocka** ou
  **esfomeia o pool** — cada chamada bloqueada segura uma thread que não volta.
- **Trabalho bloqueante/CPU-bound dentro de `async`** sem isolar (I/O síncrono,
  criptografia pesada, parsing gigante, `Thread.Sleep`): trava uma thread do pool
  e derruba a latência de todas as outras requisições.

**MUST**
- **async/await do controller/endpoint até o driver.** Se a camada de dados expõe
  API async (EF Core `ToListAsync`/`SaveChangesAsync`, `HttpClient.SendAsync`,
  `NpgsqlCommand.ExecuteReaderAsync`), **use a versão async** — nunca a síncrona
  embrulhada em `Task.Run`.
- **Espera temporal** por `await Task.Delay(...)`, nunca `Thread.Sleep`.
- **CPU-bound de verdade, num app de console/desktop/worker** → `await Task.Run(...)` (offload
  deliberado), medido. **Em ASP.NET Core, `Task.Run` para CPU-bound é anti-padrão documentado:**
  não existe "thread de UI" para liberar, e o `Task.Run` usa **o mesmo thread pool** que já está
  atendendo as requisições — você paga troca de contexto e uma thread a mais para fazer o mesmo
  trabalho, no mesmo lugar, e piora a latência sob carga (é, aliás, um jeito conhecido de
  transformar pico de CPU em thread pool starvation). O que resolve de verdade um CPU-bound pesado
  no servidor: **tirá-lo do caminho da request** (fila + `BackgroundService`, ou outro serviço),
  não movê-lo de thread. I/O **nunca** vai em `Task.Run` — não é isso que ele resolve.
- **Toda operação externa tem timeout** (`HttpClient.Timeout`, `CancellationTokenSource`
  com `CancelAfter`, `WaitAsync(timeout)`) — nada espera pra sempre.

**SHOULD**
- Em bibliotecas (não no app final), `ConfigureAwait(false)` no `await` para não
  capturar contexto desnecessário. Em ASP.NET Core (sem `SynchronizationContext`) é
  menos crítico, mas mantém a lib portável; padronize via analyzer.

---

## A2. `CancellationToken` — propagar é piso

O `CancellationToken` é como o cliente que desconectou, o timeout que estourou, ou
o SIGTERM chegam até o trabalho em voo. Engoli-lo é desperdiçar recurso e travar o
shutdown.

**MUST**
- **Todo método async público em caminho de I/O recebe `CancellationToken`** (por
  convenção o último parâmetro) e o **repassa** a cada chamada async interna
  (`await db.SaveChangesAsync(ct)`, `await http.SendAsync(req, ct)`). O
  endpoint/handler ASP.NET Core já recebe o `CancellationToken` da request por
  injeção — encadeie-o.
- **Loops longos e streams checam o token** (`ct.ThrowIfCancellationRequested()` /
  `await foreach (... .WithCancellation(ct))`).
- **Cancelamento não é erro engolido:** `OperationCanceledException` esperada
  (shutdown, disconnect) é tratada como fim limpo; qualquer outra exceção segue os
  pisos (loga + decide retry/parar).

**VETADO**
- Assinar método async sem `CancellationToken` "porque ninguém cancela". Aqui a atribuição precisa
  ser exata, senão o time descobre no CI que a regra não existe: **`CA2016` é *encaminhar* o token
  que você já recebeu** para a chamada interna que o aceita, e **`CA1068` é a posição** dele (último
  parâmetro). **Nenhum analyzer built-in obriga a DECLARAR o parâmetro** — isso é **regra de
  revisão** desta casa, e quem quiser automatizá-la escreve um analyzer/Roslyn rule próprio ou uma
  convenção verificada em code review. As duas built-in **travam o CI** (com
  `TreatWarningsAsErrors`), e as duas continuam valendo.
- Passar `CancellationToken.None` só pra calar o compilador quando há um token real
  disponível no fluxo.

---

## A3. `Task` vs `ValueTask`, `async void` e exceções

**MUST**
- **Nunca `async void`** (exceto handler de evento de UI, que não existe no backend).
  `async void` **não é aguardável**, e sua exceção **sobe direto pro thread pool e
  derruba o processo** — sem quem a capture. Use `async Task`.
- **Exceção em `async Task` é capturada no `await`** — trate-a; não deixe `Task`
  "fire-and-forget" sem observar (o `TaskScheduler.UnobservedTaskException` é rede,
  não estratégia).
- **`ValueTask` só onde há ganho medido** (hot path que frequentemente completa
  sincronamente). Regra do `ValueTask`: **await uma única vez, nunca armazenar,
  nunca `.Result`** — consumo incorreto é bug de corrupção difícil de achar. Na
  dúvida, `Task`.

**SHOULD**
- Trabalho em segundo plano disparado de um request vai para um **`Channel<T>` +
  `BackgroundService`** (ou fila/outbox), **não** para um `_ = DoAsync()` solto que
  perde a exceção e morre junto com o request.

---

## A4. Estado compartilhado, locks e `await`

**MUST**
- **Nunca segurar um `lock` através de um `await`.** `lock`/`Monitor` não pode
  envolver `await` (não compila com `lock`, mas o antipadrão aparece com `Mutex`/
  `SemaphoreSlim` mal usados): segurar seção crítica enquanto aguarda é
  deadlock/contenção. Para exclusão mútua **assíncrona**, use
  `SemaphoreSlim(1,1).WaitAsync()` com `try/finally` liberando — e segure o mínimo.
- **Preferir imutabilidade e tipos concorrentes** a lock manual: `record`
  imutável, `ConcurrentDictionary`, `Interlocked` para contadores, `ImmutableArray`.
- **Serviço singleton registrado no DI precisa ser thread-safe** (é compartilhado
  entre requests concorrentes); `DbContext` é **scoped e NÃO thread-safe** — nunca
  compartilhe uma instância entre tasks paralelas (use `IDbContextFactory` para
  paralelismo).

**SHOULD**
- Quando a contenção aparecer, prefira **passar mensagem** (uma task dona do estado
  consumindo um `Channel<T>`) a compartilhar memória com lock.

**VETADO**
- Estado mutável estático (`static` mutável) sem sincronização — corrida garantida
  sob carga concorrente.

---

## A5. Backpressure, `Channel<T>` e paralelismo limitado

**MUST**
- **Filas in-process são limitadas (`bounded`).** `Channel.CreateBounded<T>(n)` com
  `n` definido e **`BoundedChannelFullMode` explícito** (`Wait` = backpressure,
  `DropOldest`/`DropWrite` com métrica). `CreateUnbounded` em caminho de produção é
  **VETADO sem ADR** — é OOM esperando carga.
- **Fan-out concorrente é limitado.** Disparar N chamadas de uma vez contra um
  upstream sem teto derruba ele e você. Use `SemaphoreSlim` como limitador, ou
  `Parallel.ForEachAsync(source, new ParallelOptions{ MaxDegreeOfParallelism = k }, ...)`.
- **`Task.WhenAll` sobre coleção não-limitada** só depois de limitar o grau de
  paralelismo — `WhenAll` de 10k tasks é 10k conexões simultâneas.

**SHOULD**
- Streaming de resultados via `IAsyncEnumerable<T>` + `await foreach` (memória
  constante) em vez de materializar listas gigantes.
- `Parallel.For`/`Parallel.ForEachAsync` só para trabalho **independente**; ordem e
  efeitos colaterais compartilhados exigem cuidado (A4).

---

## A6. Shutdown gracioso e serviços de fundo

**MUST**
- **Trabalho de fundo é `BackgroundService`/`IHostedService`**, não `Task.Run` no
  `Startup`. O host injeta o `stoppingToken`; **respeite-o** para parar de aceitar
  trabalho, **drenar** o que está em voo (com timeout) e sair ordenadamente.
- **Graceful shutdown (SIGTERM):** o `IHostApplicationLifetime`/host dispara o
  cancelamento; feche conexões, dê flush em buffers/telemetria (OTel) e conclua o
  request em andamento dentro do `ShutdownTimeout`. Casa com o deploy destrutivo do
  `references/ops.md` (o serviço para limpo, o ops recria o clone).
- **Toda task de fundo tem o resultado/exceção OBSERVADO** — `Task.WhenAll` (ou
  `Task.WhenEach`/`Parallel.ForEachAsync` quando a forma pede) sobre as tasks que você guardou;
  task que falha reporta (log estruturado + métrica) e decide retry/parar, não morre em silêncio
  (piso "erro nunca engolido"). Task disparada e esquecida (`_ = FazAlgo();`) é o caminho mais
  curto para exceção que ninguém vê. *(Nota de higiene: `JoinSet` **não é do .NET** — é do Tokio,
  em Rust. Estava aqui como vazamento da skill de Rust, achado da vistoria de 2026-08-21; é o tipo
  de erro que a cópia entre skills irmãs produz.)*

**SHOULD**
- `Activity`/`ILogger` com escopo por task/requisição (OpenTelemetry .NET) para
  observar a concorrência real (liga com `observabilidade.md`).

> Regra de bolso: **async de ponta a ponta, nunca `.Result`/`.Wait()`; propague o
> `CancellationToken`; nada de `async void`; limite toda fila e todo fan-out; e
> assuma que qualquer `await` pode ser cancelado no pior ponto.** O compilador C#
> não garante concorrência correta — isso é com você.
