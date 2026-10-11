# Start here

This handbook is for whoever works on Rusteal itself: its crates, its CLI, its Unreal Engine plugins, its templates and its release. It is meant to be enough on its own: everything a developer new to the project needs to understand it, change it and release it, and the reasons behind how it is built. Using Rusteal to make a game is the other handbook's subject, the [game developer's handbook](https://viniciusmorgado.github.io/game.rusteal.github.io/); read its first chapters too, since you will test every change the way a game developer uses it.

## A reading order

| Day | Read | So that you can |
|---|---|---|
| 1 | [Introduction](../intro/introduction.md), [The repository](repository.md), [Glossary](glossary.md) | Name the pieces and find them. |
| 1 | [Setting up a checkout](../contributing/environment.md) | Build the workspace and a test project on your machine. |
| 2 | [Architecture](../architecture/architecture.md), [The Rust–C++ boundary](../architecture/boundary.md) | Follow a call from Rust to the engine and back. |
| 2 | [Reification and the macros](../architecture/reification.md), [Types declared in Rust](../internals/rust_types.md) | Follow a `#[uclass]` from the macro to a class in the editor. |
| 3 | [Code generation](../architecture/codegen.md), [The CLI](../architecture/cli.md) | Follow `rusteal build` step by step. |
| 3 | [The runtime](../architecture/runtime.md), [Errors](../architecture/errors.md), [Hot reload](../architecture/hot_reload.md) | Know what the runtime offers and how a reload works. |
| 4 | [Testing a change](../contributing/testing.md), [Common tasks](../contributing/tasks.md), [Known issues and sharp edges](../contributing/known_issues.md) | Make a change and prove it works. |
| 4 | [Decisions](../decisions/index.md) | Know why things are as they are before changing them. |
| 5 | [Templates](../templates/porting.md), [Releases and versions](../contributing/releases.md) | Port a template, ship a release. |

## A first change, end to end

1. Set up the checkout and make a test project from it (`cargo run -p rusteal -- new Probe --dir <dir> --runtime-path .`), as [Setting up a checkout](../contributing/environment.md) and [Testing a change](../contributing/testing.md) describe.
2. Open the project, place a `HelloActor`, press Play: the Output Log shows the greeting from Rust.
3. Change `HelloActor`'s message in the test project's crate, run `cargo run -p rusteal -- watch <project>` from the checkout, save: the editor reloads the library by itself (see [Hot reload](../architecture/hot_reload.md)).
4. Change something in Rusteal: a new `ulog!` line in `rusteal-runtime`'s `init`, say. `cargo run -p rusteal -- build <project>` rebuilds the game's library against your checkout; the reload shows your line.
5. Change the C++ plugin: `cargo run -p rusteal -- setup <project>`, then `build --all`, and restart the editor.
6. Run `cargo test --workspace` and `cargo clippy --workspace --all-targets -- -D warnings`, commit with a Conventional Commits subject on a branch made from `develop`, and open a pull request to `develop` (see [The repository](repository.md#branches)).

## Rules that are not negotiable

- **Commits**: Conventional Commits subjects (the release computes the version from them), small commits, everything in English. See [Tracking documents and conventions](../contributing/tracking.md).
- **The boundary**: a change to the API table is made on both sides at once, in the same order (see [The Rust–C++ boundary](../architecture/boundary.md#before-changing-the-boundary)).
- **Versions**: never edit a version number by hand; the release writes them (see [Releases and versions](../contributing/releases.md)).
- **The CI skip marker**: never write it in a commit message or a pull request, not even to talk about it.
- **Documentation**: a change a game developer sees updates the game developer's handbook; a change to how Rusteal works, or a decision, updates this one ([Decisions](../decisions/index.md#recording-a-decision)).
