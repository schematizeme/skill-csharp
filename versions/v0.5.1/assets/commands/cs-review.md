---
description: Roda o gate da Definition of Done e anti-padrões (§35, §37) no diff atual
argument-hint: "[ref git, ex: origin/main]"
---

Faça o **review de padrões** do diff atual (contra $ARGUMENTS ou `origin/main`),
combinando o checker determinístico com seu julgamento.

1. Rode `bash scripts/check-diff.sh ${ARGUMENTS:-origin/main}` e leia o resultado.
2. Some a isso a análise que o script NÃO faz bem sozinho:
   - **§37 (anti-padrões):** segredo no cliente, SQL concatenado/interpolado (EF Core `FromSqlRaw`/Dapper sem parâmetro), auth no client,
     `tenant_id` do body, JWT sem validar, `Guid.NewGuid`/`Random` pra token (use `RandomNumberGenerator`), `catch {}` que engole
     erro, `async void`/`.Result`/`.Wait()`, teste silenciado, etc.
   - **Lint/formatação:** **analyzers Roslyn + `.editorconfig`** ativos, `<TreatWarningsAsErrors>true` (warning-as-error) e
     `dotnet format --verify-no-changes` limpo. **Nullable reference types habilitado** (`<Nullable>enable</Nullable>`) —
     `#nullable disable` ou `!` (null-forgiving) espalhado pra calar o compilador é achado.
   - **§6:** arquivo >750 linhas (ou >~500 úteis) → bloqueia; >300 úteis (~400 obs) → flag; função/membro público sem `///` XML doc (o quê + onde) → bloqueia.
   - **§39:** o índice de funcionalidades foi atualizado no mesmo PR?
   - **§3:** backend novo em Node/PHP? (proibido).
3. Produza um relatório com `BLOQUEIA` (viola piso/DoD) e `ATENÇÃO` (melhorar),
   citando arquivo:linha. Se houver qualquer `BLOQUEIA`, a task **não está pronta** (§35).

Seja específico e acionável — aponte o conserto, não só o problema.
