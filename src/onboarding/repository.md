# The repository

Everything Rusteal ships comes from one repository, [viniciusmorgado/rusteal](https://github.com/viniciusmorgado/rusteal). This chapter is its map: what each directory is, which branches exist and how a change travels from a branch to a release.

## Top level

| Path | What it is | Versioned |
|---|---|---|
| `Cargo.toml` | The Cargo workspace: its members, the shared `[workspace.package]` (version, edition, license) and the dependencies every crate takes from `[workspace.dependencies]`, the internal ones with both a `version` and a `path` (see [Releases and versions](../contributing/releases.md)). | yes |
| `rust-toolchain.toml` | Pins Rust 1.88.0, the MSRV, with `clippy` and `rustfmt`, so the checkout cannot use newer Rust by accident. | yes |
| `rusteal-ue-flags/` | The engine's reflection flags as Rust constants. | yes |
| `rusteal-ffi/` | The `#[repr(C)]` contract between Rust and the C++ plugin. | yes |
| `rusteal-core/` | The safe runtime over that contract. | yes |
| `rusteal-macros/` | `#[uclass]`, `#[uclass_impl]`, `#[ustruct]`, `#[uenum]`, `#[udelegate]`. | yes |
| `rusteal-runtime/` | What a game crate depends on: re-exports, the prelude, `entry!()`. | yes |
| `rusteal-codegen/` | Reflection JSON to bindings crate and C++ wrappers. | yes |
| `rusteal-cli/` | The `rusteal` binary (and `cargo-rusteal`), with the game templates (`templates/`), the plugin templates and gameplay features (`plugin_templates/`), the project-file features (`project_features/`) and the default platform optimizations (`optimizations.toml`). | yes |
| `ue_plugin/Rusteal/` | The runtime UE plugin: the `Rusteal` module (the host of the Rust libraries) and the `RustealEditor` module. | yes |
| `ue_plugin/RustealGenerator/` | The build-time UE plugin: the UHT exporter (C#) and its anchor module. | yes |
| `ide/RustealRules/` | A C# project for language servers only, giving the plugins' `*.Build.cs` files their context. | yes |
| `rusteal.sln` | The .NET solution holding the exporter and `RustealRules`, for C# language servers. | yes |
| `xtask/` | Contributor tooling: `cargo xtask dev-setup`. Never published. | yes |
| `docs/` | Tracking documents: deprecations, the third person gaps, how the variants were ported, the plugin roadmap, hot reload notes (see [Tracking documents and conventions](../contributing/tracking.md)). | yes |
| `assets/` | The logo, the mascot and the template screenshots used by the README. | yes |
| `.github/workflows/` | CI and the release pipeline (see [Releases and versions](../contributing/releases.md)). | yes |
| `.cargo/config.toml` | The `cargo xtask` alias. | yes |
| `.clang-format` | LLVM style for the C++ plugin, with `SortIncludes: Never`: UHT requires a header's `.generated.h` to be its last include, which sorting would break. | yes |
| `.env.example` | The contributor settings to copy into `.env`: `RUSTEAL_DEV_ENGINE_ROOT`, the engine root. | yes |
| `.env`, `compile_commands.json`, `.cache/` | Written by `cargo xtask dev-setup` for this machine. | no |
| `Cargo.lock` | Rewritten by every build because of the internal versions; cargo packs its own into the published crates. | no |
| `rusteal-cli/ue_plugin_embed/` | The plugin snapshot the published CLI embeds, written by `rusteal sync-plugin`. | no |
| `rusteal-core/api_table.rs` | A copy of the FFI table definition for the published crate, written by `rusteal-core/build.rs`. | no |
| `ue_plugin/Rusteal/Source/Rusteal/Generated/` | Codegen output, which only exists inside a game project. | no |

## Branches

| Branch | What it is |
|---|---|
| `develop` | The default branch, where work is integrated. Every change reaches it through a pull request, with CI on it. |
| `main` | What is released. It only receives the release pull request from `develop`, protected by the "main: release checks" ruleset (the `build` and `bump` statuses, no bypass); each merge publishes a version. |
| `feat/...`, `fix/...`, `docs/...`, `chore/...` | Work branches, made from an up-to-date `develop` and merged back into it by pull request. Their names follow the Conventional Commits type of the work. |
| `ue6` | The adaptation to Unreal Engine 6, apart from `main` until UE6 ships. Its `docs/ue6-radar.md` tracks what UE6 changes for Rusteal. |

A change goes: a work branch from `develop` → commits with Conventional Commits subjects → a pull request to `develop` (CI) → merge → the release pull request `develop` → `main` (Prepare release) → merge → Release publishes and tags → Sync develop brings the version commit back to `develop`. Before opening a pull request, bring the branch up to date with `develop`: a branch made before a release still shows the previous version until then.

## The handbooks

The two handbooks are mdBooks in repositories of their own, published with GitHub Pages:

| Handbook | Repository | Site |
|---|---|---|
| The game developer's handbook: using Rusteal | [game.rusteal.github.io](https://github.com/viniciusmorgado/game.rusteal.github.io) | [viniciusmorgado.github.io/game.rusteal.github.io](https://viniciusmorgado.github.io/game.rusteal.github.io/) |
| This handbook: working on Rusteal | [dev.rusteal.github.io](https://github.com/viniciusmorgado/dev.rusteal.github.io) | [viniciusmorgado.github.io/dev.rusteal.github.io](https://viniciusmorgado.github.io/dev.rusteal.github.io/) |

A pull request that changes what a game developer sees (a command, an attribute, a template) updates the game developer's handbook in the same round; one that changes how Rusteal works inside, or decides something, updates this one. The README only presents the project and points to both.
