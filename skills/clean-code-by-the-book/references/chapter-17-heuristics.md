# Smells and heuristics: chapter 17 of *Clean Code*

Numbering and names as in the book, in English so they can be cited. Each entry has one line of
meaning and, where the language changes the reading, a note for Go, Python, TypeScript, Rust or
Oracle. Figures the book does not give are not given here. Main file: [SKILL.md](../SKILL.md).

## Comments
- **C1 Inappropriate Information:** metadata that belongs in git history or the issue tracker.
- **C2 Obsolete Comment:** a comment that no longer matches the code; fix it or delete it.
- **C3 Redundant Comment:** repeats what the code already says.
- **C4 Poorly Written Comment:** if a comment is worth writing, write it well.
- **C5 Commented-Out Code:** delete it; history keeps it.

## Environment
- **E1 Build Requires More Than One Step:** one command builds (`make`, `go build ./...`,
  `cargo build --workspace`).
- **E2 Tests Require More Than One Step:** one command runs all the tests (`go test ./...`,
  `pytest`, `npm test`, `cargo test --workspace`). Without `--workspace`, in a workspace with a root
  package, `cargo test` runs only the root and passes green even if a member fails.

## Functions
- **F1 Too Many Arguments:** zero is ideal, three are to be avoided, more needs justification. `ctx`
  and `self` do not count.
- **F2 Output Arguments:** the result is returned, not written into an argument. Go: decoding into a
  pointer (`json.Unmarshal(data, &v)`) is the standard library's convention, not a smell. Rust:
  filling a borrowed buffer (`reader.read(&mut buf)`) is the `std::io` convention. PL/SQL: an `OUT`
  parameter is the way to return several values from a procedure; if it returns a single value and
  modifies no data, a function fits better. A procedure that inserts and returns the key in an `OUT`
  is fine, because a function with DML cannot be called from a query (ORA-14551). Turning a procedure
  that already has callers into a function is a signature change: flag it, do not demand it.
- **F3 Flag Arguments:** a boolean that selects behavior. Split what it selects without duplicating
  the body (G5): often it is enough to pull out the varying part and let the caller compose it.
- **F4 Dead Function:** a function nobody calls; delete it.

## General
- **G1 Multiple Languages in One Source File:** mixing languages (SQL, HTML, shell) in one file is
  kept to a minimum. SQL in strings: named queries or `.sql` files, always with parameters (in
  Oracle, bind variables).
- **G2 Obvious Behavior Is Unimplemented:** the function does not do what its name leads you to
  expect.
- **G3 Incorrect Behavior at the Boundaries:** edge cases left uncovered; do not trust intuition,
  test them.
- **G4 Overridden Safeties:** silencing warnings or skipping tests to quiet a failure: `//nolint` or
  `# type: ignore` without a reason, `as any`, `--no-verify`, a test skipped because it fails. In
  Rust, `#[allow(...)]` without a reason, `let _ =` or `.ok()` on a `Result` without a reason, and an
  `unsafe` block without the `// SAFETY:` comment saying why it is correct. In PL/SQL,
  `WHEN OTHERS THEN NULL`. (A skipped test that documents an open question is T4.)
- **G5 Duplication:** the most obvious form is the repeated block of code; subtler is the `switch`
  or `if/else` chain that reappears in several modules (solved with polymorphism); subtler still,
  similar algorithms with different lines (Template Method or Strategy).
- **G6 Code at Wrong Level of Abstraction:** low-level details in the high-level interface, or the
  reverse.
- **G7 Base Classes Depending on Their Derivatives:** the base does not know its derived classes. In
  Go this applies to packages: the inner one does not import the outer one.
- **G8 Too Much Information:** wide interfaces that expose too much; prefer small interfaces (Go's
  `io.Reader` is the model). Rust: `pub` only on what another crate needs, and `pub(crate)` for
  internals. PL/SQL: in the package specification, only what is called from outside.
- **G9 Dead Code:** branches that never run; delete them.
- **G10 Vertical Separation:** variables and functions close to where they are used.
- **G11 Inconsistency:** if something is done one way, everything similar is done the same way.
- **G12 Clutter:** empty constructors, unused variables, comments that carry no information.
- **G13 Artificial Coupling:** things that do not depend on each other, coupled only because they sit
  in the same place.
- **G14 Feature Envy:** a method that uses another object's data more than its own.
- **G15 Selector Arguments:** an argument that selects what the function does; split it as in F3.
- **G16 Obscured Intent:** code that hides what it does (cryptic expressions, chained magic
  numbers).
- **G17 Misplaced Responsibility:** the code is not where a reader would look for it.
- **G18 Inappropriate Static:** a function that should be polymorphic was made fixed. Go and Python:
  package-level and module-level functions are normal; the smell is when another implementation
  should be able to replace it.
- **G19 Use Explanatory Variables:** give intermediate calculations a name.
- **G20 Function Names Should Say What They Do.**
- **G21 Understand the Algorithm:** that it works is not enough; you must know why it works.
- **G22 Make Logical Dependencies Physical:** if a module depends on something in another, it should
  ask for it explicitly instead of assuming it.
