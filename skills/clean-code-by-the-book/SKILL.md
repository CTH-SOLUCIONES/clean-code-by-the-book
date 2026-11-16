---
name: clean-code-by-the-book
description: Summarizes what Robert C. Martin's Clean Code and Clean Architecture actually say, adapted per language (Go, Python, TypeScript, Rust, PL/pgSQL and Oracle, covering both SQL from services and PL/SQL). Use it when writing, reviewing or refactoring code, and when deciding the layers, boundaries and dependency direction of a service. It reports what the books say without invented thresholds, and marks where each language's idiom wins over the book. It does not replace the linter or the project's recorded decisions.
license: MIT
metadata:
  author: CTH Soluciones Tecnológicas
  version: "1.0.0"
---

# Clean Code and Clean Architecture, by the book

Sources: *Clean Code* (2008), by Robert C. Martin, with chapters by other authors —names (Tim
Ottinger), error handling (Michael Feathers), boundaries (James Grenning), simple design (Jeff Langr)
and concurrency (Brett Schuchert)— and *Clean Architecture* (2017), by Martin, with a final chapter
by Simon Brown. This skill summarizes what they say, marks where each language's idiom wins and where
this skill goes beyond the books or departs from them on purpose, and explains how it coexists with decisions the project
has already made. It is judgment, not law: each rule is a signal that calls for thought.

## What takes precedence over this skill

1. **The project's recorded decisions** (ADRs, design docs, written team conventions). Before moving
   logic between layers or drawing a boundary, look for them. A recorded decision may already have
   chosen a path that departs from the books —a team decision to keep domain logic in PL/pgSQL, a
   deliberately thin BFF— or that the books don't cover, such as a schema managed without migrations.
   Respect it and don't reopen it in passing. Only if the task forces going against it, ask before
   proceeding.
2. **The language's idiom and the repo's tools.** Formatting that the formatter and linter settle
   (gofmt and golangci-lint, ruff, eslint and prettier, rustfmt and clippy) is not debated in a
   review, and what the linter already flags is not repeated.
3. **The scope of the task.** The boy scout rule applies to what you touch, in small steps and with
   the tests green; it is not a license to refactor the module.

## Clean Code

### Names (ch. 2)
- They reveal intent; no disinformation or empty distinctions (`ProductData` versus `ProductInfo`).
- Pronounceable and searchable; length grows with scope (N5).
- No type encodings or member prefixes (N6). The `_` in Python and the capital letter in Go mark
  visibility: they are not encoding.
- **In PL/SQL, the prefix on parameters and variables (`p_id`, `l_total`) is scope encoding, but here
  it is not the N6 smell.** The book discards prefixes because the environment makes them useless, and
  in Oracle the environment resolves names badly: inside a SQL statement, a name that matches a column
  of the statement is read as the column. With a parameter `id`, `DELETE FROM t WHERE id = id` deletes
  every row, and `SET name = name` changes nothing. In new code, use the prefix; in an existing
  routine, or when in doubt, qualify every use with the routine's name (`delete_order.id`), because
  renaming a parameter breaks callers that use named notation. In PostgreSQL, PL/pgSQL rejects the
  clash at run time with an ambiguous-reference error, but a `LANGUAGE sql` function behaves like
  Oracle: the column wins.
- One word per concept, and not the same word for two ideas.
- Solution-domain names (patterns, algorithms) when the reader is a programmer; problem-domain names
  when there is no technical term.
- The name states the side effects (N7).

### Functions (ch. 3)
- **Small.** The book says they should hardly ever reach 20 lines. That is a signal, not a limit: in
  Go, explicit error checking lengthens functions that are perfectly clear. What matters is that they
  do one thing (G30) and descend one level of abstraction (G34).
- **The stepdown rule:** functions read from top to bottom and each one descends a level.
- **Arguments:** zero is ideal, one and two are fine, three are avoided and more need justification
  (F1). `ctx context.Context` in Go and `self` in Python and Rust don't count.
