# The CLI

`rusteal-cli` builds two binaries: `rusteal`, which game developers install, and `cargo-rusteal`, which makes every command a Cargo subcommand. This chapter is how the commands work inside, file by file. What they do for a game developer is in the game developer's handbook's [CLI reference](https://viniciusmorgado.github.io/game.rusteal.github.io/reference/cli.html).

## Commands (`src/main.rs`)

The command line is a clap `Parser`; `rusteal --version` is the crate's version.

| Command | Flags | Code |
|---|---|---|
| `new <Name>` | `--dir` (`.`), `--template` (`blank`), `--variant`, `--runtime-path`, `--feature` (repeatable), `--no-build`, `--optimizations on\|off` | `new_cmd::run_new` |
| `plugin new <Name>` | `--template` (`blank`), `--project`, `--no-build` | `plugin_cmd::run_plugin_new` |
| `plugin package <Name>` | `--project`, `--output` (`<root>/Packaged/Plugins`) | `package_cmd::run_plugin_package` |
| `setup <project>` | | `setup::run_setup` |
| `build [project]` | `--all`, `--step N`, `--from N`, `--release`, `--plugin` | `build_cmd::run_build` |
| `watch [project]` | `--plugin`, `--optimizations` | `watch_cmd::run_watch` |
| `generate [project]` | | `rusteal_codegen::run_generate`, then `run_generate_plugin` for each plugin |
| `package [project]` | `--output` (`<root>/Packaged`), `--shipping`, `--optimizations` | `package_cmd::run_package` |
| `upgrade [project]` | | `upgrade_cmd::run_upgrade` |
| `sync-plugin` | | `sync_plugin::run_sync` |

- **Where a build starts** (`first_build_step`): `--from N` starts at step N; `--all` and `--step` start at 1 (`--step` runs that step only); otherwise step 4 (`build_cmd::RUST_STEP`), so a plain `rusteal build` runs the Rust steps, 4 and 5. `--all` conflicts with `--step` and `--from`, and both must be 1 to 5.
- **The project** (`project_root`) is the first directory holding a `.uproject`, walking up from the given one or the current one (`rusteal_codegen::config::find_project_root`).
- **The version check** runs before every command except `new`, `upgrade` and `sync-plugin` (see [Versions](#versions-srcproject_versionrs-srcupgrade_cmdrs)).

`src/bin/cargo-rusteal.rs` drops the `rusteal` argument cargo inserts, looks for the `rusteal` binary next to itself and then on `PATH`, and `exec`s it on Unix (spawns it and forwards the exit code elsewhere). `cargo rusteal watch` is therefore `rusteal watch`, the same command line on every platform, which is what IDE run configurations use.

## The build (`src/build_cmd.rs`)

`HostPlatform::CURRENT` is chosen with `#[cfg(target_os)]`: the CLI only builds for the platform it was compiled for.

| Platform | UBT | UAT | Library |
|---|---|---|---|
| Linux | `Engine/Build/BatchFiles/Linux/Build.sh` | `RunUAT.sh` | `lib<name>.so` |
| Windows | `Engine/Build/BatchFiles/Build.bat` (`Win64`) | `RunUAT.bat` | `<name>.dll` |
| Mac | `Engine/Build/BatchFiles/Mac/Build.sh` | `RunUAT.sh` | `lib<name>.dylib` |

**The libraries** (`BuildContext::new`):

- the game's, unless `--plugin` is given: the workspace `<root>/Rust`, the crate and features of `rusteal.toml`'s `[project]`, deployed to `<root>/Plugins/Rusteal/Binaries/<Platform>/` with the stem `rusteal`;
- one per Rusteal plugin, every `<root>/Plugins/*` holding a `rusteal.toml` (`find_plugins`, sorted by name): the workspace `<plugin>/Rust`, the `[plugin]` crate and features, deployed to `<plugin>/Binaries/<Platform>/` with the stem `rusteal_<Plugin>`. `--plugin` matches a name case-insensitively.

**The five steps:**

1. **UE build.** `<engine>/<UBT> <Project>Editor <Platform> Development -Project=<uproject> -NoHotReload` (`run_ubt`; on Windows `canonical_no_prefix` strips the `\\?\` prefix). UHT runs the exporter during it; the step then copies `rusteal_classes.json`, `rusteal_structs.json` and `rusteal_enums.json` from `Plugins/RustealGenerator/Intermediate/Build/<Platform>/UnrealEditor/Inc/RustealGenerator/UHT/` to `<root>/Intermediate/Rusteal/uht/`.
2. **Codegen.** `run_generate` for the game, `run_generate_plugin` for each plugin (see [Code generation](codegen.md)).
3. **UE rebuild.** UBT again, which compiles the generated wrappers.
4. **Cargo.** It needs `<workspace>/bindings/src/func_ids.rs` (missing: "run `rusteal build --all` once"), then runs `cargo build --manifest-path <workspace>/<crate>/Cargo.toml` (or the workspace's with `-p <crate>`), with `--release` and `--features` as asked.
5. **Deploy.** The library in `target/<debug|release>/` is copied to `<dest>.partial` and renamed over `librusteal.so` (`librusteal_<Plugin>.so`, `.dylib` on macOS). The deployed file is replaced, a new inode, never written into: on Linux the plugin hard-links it into its load copy (see [Hot reload](hot_reload.md)), and the editor's directory watcher must never see a half-written library.

`-NoHotReload` keeps UBT from producing hot-reload module names, so a build with the editor open does not fight the editor's own modules. The CLI does not check whether the editor is running: steps 1 to 3 rebuild C++ the open editor has loaded, which is why changing the plugin or the bindings needs an editor restart, and steps 4 and 5 are made to run with it open.

The profiles come from the template's `Rust/Cargo.toml` (see the game developer's handbook, [Development and release builds](https://viniciusmorgado.github.io/game.rusteal.github.io/cli/profiles.html)): `dev` is `opt-level = 1` for the game crate, `3` with `line-tables-only` debug info for the dependencies and the bindings (full debug info makes the library pass 100 MB, reloaded on every change); `release` is `codegen-units = 1` and thin LTO.

`run_cmd` echoes the command line (cut at 200 characters) and exits with the child's code on failure; `run_cmd_code` returns the code instead, for callers that must clean up first (`package`).

## Watching (`src/watch_cmd.rs`)

Polling, so it needs no dependency and behaves the same on every platform: every 400 ms (`POLL`) it snapshots `(mtime, size)` of the `.rs` and `.toml` files of the game's `Rust/` and each plugin's (`--plugin` filters), skipping `target`, `bindings` and directories starting with `.`. On a change it re-snapshots every 300 ms (`SETTLE`) until two agree, prints the changed files (up to three), and runs `<itself> build <root> [--plugin P]` as a child process. A failed build is logged and the loop goes on; it never returns. Changes to `rusteal.toml` or the engine need `rusteal build --all`, which `watch` does not run. It writes the [platform optimizations](../internals/platform_optimizations.md) once, when it starts.

## Setting up a project (`src/setup.rs`)

1. Writes `PLUGIN_FILES` (the embedded plugins) under `<project>/Plugins/`, skipping files already identical.
2. Writes `Plugins/RustealGenerator/Source/RustealExporter/RustealExporter.ubtplugin.csproj.props` with `<EngineDir>`, so the exporter's C# project finds the engine.
3. Without a `rusteal.toml`, renders the blank template's (`templates::render_rusteal_toml`), with the crate name `crate_name_for` derives from the project (`MyGame` → `my-game`).
4. Adds `Rusteal` and `RustealGenerator` to the `.uproject`'s `Plugins`.
5. Unless `RustealFuncIds.h` exists, writes C++ stubs in `Plugins/Rusteal/Source/Rusteal/Generated/` (`RustealFuncIds.h` with `FUNC_COUNT = 0`, `RustealFillFuncTable.cpp`, `module_deps.txt` with `Core`, `CoreUObject`, `Engine`), so the plugin compiles before codegen ever ran; plugins get the same with the prefix `<Plugin>_`.

The project's `.gitignore` comes from the templates (`.gitignore.tera`), not from `setup`.

## Creating a project (`src/new_cmd.rs`)

1. The name: ASCII letters and digits, starting with a letter, at most 20 characters (Unreal's limit for project names).
2. The template and variant (`templates::select`), then the features: the template's `features` and the `--feature` ones, without duplicates.
3. **The engine template** (`Template::load`) is read with the engine's own recipe, `Templates/<engine_template>/Config/TemplateDefs.ini`: `FoldersToIgnore`, `FilesToIgnore`, `FolderRenames`, `FilenameReplacements`, `ReplacementsInFiles` (case-insensitive unless `bCaseSensitive`), `SharedContentPacks`. A template without `Source/` is Blueprint-only; the headers of `Source/<Template>/` named after it are the classes to redirect.
4. **Instantiation** copies the engine template, skipping the ignored paths, its `.uproject` and the manifest's `exclude`, applies the renames and, in `cpp`, `h`, `ini` and `cs` files, the replacements (`%TEMPLATENAME%`, `%PROJECTNAME%` and their cases), and copies the shared packs into `Content/<Pack>`.
5. **The `.uproject`**: `EngineAssociation` from `Engine/Build/Build.version`, modules renamed; a Blueprint-only template gets a `Modules` entry and `write_game_module` writes `Source/<Project>/` (two targets, a `Build.cs` on Core, CoreUObject, Engine, InputCore and EnhancedInput, the module files).
6. **The project ini**: a `ProjectID` when `DefaultGame.ini` has none; `ActiveGameNameRedirects` and one `ActiveClassRedirects` per template class in `DefaultEngine.ini`; the manifest's `default_map` as `EditorStartupMap` and `GameDefaultMap`.
7. **The Rusteal files** of the template, rendered with `project`, `crate_name`, `version`, `glam_version` and, with `--runtime-path` (which must hold `rusteal-runtime/Cargo.toml`), `runtime_path`.
8. The bindings placeholder crate, the platform optimizations, `setup`, the features (a plugin through `plugin_cmd::create_plugin`, files through `templates::write_project_feature_files`), then the five build steps unless `--no-build`, and the next steps.

## Templates and embedding (`build.rs`, `src/templates.rs`)

`build.rs` embeds everything the CLI writes into the binary, with `include_bytes!`:

| Generated file | Constant | From |
|---|---|---|
| `plugin_files.rs` | `PLUGIN_FILES` | `../ue_plugin`, or `ue_plugin_embed/` in the published crate |
| `template_files.rs` | `TEMPLATE_FILES` | `templates/` |
| `plugin_template_files.rs` | `PLUGIN_TEMPLATE_FILES` | `plugin_templates/` |
| `project_feature_files.rs` | `PROJECT_FEATURE_FILES` | `project_features/` |

It skips `Generated`, `Binaries`, `Intermediate`, `obj` and `*.csproj.props`. In the repository it reads `ue_plugin/` directly; the published crate has no `ue_plugin/` next to it, so it reads the snapshot `rusteal sync-plugin` writes into `rusteal-cli/ue_plugin_embed/` (gitignored, written by the release workflows). A local snapshot can be stale without harm, since the workspace build prefers `ue_plugin/`.

`templates.rs` reads the manifests: a template is `<name>/<variant>/template.toml` (`Manifest`, `deny_unknown_fields`: `description`, `engine_template`, `exclude`, `default_map`, `features`, `next_step`), with a mandatory `base`; names match loosely (case and `-`/`_` ignored). A plugin template's `PluginManifest` may carry `[feature]` (`plugin`, `shared_packs`); a project feature has a `ProjectFeatureManifest`. `write_files` renders `{{ }}` in paths and `*.tera` files with Tera, dropping the suffix. How to add one is in [Template directories](../templates/directories.md).

## Plugins and packaging (`src/plugin_cmd.rs`, `src/package_cmd.rs`)

- **`plugin new`**: a name of 1 to 32 letters and digits, starting with a letter, not `Rusteal`, `RustealGenerator`, `RustealEditor` nor the project's; a feature carrying `Content/` keeps its fixed plugin name. It renders the template into `Plugins/<Name>/` (context `plugin`, `crate_name`, `version`, `glam_version`, and `runtime_path` when the project follows a checkout), writes the bindings placeholder and the prefixed C++ stubs, enables the plugin in the `.uproject`, copies its shared packs and builds.
- **`package`**: the five steps in release, a `Snapshot` of the host's `Config/<Platform>/*.ini`, the platform optimizations, `RunUAT BuildCookRun -project= -platform= -clientconfig=<Development|Shipping> -build -cook -stage -pak -archive -archivedirectory=<out> -nop4 -unattended -utf8output` (and `-package` on Mac), the snapshot restored, then UAT's exit code. On Mac with Xcode 27 or newer, UAT runs with `DEVELOPER_DIR` pointing at the `xcodebuild` workaround (`src/xcode_shim.rs`, see [Platform optimizations](../internals/platform_optimizations.md#macos-xcodebuild-without-a-stdin)).
- **`plugin package`**: a release build of that plugin, then a copy of its directory without `Intermediate`, `Saved`, `DerivedDataCache`, `Rust/target`, `obj`, and the `_hot_`, `.debug` and `.pdb` files of `Binaries/`.

## Versions (`src/project_version.rs`, `src/upgrade_cmd.rs`)

A project's pins are `rusteal-runtime`, `rusteal-core` and `rusteal-ffi` in `Rust/Cargo.toml`'s `[workspace.dependencies]`, all three either exact (`"=X.Y.Z"`, `Pins::Exact`) or paths into a checkout (`Pins::Checkout`, relative paths resolved from `<root>/Rust`), and in agreement. `compare` gives `Matches`, `ProjectOlder`, `ProjectNewer` or `OtherCheckout`: a checkout project matches only the CLI built from that same checkout (`CARGO_MANIFEST_DIR/..`). `check` looks at the project and every plugin; with `Scope::PinsAndPlugins` it also wants `VersionName` of both `.uplugin`s to be the CLI's. Every refusal names its fix (`rusteal upgrade`, `cargo install rusteal@X`, `cargo run --manifest-path ...`, `rusteal setup`).

`upgrade` refuses checkout projects and newer projects, then rewrites only the three pin lines in the game's and every plugin's workspace to `"=<CLI version>"`, deletes `Plugins/Rusteal` and `Plugins/RustealGenerator`, runs `setup` and the full build.

## The engine path (`src/global_config.rs`)

`<config dir>/rusteal/config.toml` (`~/.config/rusteal/config.toml` on Linux, `~/Library/Application Support/rusteal/config.toml` on macOS, `%APPDATA%\rusteal\config.toml` on Windows: the `dirs` crate's config directory) holds `[engine] path`. When it is empty, the CLI asks once for the engine root, until the directory has `Engine/Build/BatchFiles`, and saves it; without a terminal it exits with the setting to write. This is the only per-machine setting a game project depends on; contributors have `.env` for theirs, which the CLI never reads.

## Tests

| File | What it protects |
|---|---|
| `tests/plugin_version.rs` | Both `.uplugin`s carry the workspace version (`VersionName`) and its integer form (`Version`), which the runtime handshake and the version check rely on. |
| `main.rs` | The clap definition, the version, the default build steps, `--all`'s conflicts, repeated `--feature`, the `--optimizations` switch. |
| `build_cmd.rs` | The atomic deploy leaves no `.partial`. |
| `watch_cmd.rs` | Ignored paths, detected changes, the change description. |
| `templates.rs` | The template list, valid manifests, bundled features that exist, the default optimizations in every `rusteal.toml.tera`, the feature invariants, the RustRover files, loose names, rendered paths. |
| `new_cmd.rs` | Writable copies, the game module of Blueprint-only templates, the default map. |
| `package_cmd.rs` | The plugin package exclusions. |
| `project_version.rs`, `upgrade_cmd.rs` | Pin parsing and comparison; `rewrite_pins` touches only the pins. |
| `optimizations.rs` | The optimizations' block, user values, untouched files, the snapshot. |
| `bin/cargo-rusteal.rs` | Argument forwarding. |

`setup.rs`, `global_config.rs` and `sync_plugin.rs` have no tests: they are exercised by `rusteal new` in a test project (see [Testing a change](../contributing/testing.md)).
