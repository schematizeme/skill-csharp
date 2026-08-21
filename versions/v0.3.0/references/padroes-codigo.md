# padroes codigo — recorte C#/.NET

> **PONTEIRO, não cópia.** A normativa deste tema é da base: **`schematize-engineering`** →
> `references/padroes-codigo.md`. Leia lá primeiro; aqui fica **só o que muda em C#/.NET**.
>
> **Onde este arquivo divergir da base, a BASE MANDA** (`SKILL.md` §"Precedência e herança").
> Este arquivo era um clone com **97%** de conteúdo
> idêntico à base — deriva por cópia foi o achado da Classe C da vistoria de 2026-08-21, e ela
> já tinha atingido piso de segurança (o `argon2id-only` da casa virou "ou PBKDF2" numa skill,
> o rol de 6 linguagens virou "só Go e Rust" em três). Manter uma cópia é manter a próxima deriva.
## 3. Tudo comentado: motivo + comportamento esperado

Toda função/membro público carrega um **doc-comment** (no formato nativo: **XML
doc `///` em C#** — com `<summary>`/`<param>`/`<returns>`/`<remarks>`; `///` em
Rust, doc comment em Go, JSDoc/TSDoc em TS, docstring em Python) que responde, no
mínimo:
