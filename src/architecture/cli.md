# The CLI

`rusteal-cli` builds two binaries: `rusteal`, which game developers install, and `cargo-rusteal`, which makes every command a Cargo subcommand. This chapter is how the commands work inside, file by file. What they do for a game developer is in the game developer's handbook's [CLI reference](https://viniciusmorgado.github.io/game.rusteal.github.io/reference/cli.html).

## Commands (`src/main.rs`)

The command line is a clap `Parser`; `rusteal --version` is the crate's version.

| Command | Flags | Code |
|---|---|---|
| `new <Name>` | `--dir` (`.`), `--template` (`blank`), `--variant`, `--runtime-path`, `--feature` (repeatable), `--no-build`, `--optimizations on\|off`, `--engine`, `--in-place` | `new_cmd::run_new` |
| `plugin new <Name>` | `--template` (`blank`), `--project`, `--no-build`, `--editor` | `plugin_cmd::run_plugin_new` |
| `plugin editor <Name>` | `--project`, `--no-build` | `plugin_cmd::run_plugin_editor` |
| `plugin package <Name>` | `--project`, `--output` (`<root>/Packaged/Plugins`) | `package_cmd::run_plugin_package` |
| `setup <project>` | | `setup::run_setup` |
| `build [project]` | `--all`, `--step N`, `--from N`, `--release`, `--plugin` | `build_cmd::run_build` |
| `watch [project]` | `--plugin`, `--optimizations` | `watch_cmd::run_watch` |
| `generate [project]` | | `rusteal_codegen::run_generate`, then `run_generate_plugin` for each plugin |
| `package [project]` | `--output` (`<root>/Packaged`), `--shipping`, `--static`, `--platform`, `--optimizations` | `package_cmd::run_package` |
| `platform list` / `platform show <name>` | | `platform::builtins`, `platform::find` |
| `upgrade [project]` | | `upgrade_cmd::run_upgrade` |
| `launcher install` / `launcher remove` | `--engine` | `launcher_cmd::run_install`, `run_remove` |
| `sync-plugin` | | `sync_plugin::run_sync` |

- **Where a build starts** (`first_build_step`): `--from N` starts at step N; `--all` and `--step` start at 1 (`--step` runs that step only); otherwise step 4 (`build_cmd::RUST_STEP`), so a plain `rusteal build` runs the Rust steps, 4 and 5. `--all` conflicts with `--step` and `--from`, and both must be 1 to 5.
- **The project** (`project_root`) is the first directory holding a `.uproject`, walking up from the given one or the current one (`rusteal_codegen::config::find_project_root`).
- **The version check** runs before every command except `new`, `upgrade` and `sync-plugin` (see [Versions](#versions-srcproject_versionrs-srcupgrade_cmdrs)).

`src/bin/cargo-rusteal.rs` drops the `rusteal` argument cargo inserts, looks for the `rusteal` binary next to itself and then on `PATH`, and `exec`s it on Unix (spawns it and forwards the exit code elsewhere). `cargo rusteal watch` is therefore `rusteal watch`, the same command line on every platform, which is what IDE run configurations use.

## The build (`src/build_cmd.rs`)

`HostPlatform::CURRENT` is chosen with `#[cfg(target_os)]`: the host is the platform the CLI was compiled for, which runs UBT, UAT and the editor, and the editor's libraries are always built for it. What a game is packaged for is a separate **target platform** (`src/platform.rs`, below).

| Platform | UBT | UAT | Library |
|---|---|---|---|
| Linux | `Engine/Build/BatchFiles/Linux/Build.sh` | `RunUAT.sh` | `lib<name>.so` |
| Windows | `Engine/Build/BatchFiles/Build.bat` (`Win64`) | `RunUAT.bat` | `<name>.dll` |
| Mac | `Engine/Build/BatchFiles/Mac/Build.sh` | `RunUAT.sh` | `lib<name>.dylib` |

**The libraries** (`BuildContext::new`):

- the game's, unless `--plugin` is given: the workspace `<root>/Rust`, the crate and features of `rusteal.toml`'s `[project]`, deployed to `<root>/Plugins/Rusteal/Binaries/<Platform>/` with the stem `rusteal`;
- one per Rusteal plugin, every `<root>/Plugins/*` holding a `rusteal.toml` (`find_plugins`, sorted by name): the workspace `<plugin>/Rust`, the `[plugin]` crate and features, deployed to `<plugin>/Binaries/<Platform>/` with the stem `rusteal_<Plugin>`, and a second library for its `[editor]` crate when it has one (stem `rusteal_<Plugin>Editor`; an editor crate that depends on the runtime crate is refused). `--plugin` matches a name case-insensitively and selects all of that plugin's libraries.

**The five steps:**

1. **UE build.** `<engine>/<UBT> <Project>Editor <Platform> Development -Project=<uproject> -NoHotReload` (`run_ubt`; on Windows `canonical_no_prefix` strips the `\\?\` prefix). UHT runs the exporter during it; the step then copies `rusteal_classes.json`, `rusteal_structs.json` and `rusteal_enums.json` from `Plugins/RustealGenerator/Intermediate/Build/<Platform>/UnrealEditor/Inc/RustealGenerator/UHT/` to `<root>/Intermediate/Rusteal/uht/`.
2. **Codegen.** `run_generate` for the game, `run_generate_plugin` for each plugin (see [Code generation](codegen.md)).
3. **UE rebuild.** UBT again, which compiles the generated wrappers.
4. **Cargo.** It needs `<workspace>/bindings/src/func_ids.rs` (missing: "run `rusteal build --all` once"), brings the workspace up to the bindings package's name (`migrate::bindings_dependency`, also run after step 2), then runs `cargo rustc --manifest-path <workspace>/<crate>/Cargo.toml` (or the workspace's with `-p <crate>`) `--lib --crate-type cdylib`, with `--release` and `--features` as asked. The crate itself is a plain library (no `crate-type`): `--crate-type` makes the dynamic library the plugin loads, whatever the manifest says, and leaves the crate an `rlib` for the static build.
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
5. Unless `RustealFuncIds.h` exists, writes C++ stubs in `Plugins/Rusteal/Source/Rusteal/Generated/` (`RustealFuncIds.h` with `FUNC_COUNT = 0` and `FUNC_FINGERPRINT = 0`, `RustealFillFuncTable.cpp`, `module_deps.txt` with `Core`, `CoreUObject`, `Engine`), so the plugin compiles before codegen ever ran; plugins get the same with the prefix `<Plugin>_`. Generated files without `RustealGetFuncFingerprint` (an older Rusteal's) are deleted and replaced by the stubs. It also writes `Generated/static_link.txt` (`0`) when it is missing.

The project's `.gitignore` comes from the templates (`.gitignore.tera`), not from `setup`.

## Creating a project (`src/new_cmd.rs`)

1. The name: ASCII letters and digits, starting with a letter, at most 20 characters (Unreal's limit for project names). With `--in-place`, the directory must be a launcher's stub, which `clear_stub` clears (see [The launcher](launcher.md#failures-and-retries)); otherwise it must not exist.
2. The template and variant (`templates::select`), then the features: the template's `features` and the `--feature` ones, without duplicates.
3. **The engine template** (`Template::load`) is read with the engine's own recipe, `Templates/<engine_template>/Config/TemplateDefs.ini`: `FoldersToIgnore`, `FilesToIgnore`, `FolderRenames`, `FilenameReplacements`, `ReplacementsInFiles` (case-insensitive unless `bCaseSensitive`), `SharedContentPacks`. A template without `Source/` is Blueprint-only; the headers of `Source/<Template>/` named after it are the classes to redirect.
4. **Instantiation** copies the engine template, skipping the ignored paths, its `.uproject` and the manifest's `exclude`, applies the renames and, in `cpp`, `h`, `ini` and `cs` files, the replacements (`%TEMPLATENAME%`, `%PROJECTNAME%` and their cases), and copies the shared packs into `Content/<Pack>`.
5. **The `.uproject`**: `EngineAssociation` from `Engine/Build/Build.version`, modules renamed; a Blueprint-only template gets a `Modules` entry and `write_game_module` writes `Source/<Project>/` (two targets, a `Build.cs` on Core, CoreUObject, Engine, InputCore and EnhancedInput, the module files).
6. **The project ini**: a `ProjectID` when `DefaultGame.ini` has none; `ActiveGameNameRedirects` and one `ActiveClassRedirects` per template class in `DefaultEngine.ini`; the manifest's `default_map` as `EditorStartupMap` and `GameDefaultMap`.
7. **The Rusteal files** of the template, rendered with `project`, `crate_name`, `version`, `glam_version` and, with `--runtime-path` (which must hold `rusteal-runtime/Cargo.toml`), `runtime_path`.
8. The bindings placeholder crate, the platform optimizations, `setup`, the features (a plugin through `plugin_cmd::create_plugin`, files through `templates::write_project_feature_files`), then the five build steps unless `--no-build`, the stub's marker with `--in-place`, and the next steps.

## Templates and embedding (`build.rs`, `src/templates.rs`)

`build.rs` embeds everything the CLI writes into the binary, with `include_bytes!`:

| Generated file | Constant | From |
|---|---|---|
| `plugin_files.rs` | `PLUGIN_FILES` | `../ue_plugin`, or `ue_plugin_embed/` in the published crate |
| `launcher_files.rs` | `LAUNCHER_FILES` | `../ue_launcher`, or `ue_launcher_embed/` in the published crate |
| `template_files.rs` | `TEMPLATE_FILES` | `templates/` |
| `plugin_template_files.rs` | `PLUGIN_TEMPLATE_FILES` | `plugin_templates/` |
| `project_feature_files.rs` | `PROJECT_FEATURE_FILES` | `project_features/` |
| `plugin_editor_library_files.rs` | `PLUGIN_EDITOR_LIBRARY_FILES` | `plugin_editor_library/`: an editor library's files and its `rusteal.toml` section |

It skips `Generated`, `Binaries`, `Intermediate`, `obj` and `*.csproj.props`. In the repository it reads `ue_plugin/` directly; the published crate has no `ue_plugin/` next to it, so it reads the snapshot `rusteal sync-plugin` writes into `rusteal-cli/ue_plugin_embed/` (gitignored, written by the release workflows); the launcher's goes into `rusteal-cli/ue_launcher_embed/` the same way. A local snapshot can be stale without harm, since the workspace build prefers `ue_plugin/`.

`templates.rs` reads the manifests: a template is `<name>/<variant>/template.toml` (`Manifest`, `deny_unknown_fields`: `description`, `engine_template`, `exclude`, `default_map`, `features`, `next_step`), with a mandatory `base`; names match loosely (case and `-`/`_` ignored). A plugin template's `PluginManifest` may carry `[feature]` (`plugin`, `shared_packs`); a project feature has a `ProjectFeatureManifest`. `write_files` renders `{{ }}` in paths and `*.tera` files with Tera, dropping the suffix. How to add one is in [Template directories](../templates/directories.md).

## Plugins and packaging (`src/plugin_cmd.rs`, `src/package_cmd.rs`)

- **`plugin new`**: a name of 1 to 32 letters and digits, starting with a letter, not `Rusteal`, `RustealGenerator`, `RustealEditor` nor the project's; a feature carrying `Content/` keeps its fixed plugin name. It renders the template into `Plugins/<Name>/` (context `plugin`, `crate_name`, `version`, `glam_version`, and `runtime_path` when the project follows a checkout), writes the bindings placeholder and the prefixed C++ stubs, enables the plugin in the `.uproject`, copies its shared packs and builds.
- **`package`**: the five steps in release (the editor libraries, which the cook loads), a `Snapshot` of the host's `Config/<Platform>/*.ini`, the platform optimizations, `RunUAT BuildCookRun -project= -platform= -clientconfig=<Development|Shipping> -build -cook -stage -pak -archive -archivedirectory=<out> -nop4 -unattended -utf8output`, then the target platform's `uat_args` (`-package` for the built-in Mac), the snapshot restored, then UAT's exit code. On Mac with Xcode 27 or newer, UAT runs with `DEVELOPER_DIR` pointing at the `xcodebuild` workaround (`src/xcode_shim.rs`, see [Platform optimizations](../internals/platform_optimizations.md#macos-xcodebuild-without-a-stdin)).
- **`package --static`** (`src/static_link.rs`) builds the Rust code into the game instead (see [Static linking](boundary.md#static-linking)) before running UAT:
  1. the libraries: the game's and every runtime plugin's (editor plugins and editor libraries are left out), a `crate-type` with `cdylib` rewritten to `["rlib"]` (`migrate::crate_types`): Cargo would otherwise link each library as a dynamic library for the target too, with a linker the host may not have for it;
  2. the workspace in `Intermediate/Rusteal/Static/`: `Cargo.toml` (the Rusteal runtime as the project pins it, exact or from its checkout, with `static-link`; each library's crate by path, with its features; the release profile with fat LTO and one codegen unit; `debug = false` for the dev profile), the `archive` and `manifest` members, the game's `Cargo.lock` copied the first time;
  3. `cargo metadata`, for the crates each library owns (`static_link::ownership`);
  4. `cargo run -p rusteal-static-manifest` (dev profile, host), whose lines are every registration, grouped by library (`group_expected`);
  5. `archive/src/lib.rs` written with both, and `cargo rustc -p rusteal-static --lib --release -- --print=native-static-libs` (with the target platform's toolchain, `--target`, `cargo_args`, `rustflags` and `env`), whose note gives the system libraries (`parse_native_libs`), to which the platform's `system_libraries` are added;
  6. with the engine allocator (default), the `archive` member enables `rusteal-runtime`'s `engine-allocator` and its `lib.rs` sets `#[global_allocator]`; `[static] engine_allocator = false` leaves both out;
  7. the archive copied to `Plugins/Rusteal/Source/ThirdParty/RustealStatic/lib/<Platform>/` with `link.txt`, and `Generated/static_link.txt` set to `1` until UAT returns;
  8. after UAT, any Rust dynamic library found in the output is reported: a plugin whose `Build.cs` predates `Rusteal.AddRustLibrary` stages its own.
- **`plugin package`**: a release build of that plugin, then a copy of its directory without `Intermediate`, `Saved`, `DerivedDataCache`, `Rust/target`, `obj`, and the `_hot_`, `.debug` and `.pdb` files of `Binaries/`.

## Target platforms (`src/platform.rs`)

`Platform` is a packaging target, serialized as `[platform]` in TOML (`deny_unknown_fields`): `name` (the engine's platform name, passed to `BuildCookRun -platform=`), `rust_target` (a triple, or a target JSON resolved from the definition's directory; none means the host's), `toolchain` (`cargo +<toolchain>`), `build_std` (`-Zbuild-std=<crates>`), `cargo_args`, `rustflags` (`RUSTFLAGS`), `env`, `static_archive` (the archive's file name), `dynamic_library` (`prefix` and `extension`; none for a platform that only links statically), `system_libraries` (added to `link.txt`) and `uat_args`.

- `builtins()`: Linux, LinuxArm64 (`aarch64-unknown-linux-gnu`), Win64 and Mac.
- `find(name or file, dirs)`: a path ending in `.toml`, or an existing file, is loaded; otherwise a `<name>.toml` (case-insensitive) in `RUSTEAL_PLATFORM_PATH`'s directories and then `<config dir>/rusteal/platforms/`, and only then a built-in, so a definition file can replace one; an unknown name is refused, naming the built-ins and the directories searched.
- `package` resolves `--platform` (the host's name by default). Without `--static`, a target that is not the host, or has no dynamic library, is refused.

The private platform pack of a console is a definition file there, its target JSON and its toolchain: see [Consoles](../platforms/consoles.md).

## The console check (`src/console_check.rs`)

With `[consoles] check = true` in `rusteal.toml`, a build that runs step 4 lists, as warnings, each line of the libraries' workspaces that names `std::thread`, `std::time::Instant` or `std::time::SystemTime` (comments aside; `target/` and `bindings/` skipped), and each package of their `Cargo.lock` known to start threads or use the OS's I/O (`rayon-core`, `tokio`, `async-std`, `smol`, `threadpool`, `mio`). It refuses nothing.

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
| `platform.rs` | The built-in definitions round-trip through TOML, the host's names its libraries as the build does, a console definition (target JSON resolved, `build-std`), unknown fields and platforms refused, files found by name and preferred to built-ins. |
| `console_check.rs` | `std` threads and clocks found in code, not comments; threaded crates in `Cargo.lock`; off unless asked. |
| `static_link.rs` | The static workspace and its runtime pin, the engine allocator on or off, the archive's library list, ownership through local crates (shared crates refused), registrations grouped by library, rustc's system libraries for Linux, Windows and macOS, the staged library report. |
| `migrate.rs` | The bindings dependency and profile, `crate-type`, rewritten once and only there. |
| `project_version.rs`, `upgrade_cmd.rs` | Pin parsing and comparison; `rewrite_pins` touches only the pins. |
| `optimizations.rs` | The optimizations' block, user values, untouched files, the snapshot. |
| `bin/cargo-rusteal.rs` | Argument forwarding. |

`setup.rs`, `global_config.rs` and `sync_plugin.rs` have no tests: they are exercised by `rusteal new` in a test project (see [Testing a change](../contributing/testing.md)).