- **No flag arguments (F3) and no selector arguments (G15).** Split what the boolean selects without
  duplicating the body (G5): often it is enough to pull the varying part out of the function and let
  the caller compose it.
- **No hidden side effects, and command-query separation:** a function changes state or answers
  something, not both. Returning an error alongside the result does not violate this; additionally
  returning a computed value that the caller could ask for separately does. The exception is when the
  operation itself generates the value or when separating them breaks atomicity: `INSERT ...
  RETURNING`, a counter that returns its new value, a compare-and-swap or a `pop` go together. Never
  insert and then look the row up (`MAX(id)`, the most recent one): another session can insert in
  between. In Oracle, the equivalent is `RETURNING ... INTO` with output variables; `QueryRow(...
  RETURNING id)`, which is right in PostgreSQL, fails against Oracle
  ([references/oracle.md](references/oracle.md)).
- **No duplication** (G5).

### Comments (ch. 4)
- For the book, the good use of a comment is to make up for what you failed to say in code: before
  writing one, try to make a name or the structure say it.
- **Good:** legal, intent, clarification, warnings of consequences, TODO, amplification, and public
  API documentation, which says what the signature doesn't. A comment required by rule everywhere is
  bad according to the book. Go's convention asks for a doc comment on exported identifiers; Python's,
  a docstring on the public API; and Rust's, `///` on the public items of a library that other crates
  use. If the repo's linter requires it, the linter wins.
- **Bad:** redundant, misleading, journal comments, noise, commented-out code (C5) and information that
  lives elsewhere, such as history or the issue tracker (C1).
- Comments that explain *why* are kept: they are what the book calls explanation of intent.

### Objects and data structures (ch. 6)
- **Antisymmetry:** objects hide their data and expose behavior; data structures expose data and have
  no behavior. Hybrids get the worst of both.
- **The trade-off:** procedural code makes it easy to add functions without touching the data
  structures; object-oriented code makes it easy to add types without touching the functions. A
  TypeScript union with an exhaustive `switch`, or a Go `type switch` —with a `default` that fails
  loudly, because Go doesn't check exhaustiveness— are correct when what grows is the operations; the
  polymorphism of G23 is for when the types grow. In Rust, a `match` on your own `enum` is exhaustive,
  unless a wildcard switches that off and lets a new variant through silently (G23).
- **Law of Demeter:** don't navigate through what another object returns (`a.B().C().D()`; G36).
  Navigating a pure data structure —a DTO, a Go struct with no behavior— doesn't violate it. Neither
  does chaining iterator adapters (`iter().filter().map()`) or builder methods: each call transforms
  or configures, and none reaches into another object's internal structure.

### Error handling (ch. 7), by language
The book asks for exceptions instead of return codes, context on every error, error types defined by
what the caller needs, a normal flow without special cases, and not returning or passing null. How
this translates:

- **Go: errors are values.** They are returned as the last result, wrapped with context
  (`fmt.Errorf("save order %s: %w", id, err)`) and told apart with `errors.Is` and `errors.As`.
  Sentinel or typed errors, depending on what the caller needs to distinguish. `panic` only for
  programming errors. "Don't return null" translates to the zero value plus an error, or the
  `value, ok` pattern. A `nil` slice ranges without trouble, but `encoding/json` serializes it as
  `null`: in a response, initialize it (`[]T{}`) or have the contract say it may come as `null`.
- **Rust: errors are values, as in Go.** `Result` and `?`, with context added as the error propagates.
  Your own `enum` where the caller distinguishes cases, and `anyhow` at the edge of a binary, where
  the error is only reported. `unwrap` and `expect` only for programming errors. Nothing is discarded
  without saying why: `let _ =`, `.ok();`, `map_err(|_| ...)` and a `JoinHandle` that nobody awaits
  silence the error without the compiler warning. `Option` where the caller needs to know why there
  is no value is the smell; use a `Result` there. Legitimate exceptions and detail in
  [references/rust.md](references/rust.md).