- **G23 Prefer Polymorphism to If/Else or Switch/Case:** a single selection by type, which creates
  the polymorphic objects. It applies when what grows is the set of types; when the operations grow
  instead, the right tools are a TypeScript union with an exhaustive `switch`, a Rust `match` on your
  own `enum` with no wildcard arm (`_` or a catch-all binding; on a `#[non_exhaustive]` enum from
  another crate the wildcard is mandatory), or a Go `type switch` with a `default` that fails loudly
  (Go does not check exhaustiveness) (ch. 6).
- **G24 Follow Standard Conventions:** Effective Go, PEP 8, the repo's eslint configuration, the Rust
  API guidelines with rustfmt and clippy.
- **G25 Replace Magic Numbers with Named Constants.**
- **G26 Be Precise:** do not use floats for money, do not ignore concurrency, do not assume a query
  returns a single record. Money goes as an integer in the smallest unit or in a decimal type the
  repo already uses; changing the type in an existing contract (a public JSON, a column) is the
  owner's decision, and in a review it is flagged along with what it would break. Oracle: do not
  assume `''` and `NULL` are different in a `VARCHAR2`, nor that a value fits its column without
  measuring it in the column's unit (with `BYTE` semantics, an accented letter or an ñ takes two
  bytes), nor that python-oracledb returns a `NUMBER` with decimals as `Decimal` (by default it
  arrives as `float`). Details in [oracle.md](oracle.md).
- **G27 Structure over Convention:** enforce the decision with structure (types, interfaces) before
  doing it with a naming convention.
- **G28 Encapsulate Conditionals:** `if shouldBeDeleted(timer)` instead of the compound condition.
- **G29 Avoid Negative Conditionals.**
- **G30 Functions Should Do One Thing.**
- **G31 Hidden Temporal Couplings:** if order matters, make the signature enforce it (each step
  receives what the previous one returns).
- **G32 Don't Be Arbitrary:** the structure has a reason, and it shows.
- **G33 Encapsulate Boundary Conditions:** the `+1` or `-1` at the edge goes in a named variable.
- **G34 Functions Should Descend Only One Level of Abstraction.**
- **G35 Keep Configurable Data at High Levels:** configuration comes in at the top (environment
  variables, a ConfigMap or, if it is secret, a Secret) and is passed down.
- **G36 Avoid Transitive Navigation:** no `a.B().C()`; each module knows only its direct
  collaborators.

## Java (and how it reads in the languages this skill covers)
- **J1 Avoid Long Import Lists by Using Wildcards:** specific to Java. In Python and TypeScript it is
  the opposite: explicit imports, no `from x import *` or `import *` except for namespaces. Go: no
  dot imports.
- **J2 Don't Inherit Constants:** constants are imported from where they live, not inherited.
- **J3 Constants versus Enums:** use the language's enumerated type. Go: a dedicated type with
  constants; string-based (`type Status string`) if the value is stored or serialized, with `iota`
  if it is only internal. Python: `Enum`. TypeScript: a union of literals or `as const`. Rust:
  `enum`.

## Names
- **N1 Choose Descriptive Names.**
- **N2 Choose Names at the Appropriate Level of Abstraction:** the name states the concept, not the
  implementation.
- **N3 Use Standard Nomenclature Where Possible:** pattern names and the project's domain language.
  Rust: the conversion prefixes from the API guidelines (`as_`, `to_`, `into_`) tell you the cost and
  whether the value is consumed.
- **N4 Unambiguous Names.**
- **N5 Use Long Names for Long Scopes:** `i` is fine in a short loop (the book's example is a
  five-line scope).
- **N6 Avoid Encodings:** no type or scope prefixes. Python's `_` and Go's capital letter are
  visibility, not encoding. PL/SQL: the prefix on parameters and variables (`p_`, `l_`) is scope
  encoding, but Oracle's name resolution justifies it: inside a SQL statement, a name that matches a
  column is read as the column. It is not flagged as N6 (see the naming guidance in
  [SKILL.md](../SKILL.md)).
- **N7 Names Should Describe Side-Effects:** `createOrGet`, not `get`, if it creates.

## Tests
- **T1 Insufficient Tests:** test everything that could fail.
- **T2 Use a Coverage Tool!** (`go test -cover`, coverage.py, c8 or istanbul).
- **T3 Don't Skip Trivial Tests:** they document and are cheap.
- **T4 An Ignored Test Is a Question about an Ambiguity:** a skipped test that documents an
  unclarified requirement, with the reason in plain sight. (Skipping it to silence a failure is G4.)
- **T5 Test Boundary Conditions.**
- **T6 Exhaustively Test Near Bugs:** bugs come in clusters.
- **T7 Patterns of Failure Are Revealing:** how the tests fail guides the diagnosis.
- **T8 Test Coverage Patterns Can Be Revealing:** what does not run explains why what does run
  fails.
- **T9 Tests Should Be Fast:** a slow test stops being run. The book gives no figure.
