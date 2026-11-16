# clean-code-by-the-book

An [Agent Skill](https://agentskills.io) that applies what *Clean Code* and *Clean Architecture* by Robert C.
Martin actually say. It covers Go, Python, TypeScript, Rust, PL/pgSQL and Oracle (SQL from services, and
PL/SQL).

## Why this skill

Coding agents already know the vocabulary of clean code. What they get wrong is the content.

Many published "clean code" skills turn the books' guidance into hard limits ("functions must be under
20 lines", where the book only says they should hardly ever reach 20), add limits the books never set
("max 3 dependencies"), and score code from 0 to 10. They also apply Java-era advice to
languages where it does not fit. The result is review noise and refactors nobody needed.

This skill takes a stricter line:

- **It reports what the books say, and no more.** It invents no thresholds. Where the book gives a
  number, it treats that number as a signal, not a limit.
- **The language's idiom wins where it should.** In Go and Rust, errors are values. A Python `_` prefix
  is not "encoding". A `type switch` is fine when operations grow and types don't.
- **It marks its own positions.** Where it goes beyond the books or departs from them on purpose, it
  says so. Testing against a real database and single-statement state transitions go beyond the books;
  Result types for expected errors in TypeScript depart from them.
- **It catches database traps the compiler won't.** It flags name capture in PL/SQL, `INSERT` followed
  by `MAX(id)`, overloads in PostgreSQL's `CREATE OR REPLACE`, and Oracle errors that leak personal
  data into logs.
- **Its reviews are actionable.** Each finding names the principle and its chapter-17 code
  (`G14 Feature Envy`), gives `file:line`, says why it matters in that code, and proposes the smallest
  concrete change. Violations come before signals. There are no global scores.

It is opinion with sources, not law: every rule is a signal that asks for judgment. Project decisions
that are already recorded, the repo's linter and the scope of the task take precedence over it.

## What's inside

| File | Loaded | Content |
|---|---|---|
| [`SKILL.md`](skills/clean-code-by-the-book/SKILL.md) | When the skill activates | Clean Code by chapter, Clean Architecture, how to write and how to review |
| [`references/chapter-17-heuristics.md`](skills/clean-code-by-the-book/references/chapter-17-heuristics.md) | When naming a finding | The full catalog of smells and heuristics (C, E, F, G, J, N, T) with the book's numbering, with notes per language |
| [`references/rust.md`](skills/clean-code-by-the-book/references/rust.md) | When writing or reviewing Rust | Errors, tokio, `main.rs`/`lib.rs`, workspaces, what rustc and clippy don't warn about |
| [`references/oracle.md`](skills/clean-code-by-the-book/references/oracle.md) | When writing or reviewing Oracle SQL or PL/SQL | Name capture, errors carrying personal data, `RETURNING ... INTO`, `''` vs `NULL`, byte length, `MERGE`, package changes. Checked against Oracle 23ai |

## Install

**Any agent that supports skills** (Claude Code, Codex, Gemini CLI, Cursor, GitHub Copilot and others),
with the [`skills`](https://github.com/vercel-labs/skills) CLI:

```sh
npx skills add CTH-SOLUCIONES/clean-code-by-the-book        # current project
npx skills add CTH-SOLUCIONES/clean-code-by-the-book -g     # every project (global)
```

**Claude Code**, as a plugin:

```
/plugin marketplace add CTH-SOLUCIONES/clean-code-by-the-book
/plugin install clean-code-by-the-book@clean-code-by-the-book
```

**By hand:** copy `skills/clean-code-by-the-book/` into your agent's skills folder, for example
`~/.claude/skills/clean-code-by-the-book/` for Claude Code.

## Use

The skill activates on its own when you write, review or refactor code, or when you decide the layers and
boundaries of a service. You can also ask for it directly:

- "Review this diff with clean-code-by-the-book."
- "Should this handler call the repository directly, or does it need a use case in between?"
- "Is this PL/SQL procedure safe to rename a parameter on?"

## Scope and limits

- **Languages:** Go, Python, TypeScript, Rust, PL/pgSQL, and Oracle SQL and PL/SQL. Java appears only
  where chapter 17 is Java-specific.
- **Not a linter:** formatting, and anything gofmt, golangci-lint, ruff, eslint, prettier, rustfmt or
  clippy already report, is left to those tools.
- **Not a mandate to refactor:** the boy-scout rule applies to the code you touch, in small steps and with
  the tests passing.
- **Not a substitute for recorded decisions:** if your project chose to keep domain logic in the
  database, or to keep a BFF thin, the skill respects that choice and doesn't reopen it.

## Sources

- Robert C. Martin, *Clean Code: A Handbook of Agile Software Craftsmanship* (Prentice Hall, 2008). It
  includes chapters by Tim Ottinger, Michael Feathers, James Grenning, Jeff Langr and Brett Schuchert.
- Robert C. Martin, *Clean Architecture: A Craftsman's Guide to Software Structure and Design* (Prentice
  Hall, 2017). Its last chapter is by Simon Brown.
- Steve Klabnik, Carol Nichols and Chris Krycho, *The Rust Programming Language*, chapter 12
  ("Separating Concerns in Binary Projects").

This project is not affiliated with or endorsed by the authors or publishers. The skill paraphrases the
books and cites the chapter-17 heuristics by their code and name only; to learn the material, read the
books.

## Contributing

Issues and pull requests are welcome. Every rule has to be traceable to the books, to a language's
official documentation, or to a behavior you can reproduce (say the version you tested). Please don't
add thresholds the books don't give. Read [CONTRIBUTING.md](CONTRIBUTING.md) before you open a pull
request.

## License

[MIT](LICENSE) © 2026 CTH Soluciones Tecnológicas
