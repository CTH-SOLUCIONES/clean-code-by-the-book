# Rust: what changes when writing and reviewing

Read this when writing or reviewing Rust. It is not a language manual: these are the points where
the rules of the skill ([SKILL.md](../SKILL.md)) apply differently, or where rustc and clippy stay
silent. Whatever was verified was verified with Rust 1.99, clippy 0.1.99 and tokio 1.53.

## Errors
- **The error type comes from what the caller needs.** A custom `enum`, written by hand or with
  `thiserror`, when the caller tells cases apart with `match`. `anyhow` with `.context(...)` at the
  edge of a binary, where errors are only reported, e.g., a service binary. The two complement each
  other. What you do not do is add a second library for the same thing the one already in the repo
  solves (G11, see [chapter-17-heuristics.md](chapter-17-heuristics.md)).
- **The cause is not lost when translating.** The custom variant keeps it, with `#[source]` or
  `#[from]` if using `thiserror`, or in a field. The exception is a database error whose message
  carries values from the row: for that one, the variant keeps the code and the constraint, not the
  error (see "Oracle from the service" in [SKILL.md](../SKILL.md) and [oracle.md](oracle.md)).
  `map_err(|_| ...)` throws the cause away, and neither rustc nor clippy flags it by default. It is
  only legitimate when the discarded error says nothing the new message does not say: a
  `TryFromIntError`, or `env::var` of a mandatory variable with a message that says "missing or not
  UTF-8", because that error has both cases. A channel's `SendError` does not qualify: it carries
  the value that was not sent.
- **`unwrap` and `expect` are for programmer errors**, like `panic` in Go. User input, files, the
  network and the database are not. Outside tests, an `expect` that says why it cannot fail.
- **Discarding a `Result`.** The compiler warns about an unused `Result`, but `let _ =` and `.ok();`
  silence it, and clippy says nothing either. It is legitimate with a comment or a name that
  explains it: the receiver of a channel that is already gone, or `env::var("X").ok()` when the
  absence is the meaning.
- **The `JoinHandle` of a `tokio::spawn` is awaited, collected in a `JoinSet` or supervised.** If
  nobody looks at it, the task's `Err` is lost, the panic only lands on stderr, outside the
  structured log, and the process carries on as if nothing had happened.
- **`Option` versus `Result`.** "Do not return null" is already met by `Option`. The smell is using
  it where the caller needs to know why there is no value; there a `Result` belongs.
- **In a terminal tool**, an input error (arguments, a file that does not exist, a missing column)
  goes to stderr with a message and a non-zero exit code: `main` returns `Result` or
  `std::process::ExitCode`, it does not panic.

## Concurrency with tokio
- `Send` and `Sync` prevent data races in safe code. They do not prevent deadlocks or logic races.
- **The guard of a `std::sync::Mutex` lives in its own block, which closes before the `.await`.**
  With `drop(guard)` the compiler still counts it as alive: the future stops being `Send`,
  `tokio::spawn` rejects it, and clippy flags it too (`await_holding_lock`). `tokio::sync::Mutex`,
  only if it really has to be held across the `.await`.
- Blocking work (CPU, large files, a synchronous library) goes in `spawn_blocking`.
- **A future is cancelled when it is dropped**: the losing branch of a `select!`, a `timeout` that
  expires, an `abort()`. It can happen at any of its `.await` points. Dropping the `JoinHandle` does
  not cancel the task, it leaves it detached. What cannot be left half-done goes in a database
  transaction (the one in `sqlx` rolls back if dropped without `commit`, according to its
  documentation) or in its own task, not inside a `select!` or a `timeout`.

## `main.rs`, `lib.rs` and workspaces
- *The Rust Programming Language* (ch. 12, "Separating Concerns in Binary Projects") recommends that
  `main` limit itself to reading arguments and configuration, calling `run` and handling its error,
  with the logic in `lib.rs` so it can be tested. A `main` that only wires modules together, e.g.,
  a service binary's, already meets this and is not a finding. In a small tool, that separation is
  usually the whole boundary.
- **Workspace crates, only when a complete boundary has already been decided.** Then the compiler
  guards it: Cargo rejects cycles between normal dependencies (it does admit them through
  `dev-dependencies`), and a domain crate that does not list `sqlx` in its `Cargo.toml` cannot name
  it.
- In a workspace, `cargo build --workspace` and `cargo test --workspace`. Without the flag, with a
  root package, `cargo test` runs only the root and passes green even if a member fails.
- `pub` only on what another crate uses; `pub(crate)` for what is internal (G8).

## `match`, names and documentation
- A `match` on your own `enum` with no catch-all arm (`_` or a bare binding name) is exhaustive: a
  new variant does not compile until it is handled. The catch-all turns that off. On a
  `#[non_exhaustive]` enum from another crate, such as `std::io::ErrorKind`, the catch-all is
  mandatory (E0004).
- The conversion prefixes of the API guidelines (`as_`, `to_`, `into_`) state the cost and whether
  the value is consumed (N3). Clippy does not check them on the public API unless configured.
- `///` on the public items of a library that other crates use, not in a `lib.rs` that only its own
  `main.rs` uses. If the repo enables `#![warn(missing_docs)]`, the linter rules.
- An `unsafe` block carries a `// SAFETY:` comment that says why it is correct (G4).

## Outputs and money
- An output that is compared or versioned is not built by iterating a `HashMap`: its order changes
  from one run to the next. Use a `BTreeMap` or sort the keys.
- Money does not go in `f64` (G26): integers in the smallest unit, or a decimal type the repo
  already uses.
