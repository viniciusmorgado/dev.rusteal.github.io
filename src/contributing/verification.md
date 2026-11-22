# Formal verification

Tests and the engine harness check the inputs someone thought of. A few parts of Rusteal take inputs nobody can enumerate (a static library a user hands to the CLI, or a buffer whose length the engine reports), and a mistake in them silently corrupts a binary or memory instead of failing a test. For those parts, and only those, Rusteal uses [Verus](https://github.com/verus-lang/verus) to prove the code correct for every input (decision I9).

Verus is a verifier for Rust, under the MIT license. A function gets a specification: what it requires of its inputs (`requires`) and what it guarantees (`ensures`). Verus then proves, with an SMT solver, that every execution meets it, overflow included: arithmetic on machine integers must be proven not to wrap. Specifications and proofs are ghost code that is erased before the program is compiled, so a proven function runs exactly as the same function without them.

## Where Rusteal uses it

| Part | What is proven | Why there |
|---|---|---|
| `rusteal-cli/src/shared_std.rs`, ELF first, then Mach-O (universal libraries too) and COFF (bigobj too) | Every read and write stays inside the file. No offset taken from a header overflows. The only bytes that change are those of the `rust_eh_personality` symbol's entries (binding, `n_desc` or storage class). A malformed file is refused, never a panic. | It parses and rewrites any library it is given, from offsets the file itself declares (D66). A wrong offset produces a library that links and then fails at run time, on a platform this repository's tests may not run. |
| `rusteal-core/src/containers.rs` and `strings.rs`, the buffers that container elements and strings cross the boundary in | Decoding reads only bytes the engine wrote and the buffer holds. Encoding writes at most `write_size()` bytes. | The length comes from the engine: today the code assumes the engine wrote no more than the buffer holds (`written <= buf.len()`) and never checks it. A proof makes that condition explicit, as a check or as a stated precondition. |

The engine's side is not verified: what the engine writes is a precondition the proof states, not something it proves.

## Where it is not used, and why

- **The C++ boundary.** The generated bindings, the API tables and the plugin. Verus sees only Rust, while most of the risk in a binding layer is in how the two sides agree. The contract tests and the engine harness check that.
- **Object lifetimes, the garbage collector and the game thread.** Rust's types already check these (D61–D65): lifetimes, `!Send`, and the debug check of every engine call. A proof would restate what the compiler checks.
- **The macros, the code generator and the CLI's commands.** What they emit is checked by compiling it: the trybuild tests, `templates-check` and the harness.
- **The async executor, tasks, `Drop`-based types, and anything built on `std::sync::Mutex` or `RwLock`.** Verus does not support async or `Drop`, and supports locks only through its own library.
- **Game code and the templates.** A game developer never needs Verus, just as they never need `unsafe` (I7).

## Why it stays small

A proof costs more than the code it proves, and it has to be kept up whenever the code changes. Verus also brings constraints of its own:

- **Its own Rust toolchain.** Verus pins the Rust version it is built with, 1.100.0 in November 2026, and moves it with its releases. The workspace's Rust version is chosen for the engine and for game developers, not for Verus.
- **A moving tool.** Releases come out every week, and attributes are renamed between them.
- **A subset of Rust.** Some features are partial (raw pointers, `dyn`, closures that capture mutably), and some are missing (async, `Drop`, `transmute`).
- **Platforms.** Prebuilt releases exist for Ubuntu, macOS and Windows. Other Linux distributions build it from source.

So a part joins the table above only when three things hold: its mistakes cannot surface as a failing test, its inputs cannot be enumerated, and it is plain Rust with no engine in the loop. Adding one is a decision recorded in I9.

## How it is kept apart

- The shipped crates never depend on Verus's crates (`vstd`, `verus_builtin`, `verus_builtin_macros`). They do not change their Rust version for it either. Building Rusteal, or a game with it, never involves Verus.
- Verification is its own command, `cargo xtask verify` (planned), run by whoever changes a verified part. It is not part of the CI checks; adding it there is a separate decision.
- When a proof needs the code in another shape, the shipped code is refactored to that shape and keeps its behaviour and its tests. An example is a function over a byte slice instead of a raw pointer.
- How the proof stays attached to the code it proves is settled by the pilot, on `shared-std`'s ELF path. The rule above holds whatever the pilot picks: the verified code is the shipped code, never a copy kept beside it.

## Status

Planned, as a parallel track of the [road to 1.0](https://github.com/viniciusmorgado/rusteal/blob/develop/docs/roadmap-1.0.md#parallel-tracks) that does not gate 1.0. Its tasks are in the checklist's Formal verification section.
