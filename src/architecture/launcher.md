# The launcher

The launcher (`ue_launcher/RustealLauncher`) is an engine plugin that puts Rusteal into Unreal's **New Project** dialog: a **Rust** category with one template per Rusteal template and variant. It is how Rusteal is meant to reach Fab, and how a game developer creates a project without the terminal. How it is used is in the game developer's handbook, [Creating a project from the editor](https://viniciusmorgado.github.io/game.rusteal.github.io/getting_started/from_the_editor.html).

## Why a plugin of its own

The Rusteal plugin (`ue_plugin/Rusteal`) cannot be the one installed into the engine: it lives in each project, and `rusteal setup` writes that project's generated C++ wrappers into it (`Source/Rusteal/Generated`). The launcher is a small editor-only plugin, C++ only, that ships no Rust: Fab builds plugins itself, with no `cargo`, and a project's Rust comes from the CLI anyway. It is kept out of `ue_plugin/`, because everything under `ue_plugin/` is copied into every project (decision A17).

## What the engine allows

From the UE 5.8 sources (`Engine/Source/Editor/GameProjectGeneration`):

- **No third language.** The BLUEPRINT/C++ choice is a segmented control with two fixed slots (`SProjectDialog.cpp`); nothing can add a third.
- **Plugins can bring templates and categories.** The dialog lists every folder of a discovered plugin's `Templates/` that holds one `.uproject` and a `Config/TemplateDefs.ini`, and reads `Templates/TemplateCategories.ini` from the same place, whether the plugin is enabled or not (a plugin that is not enabled by default gets a warning, which does not block **Create**).
- **No code runs in the dialog.** `TemplateDefs.ini` can name a `TemplateProjectDefsClass` with `PreGenerateProject` and `PostGenerateProject` hooks, but only a loaded module provides it, and a plugin from Fab is not loaded: `BuildPlugin` removes `EnabledByDefault` from the descriptor it packages, so an engine plugin is off until a project enables it.
- **The variant is not recorded.** The variant picked in the dialog only selects shared content packs; nothing in the created project says which one it was.

## How a project is made

1. A Rust template is a **stub**: a `.uproject` with no modules that enables `RustealLauncher`, two nearly empty ini files (the engine writes the project ID and the renderer settings only into ini files the template has), and `RustealLauncher.json`: `{"stub": true, "template": "...", "variant": "..."}`. The marker is JSON because the engine's text replacements (`%TEMPLATENAME%`) apply to `cpp`, `h`, `ini` and `cs` files only.
2. **Create** copies the stub and opens it in a new editor process, where the launcher is enabled. The launcher's module (`RustealLauncherModule.cpp`) reads the marker when the main frame exists and opens the **Creating the Rust project** window (`SRustealCreateProject`).
3. The window checks `cargo` and the CLI (`RustealLauncherCli.cpp`): `RUSTEAL_CLI` first, then `PATH`, then Cargo's `bin` (`CARGO_HOME`, else `~/.cargo`). A CLI is new enough when `rusteal new --help` lists `--in-place` and `--engine`, a capability check rather than a version number, so the launcher does not need a release of its own for each CLI.
4. When both are there and the project name is one the CLI accepts, it starts at once: `rusteal new <Name> --dir <parent> --template <t> [--variant <v>] --engine <engine root> --in-place`, with `cargo`'s directory added to `PATH`. Its output, standard error included, goes through a pipe read on a ticker (`RustealLauncherProcess`): `FMonitoredProcess` pipes standard output only, and the CLI writes to standard error. Each line goes to the window and to `LogRustealLauncher`.
5. On success the editor restarts (`FUnrealEdMisc::RestartEditor`) into the converted project, whose `.uproject` no longer lists the launcher.

Every variant is a template of its own (`TP_Rust<Template><Variant>`), since the dialog's variant would be lost; `every_rust_template_names_a_template_the_cli_has` (`rusteal-cli/src/launcher_cmd.rs`) checks that the stubs and the CLI's templates match one to one (decision A18).

## Failures and retries

`rusteal new --in-place` (`new_cmd::clear_stub`) accepts a directory only with a marker that says `stub`, keeps `Saved/`, `Intermediate/` and `DerivedDataCache/` (the open editor holds files there), removes the stub's files and anything a conversion makes (`Config`, `Content`, `Source`, `Rust`, `Plugins`, `Binaries`, `rusteal.toml`, `.gitignore`, the `.uproject`), and refuses anything else. The marker is removed only after the build succeeds.

When the setup fails, is canceled, or the editor closes in the middle, the launcher writes the stub's `.uproject` back and deletes the `Source/`, `Plugins/` and `Binaries/` the stub did not have: the engine would otherwise ask to build the project's modules, or enable the half-installed Rusteal plugin, before the launcher could load. Opening the project again starts over (decision E14).

## Building and installing it

- **Fab** builds the plugin from its sources with `BuildPlugin`. The local equivalent is `RunUAT.sh BuildPlugin -Plugin=<repo>/ue_launcher/RustealLauncher/RustealLauncher.uplugin -Package=<dir> -Rocket -TargetPlatforms=<host>`, which must build with no warnings. `Config/FilterPlugin.ini` adds `/Config/...` and `/Templates/...` to the package; without it they are dropped.
- **`rusteal launcher install`** (`src/launcher_cmd.rs`) installs it without Fab, for Linux (whose editor cannot install Fab plugins) and checkouts: the launcher's sources are embedded in the CLI (`LAUNCHER_FILES`, from `ue_launcher/` or `ue_launcher_embed/` in the published crate, which `rusteal sync-plugin` writes), written to a temporary directory and built with the engine's `BuildPlugin` (`-Rocket` on an installed engine) into `Engine/Plugins/Marketplace/RustealLauncher`. `rusteal launcher remove` deletes it (decision E15).

## Versions

The launcher has its own version (`0.1.0`), outside the release bump: a Rusteal release does not force a Fab submission, which goes through review each time. It works with any CLI that passes the capability check (decision H11). Its `.cpp` and `.h` files carry a one-line copyright header, which Fab requires, the one exception to the no-comments rule (decision I8).

## Testing it

No editor window is needed for most of it:

- the CLI side: `cargo test -p rusteal` (`clear_stub`, the embedded files, the stubs against the templates);
- the package: `BuildPlugin` as above;
- the window's states: open a copy of a stub with `RUSTEAL_CLI` set to a missing file, to a script that prints an old version, or to one that fails; the launcher logs what it found (`Rust: ...; Rusteal: ...`) and the exit code.

A full run, from **Create** to the restarted editor with `Rust side initialized` in its log, opens an editor on the desktop for several minutes and builds with every core. `Rusteal.Launcher.Create` starts the setup from the console, for a run with `-ExecCmds`. A script driving editors must wait for each one to exit and kill it after a timeout, so that two never run at once.

## Open

- Not yet run on Windows and macOS.
- Fab: the listing, and the questions for Fab support, are in the repository's `docs/fab-submission.md`.
