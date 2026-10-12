# Common tasks

Step by step, the changes that come up most. Each links to the chapter explaining the parts it touches.

## Add a service to the API table

When Rust needs something from the engine that neither the bindings nor reflection give.

1. Add the function pointer to its sub-table in `rusteal-ffi/src/api_table.rs`, and at the **same position** in `ue_plugin/Rusteal/Source/Rusteal/Public/RustealApiTable.h`.
2. Implement it in the matching `Private/Rusteal*ApiImpl.cpp` and add it at the same position of that file's positional `G*Api = {...}` initializer. Fail with `RustealFail(code, "reason")`.
3. Shared structs: `#[repr(C)]`, named `Rusteal*`, with size and offset asserts in `rusteal-ffi/src/contract_tests.rs` and `Private/RustealContractTests.cpp`.
4. Wrap it in `rusteal-core` behind a safe function (`crate::ffi_dispatch::<table>_<field>`), returning `RustealResult` through `check_ffi`, `#[track_caller]`.
5. Test: `cargo test --workspace`, then `setup` and `build --all` in the test project, and the editor.

See [The Rust–C++ boundary](../architecture/boundary.md).

## Bind an engine module

1. In every `rusteal-cli/templates/*/*/rusteal.toml.tera` (and the plugin templates that need it), add `Package = { module = "...", feature = "...", plugin = "..." }` to `[codegen.modules]`, and the feature to `features` where it should be on.
2. In the test project, the same lines in `rusteal.toml`, then `rusteal build --all`.
3. Link errors: blocklist the functions (next task).
4. Helpers reflection cannot describe: a file in `rusteal-codegen/manual/`, gated by `#[cfg(feature = "...")]`, listed in `manual/mod.rs`; prelude entries in `rust_gen/prelude.rs`.
5. `cargo test -p rusteal-codegen`; a fixture update if the module should be tested without the engine (`regenerate_fixture`).

See [Code generation](../architecture/codegen.md).

## Blocklist a function that does not link

1. Note the wrapper's symbol in the linker error: `Rusteal_<Class>_<Function>`.
2. Add `"<Class>.<Function>"` to `[codegen.blocklist] functions`, sorted, in every template's `rusteal.toml.tera` that carries the blocklist, and in the test project.
3. `rusteal build --from 2` in the test project.

## Add a macro attribute

1. Parse it in `rusteal-macros` (`uclass.rs`, `uclass_impl.rs`...), with a compile error for invalid input rather than silently ignoring it.
2. If it changes what the class is made of, include it in the shape (`shape.rs`), or hot reload will keep a stale class.
3. Generate the registration calls; if the reify API lacks what it needs, add it first (first task).
4. Test it in `rusteal-codegen/tests/manual_compiles.rs` (`uclass_types_compile`) and in the editor.
5. Document it in the game developer's handbook, [Attributes](https://viniciusmorgado.github.io/game.rusteal.github.io/reference/attributes.html).

## Change the C++ plugin

1. Edit `ue_plugin/`. Format with `.clang-format` (includes keep their order).
2. `cargo run -p rusteal -- setup <project>` and `build --all`, then restart the editor: plugin code is never hot reloaded.
3. After a new source file or a changed `UCLASS`/`USTRUCT` header, run `cargo xtask dev-setup` again so clangd sees it.
4. The published CLI embeds a snapshot of the plugin, written by `rusteal sync-plugin`; the release workflows run it, nothing to commit.

## Add a CLI command or flag

1. The clap definition in `rusteal-cli/src/main.rs`, a module of its own for the work, the version check (`check_version`) if it acts on a project.
2. A test of the parsing in `main.rs`'s tests.
3. The game developer's handbook: [CLI reference](https://viniciusmorgado.github.io/game.rusteal.github.io/reference/cli.html) and the chapter about it.

## Add a template or a variant

See [Template directories](../templates/directories.md) and [Porting an engine template](../templates/porting.md). A variant starts as a copy of the closest existing one; a template's page in the game developer's handbook goes with it, and its screenshot in `assets/templates/` and the handbook's `images/templates/`.

## Add a feature

See [Features](../templates/features.md): a plugin template with a `[feature]` table, or a directory in `rusteal-cli/project_features/` for files added to the project. Its page goes in the game developer's handbook under *Adding features*.

## Update to a new engine version

1. Build a test project with the new engine; regenerate (`build --all`).
2. Check the engine's deprecation warnings against `docs/ue-deprecations.md`; record new entries with their ids.
3. Blocklist what no longer links; fix what the exporter no longer reads.
4. Run every template; update the prerequisites in the game developer's handbook from the engine's `Windows_SDK.json`, `Linux_SDK.json`, `Mac_SDK.json` and `Apple_SDK.json` (the macOS and Xcode versions).

## Release

Nothing to do by hand but the pull request: open `develop` → `main`; Prepare release computes the version, commits it to the pull request and checks everything; merge when its statuses pass. See [Releases and versions](releases.md).
