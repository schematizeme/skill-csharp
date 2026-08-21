# dados eventos — recorte C#/.NET

> **PONTEIRO, não cópia.** A normativa deste tema é da base: **`schematize-engineering`** →
> `references/dados-eventos.md`. Leia lá primeiro; aqui fica **só o que muda em C#/.NET**.
>
> **Onde este arquivo divergir da base, a BASE MANDA** (`SKILL.md` §"Precedência e herança").
> Este arquivo era um clone com **95%** de conteúdo
> idêntico à base — deriva por cópia foi o achado da Classe C da vistoria de 2026-08-21, e ela
> já tinha atingido piso de segurança (o `argon2id-only` da casa virou "ou PBKDF2" numa skill,
> o rol de 6 linguagens virou "só Go e Rust" em três). Manter uma cópia é manter a próxima deriva.
## 10. Banco de Dados

- Migrations versionadas, **reversíveis** (com `down`; em C#, `dotnet ef` migrations com `Down` implementado), automatizadas no deploy.

- **Toda query com input externo é parametrizada** por desenho. Concatenação de string em SQL é **VETADA** (§37). Em C#, EF Core ou Dapper com parâmetros; nunca string interpolation em SQL.

**Ferramentas sugeridas:** `dotnet ef` migrations (com `Down`), `golang-migrate`, `node-pg-migrate`, `sqlx-cli`, `flyway`.

## 19. Jobs e Workers

- Cancelamento gracioso (respeitar context/signal; em C#, `CancellationToken` propagado).

- Fila in-process com `System.Threading.Channels` (`Channel<T>`) para desacoplar produtor/consumidor quando não exigir broker.
