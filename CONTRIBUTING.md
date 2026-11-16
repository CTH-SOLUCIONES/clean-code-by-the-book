# Contributing

Thanks for helping. This skill is only useful if it is accurate, so a contribution has to meet a
higher bar for correctness than for style.

Everyone who takes part follows the [Code of Conduct](CODE_OF_CONDUCT.md).

## What makes a change acceptable

1. **Every rule is traceable.** It comes from *Clean Code* or *Clean Architecture* (cite the chapter),
   from a language's or a database's official documentation (link it), or from a behavior you
   reproduced. If you reproduced it, say the tool and the version: `go 1.27`, `rustc 1.100`,
   `PostgreSQL 18`, `Oracle 23ai Free`.
2. **No invented thresholds.** Don't add line counts, dependency counts, milliseconds or scores that
   the books don't give. Where the book gives a number, it stays a signal, not a limit.
3. **Say what kind of claim it is.** Something the books say is written as theirs. Where the skill
   goes beyond them or departs from them, mark it with "This skill's position:" or "Departure from
   the book:".
4. **The language's idiom wins.** A rule that fights the idiom of the language needs a strong reason.
   Errors as values in Go and Rust, or EAFP in Python, are examples of such idioms.
5. **No verbatim passages from the books.** Paraphrase them. Heuristic codes and names
   (`G14 Feature Envy`) are fine.
6. **Keep `SKILL.md` lean.** It is loaded every time the skill activates. Detail for a single language
   goes in `references/`, linked one level deep from `SKILL.md`.

Corrections of facts are the most valuable contributions: a wrong ORA code, a rule that misreads a
chapter, an example that doesn't compile.

## How to propose a change

- **An error or a doubt:** open an issue with the "Inaccurate rule" form. Include the source.
- **A fix or an addition:** fork the repo, make the change on a branch, and open a pull request. Fill
  in the template.
- **A new language or a big section:** open an issue first, so that we agree on the scope before you
  write it.

Before opening the PR, validate the skill locally. CI runs the same check:

```sh
pipx run --spec skills-ref==0.1.1 agentskills validate skills/clean-code-by-the-book
```

## Versions and releases

Add a line under `## Unreleased` in [CHANGELOG.md](CHANGELOG.md). The maintainers choose the version
number when they cut a release, following [Semantic Versioning](https://semver.org): a new rule or
reference is a minor version, and a correction is a patch. Don't change the version in `SKILL.md` or
in the plugin manifests.

## License

By contributing, you agree that your contribution is licensed under the [MIT License](LICENSE).