- **Python:** exceptions, as in the book (EAFP). Your own types according to what the caller handles,
  with `raise ... from err` so the cause isn't lost; never an `except Exception` that swallows the
  error. In FastAPI, `HTTPException`, the Pydantic request models and the SQLAlchemy rows stay at the
  edge: they don't enter the domain.
- **TypeScript:** exceptions for the unexpected. **Departure from the book:** for expected domain
  errors this skill uses discriminated unions or a Result type, because the compiler forces the case
  to be handled, but only where the repo already uses them (G11); don't introduce a Result library on
  your own initiative. With `strictNullChecks`, absence is declared in the type and empty collections
  replace null ones.
- **PL/pgSQL:** `RAISE EXCEPTION` with your own `ERRCODE` and `DETAIL` or `HINT` carrying the context,
  so the calling service can tell the error apart.
- **PL/SQL.** A `WHEN OTHERS` goes only where it records context or releases something, and ends in
  `RAISE`, which keeps the ORA code the service translates; never in `NULL` or in
  `RAISE_APPLICATION_ERROR`, which replaces it. `RAISE_APPLICATION_ERROR` is for domain errors: a
  validation that fails or the translation of a specific exception (`DUP_VAL_ON_INDEX`,
  `NO_DATA_FOUND`) in its handler, and there with `TRUE` as the third argument so the cause isn't
  lost. It carries a code between -20000 and -20999 in a named constant (G25) and a message with
  identifiers, not personal data. Logging goes in an autonomous routine; otherwise the log is rolled back
  along with the exception ([references/oracle.md](references/oracle.md)).
- **Oracle from the service.** The gateway never sends the driver's error or its text to the log: it
  doesn't wrap it (`%w`, `source`), doesn't copy it (`%v`, `err.Error()`, `str(e)`), and in Python it
  raises its own error with `from None`. It keeps the ORA code and, if it knows them, the constraint
  or the column. The codes the use case distinguishes —ORA-00001 when a unique key collides,
  ORA-12899 when a value doesn't fit, your own -20xxx— are translated into domain errors; the rest
  leaves as an infrastructure error with its code, not as a bare 503 with no cause, which makes a
  failure undiagnosable. It is a deliberate exception to "don't lose the cause": in 23ai several
  messages carry the value that failed —the key in ORA-00001, the number in ORA-01438, the text in
  ORA-01722— which may be personal data (under a personal-data law, e.g., GDPR or Colombia's Law 1581
  of 2012), and SQLAlchemy adds the bound variables to any error unless `hide_parameters=True`. Length
  is validated before writing, and in a `VARCHAR2` `''` is `NULL`: detail in
  [references/oracle.md](references/oracle.md).

### Boundaries with third-party code (ch. 8)
Few places in the code touch a third-party API —the OCI SDK, an ORM, the Oracle driver, a vendor's
client— and its types don't circulate through the rest of the system: you wrap or adapt it at that
point, with learning tests that pin down how it behaves. In Rust, a `sqlx` row or a `reqwest`
response is converted into your own type right there. Don't wrap every use of the standard library
(`net/http`, `fetch`, `std`). Toward Oracle, every value travels as a bind variable, never
concatenated into the text. Identifiers and DDL, which can't be bound, come from constants in the code
or, if they come from configuration, go through `DBMS_ASSERT`; never from user input.

### Tests (ch. 9)
- Tests are first-class code: readable, one concept per test, build-operate-check and F.I.R.S.T.
  (fast, independent, repeatable, self-validating and timely).
- TDD, with its three laws, is the preferred practice, not a delivery requirement.
- **This skill's position:** whatever touches the database —repositories, gateways, SQL, PL/pgSQL,
  PL/SQL, concurrency— is tested against a real database (a container or the dev environment) **of
  the same engine**, not against in-memory doubles: testing against PostgreSQL doesn't test Oracle. A
  race is tested with two real connections, and test values come from what the real component
  produces, not from the shape you assume; in Oracle that includes values at the maximum length of
  each column, measured the way the column measures it and with accented letters and ñ. Doubles are for
  testing a use case in isolation.
- Test the boundaries (T5) and test exhaustively near a bug (T6).

### Classes and modules (ch. 10)
Small in responsibilities, not in lines; high cohesion; organized for change. SRP is read as *Clean
Architecture* refines it: a module answers to a single actor.

### Simple design (ch. 12)
Kent Beck's four rules, in order: it passes all the tests, it has no duplication, it expresses
intent, and it has the minimum number of classes and functions.

### Concurrency (ch. 13)
- Concurrent code is a responsibility of its own: keep it apart from the logic.
- Limit the scope of shared data, prefer copies, keep critical sections small and shut down in an
  orderly way, which is hard to write well.
- An intermittent failure is treated as a possible concurrency problem, not as "noise".
- **This skill's position:** in the database, a state transition goes in a single conditional
  statement (`UPDATE ... WHERE status = :expected`) and you check how many rows it touched. Reading
  without locking, validating and writing separately is a race; if the validation doesn't fit in the
  `WHERE`, read with `SELECT ... FOR UPDATE` in the same transaction.
- In Go: channels or mutexes with a clear owner, `context` for cancellation and `go test -race`.
- In Rust, `Send` and `Sync` prevent data races in safe code, but not deadlocks or logic races: the
  chapter still applies. Guards across an `.await`, blocking calls and cancellation with tokio are in
  [references/rust.md](references/rust.md).

## Clean Architecture

- **The dependency rule.** Source-code dependencies point inward, toward the policies. What is inside
  names nothing from outside: no framework types, no database rows, no HTTP. What crosses a boundary
  is simple structures, in the form that suits the inner side.
- **Entities and use cases.** Critical business rules don't depend on frameworks; use cases
  orchestrate and receive and return simple structures.
- **SOLID.** SRP per actor; OCP; LSP; ISP with small interfaces; DIP with the interface on the side of
  whoever uses it: the inner circle defines it and the outer one implements it. In Go this matches the
  idiom: the interface is declared where it is consumed. In Rust, the trait lives in the inner module
  or crate and the outer one implements it.
- **Boundaries are decided, not placed by default.** A full boundary is expensive, and adding it when
  it is already needed is also expensive and risky. The book asks you to weigh both: decide which
  ones are implemented fully, which partially and which are ignored for now, watch for friction and
  draw them at the inflection point. Partial boundaries come in three forms: prepare everything to
  separate but keep it in the same component, a strategy, or a facade.
- **Humble Object at every boundary.** What is hard to test is kept humble and minimal: the view, the
  HTTP handler, the database gateway implementation. What has logic is tested: the presenter and the
  interactor with unit tests, and, per this skill's testing position, the gateway against a real
  database.
- **Main is just another plugin:** the dirtiest component, the only one that knows the concrete
  implementations and wires them together. In Go it lives in `main` or in a wiring package; in
  FastAPI, in the `Depends` providers and at startup. In Rust, *The Rust Programming Language* (ch.
  12, "Separating Concerns in Binary Projects") recommends a `main.rs` that only reads arguments and
  configuration, calls `run` and handles its error, with the logic in `lib.rs`; a `main` that only
  wires modules already meets that. In a small command-line tool, that separation is often all the
  boundary needed.
- **Services are not architecture by themselves.** If they share the data model they remain coupled
  even though the network separates them, and each service needs its own internal architecture.
- **The database and the web are details; the data model is not.** What is a detail is the technology
  that stores the data, not the shape of the data.
- **Components.** REP, CCP and CRP decide what goes together. ADP: no cycles. SDP: depend in the
  direction of stability. SAP: stable means abstract. What is stable and concrete falls in the "zone
  of pain", and the book's example is the database schema: everything depends on it and it is costly
  to change. It only hurts if it changes often. With logic in the database (PL/pgSQL or PL/SQL) and a
  schema managed without migrations, a signature change is first tested against a disposable database
  of the same engine; then it is applied by hand to each environment's database, with the previous
  definition backed up and without breaking whoever still calls the old signature; and finally it is
  carried into the startup code, which only creates what is missing. If startup isn't updated, a new
  environment is born different and nothing fails. In PostgreSQL, `CREATE OR REPLACE` with a new
  parameter doesn't replace: it creates another overload. Without a `DEFAULT`, the old call keeps
  silently going to the old version; with a `DEFAULT`, it becomes ambiguous and fails. Renaming a
  parameter is rejected. In Oracle, replacing a package that has state makes the next call fail once
  in every session that was already using it (ORA-04068). All of this without reopening the decision to keep
  the logic there. In Rust, splitting a full boundary that is already decided into the crates of a
  workspace puts the compiler in charge of guarding it ([references/rust.md](references/rust.md)).
- **Screaming architecture, and Simon Brown's chapter.** The package structure says what the system
  does. Brown recommends packaging by component and, above all, having the compiler enforce
  encapsulation: in Go, unexported identifiers and `internal/`; in Rust, `pub(crate)` and crate
  boundaries; in PL/SQL, whatever is not in the package specification is private to its body.

## How to apply it

### When writing
Names and functions as above; errors by language; tests in the same change. A new layer or boundary
only if it isolates an expected change, and say which one.

### When reviewing
For each finding:

- the principle, with its chapter-17 code when it has one (for example "G14 Feature Envy") or the
  *Clean Architecture* concept;
- where (`file:line`);
- why it matters in that code, not in the abstract;
- the smallest concrete change.

Order: first correctness or security errors, whether or not they are in the catalog; then violations;
finally signals. A **violation** has a concrete cost in that code: a bug, a change that already forces
touching two places, a caller that can't tell one error from another. A **signal** depends on
something you can't see —the type, the callers, the tests— or is a matter of judgment: say what data
would settle it. What already existed outside the change is mentioned separately and briefly. No
global scores. If nothing is worth raising, say so.

The review does not widen the scope, and its depth follows the risk:

- Only blocking and important findings call for a round of fixes; minor ones go on a list, and what
  already existed before the change goes to the backlog.
- Don't propose a queue, a cache, a separate process, a layer or a configuration option without a
  requirement, a measurement or an incident that calls for it. Where there is personal data, tenant
  isolation, legal deadlines, hostile input or money, the review goes deep.
- If there is a spec, measure against it: a behavior that is not written down, or that contradicts
  it, is an important finding, even if the code is good.

### What not to do
- Invent thresholds —lines, milliseconds, number of dependencies— that the books don't give.
- Introduce patterns or layers "because clean architecture asks for it": a layer that only delegates
  is complexity.
- Propose moving logic between layers, or into or out of the database, when a recorded project
  decision covers it.

## References

- [references/chapter-17-heuristics.md](references/chapter-17-heuristics.md): the complete catalog of
  smells and heuristics from chapter 17 with their real numbering, and per-language notes where the
  idiom changes the reading. Consult it when naming a finding in a review.
- [references/rust.md](references/rust.md): errors, tokio, `main.rs` and `lib.rs`, workspaces and what
  rustc and clippy don't warn about by default. Read it when writing or reviewing Rust.
- [references/oracle.md](references/oracle.md): what bites in Oracle without the compiler warning
  —name capture, errors with personal data, `RETURNING ... INTO`, `''` and `NULL`, length in bytes,
  decimals, `MERGE`, package changes—, checked against Oracle 23ai. Read it when writing or reviewing
  Oracle SQL or PL/SQL.
