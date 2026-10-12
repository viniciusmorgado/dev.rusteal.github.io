# Decisions

Why Rusteal is the way it is. Each record gives the decision, the reasons, what was rejected when that is known, what it implies, and where it was made (a commit of the repository, or a file). The sources carry no rationale comments, so this chapter, the commit messages and `docs/` are where the reasons live: **a decision that changes how Rusteal works is added here in the same pull request** (see [Recording a decision](#recording-a-decision)).

## Architecture

### A1. Bindings come from the engine's reflection
- **Decision.** A UHT exporter writes the engine's reflection to JSON; `rusteal-codegen` generates the bindings crate and the C++ wrappers from it. Nothing is bound by hand except `manual/`.
- **Why.** Whatever Blueprints reach, Rust reaches, for the installed engine version and the project's plugins, with no binding to maintain. It also bets well on UE6: Verse is built on the same reflection and UHT.
- **Implies.** What UHT never sees cannot be generated: Slate, C++ templates (`BindAction`), delegates outside reflection. They need `manual/`, native shims, or are out of reach (see `docs/plugin-roadmap.md`).
- **Source.** The design inherited from Uika; `docs/plugin-roadmap.md`.

### A2. Every engine call crosses a function pointer table; Rust links no C++
- **Decision.** The library calls the engine only through `RustealApiTable` and the generated function table; `rusteal-core/build.rs` generates the dispatch from `rusteal-ffi/src/api_table.rs`.
- **Why.** Iterating on Rust needs no C++ build; the library can be swapped at run time, since pointing functions at a new library is the whole code swap.
- **Implies.** The table's layout is the ABI, guarded by exact version equality (C1). `rusteal-core` ships a copy of `api_table.rs`, because a published crate cannot read `../rusteal-ffi` (`4867c56`).

### A3. Generated wrappers first, reflection as the fallback
- **Decision.** Generated functions call direct C++ wrappers; `DynamicCall` goes through reflection for functions defined in Blueprints and anything not bound.
- **Why.** The direct path is fast and covers almost every call; reflection covers what codegen cannot see.
- **Implies.** The wrappers must compile and link: functions in non-public headers are skipped (`a650546`), functions their module does not export are blocklisted (`f90d0f9`, `01ddef7`).

### A4. One library for the game, one per Rusteal plugin
- **Decision.** The game's code is a `cdylib` the Rusteal plugin loads. A Rusteal plugin has its own library, workspace, bindings and C++ module, which registers the library (`RustealRegisterPluginLibrary`); its C++ symbols are prefixed `<Plugin>_` (`8e66cd7`, `5c1ac09`).
- **Why.** Gameplay systems become reusable across projects. Classes, functions and pins remember the library that made them, and every call runs with that library current.
- **Implies.** A plugin is runtime or editor, not both (two plugins otherwise); a Rust class cannot derive from another library's Rust class.

### A5. A plugin's classes live in `/Script/<Plugin>`
- **Why.** Plugins from different authors cannot clash on a class name, and the engine sees them by their module (`0a7bbae`).

### A6. Rust classes are created at load time, and Blueprints derive from them
- **Decision.** `#[uclass]` structs become `URustealReifiedClass`es (Blueprint-generated classes) with a stub Blueprint `RS_<Class>`; a Blueprint child holds the assets, as the engine's C++ templates do.
- **Why.** It is the engine's own pattern for C++ templates, and it needed no new engine feature (`4118180`).
- **Implies.** Editor Python sees Rust classes by Unreal names; Rust classes had to be registered for cooked loading (`6b0727d`); classes are created first and given members after, parents first.

### A7. UE6 work stays on the `ue6` branch
- **Why.** UE6 changes the foundations (Verse, Scene Graph, Blueprints deprecated); the adaptation is tracked in `docs/ue6-radar.md` there and stays off `main` until UE6 ships (`1b0f730`).

### A8. A hard fork, independent from Uika
- **Decision.** Everything was renamed (`1c82d12`); Rusteal develops on its own, unless Uika's author wants to port improvements back.

### A9. No WASM hosting
- **Decision.** Uika's wasmtime embedding (`65f606e`) was removed before the fork (`d5841ff`), with no reason recorded. Rusteal has no WASM support; do not assume it.

### A10. Consoles: a static library per target, no engine fork, private platform packs
- **Decision.** Rust reaches a console build as a `staticlib` per target, consumed by an external module and built by Rusteal's CLI before UBT; the default route links one aggregated archive into the game's executable. The repository keeps the platform-agnostic structure; each console's target, flags, build rules and CI live in a private platform pack. The runtime takes its system services from the engine.
- **Why.** The engine's documented external-library mechanism links a prebuilt library on every platform, so no engine fork is needed; the console code cannot be public; and the engine is already ported to each console, unlike Rust's standard library.
- **Rejected.** Patching UBT to compile Rust (kept for a limitation measured with the SDK); loose objects or LLVM bitcode as the artifact (dependencies missing, LLVM versions tied); one self-contained static library per Rusteal library (duplicate runtimes and symbols).
- **Implies.** The module contract and the static backend are built and tested on desktop first. See [Consoles](../platforms/consoles.md).

### A11. The plugin names the Rust libraries itself
- **Decision.** `RustealLibraryFileName` builds `lib<stem>.<ext>` on every platform but Windows, `<stem>.dll` there: the names `rusteal build` deploys and `Rusteal.Build.cs` stages (`38541c4`).
- **Why.** `FPlatformProcess::GetModulePrefix()`, which it used before, is empty on Mac (only the Unix platform returns `lib`): the plugin looked for `rusteal.dylib`, the Rust side never loaded, and every Blueprint derived from a Rust class failed to load.

### A12. macOS is supported on Apple silicon only
- **Decision.** The supported platforms are Linux x64, Windows x64 and macOS on Apple silicon.
- **Why.** Apple silicon is what Rusteal was tested on; macOS on Intel processors has not been, and is not claimed.

## Runtime

### B1. Panics are caught at the boundary
- **Decision.** Every entry point runs in `ffi_boundary` (`catch_unwind`); a chained panic hook records the location; the panic is reported as a script error and the call returns a default.
- **Why.** Unwinding into C++ is undefined behaviour, and a panic in game code must not take the editor down.
- **Implies.** A panicking function returns its default; spawned tasks and tick hooks are guarded the same way.

### B2. Errors are modelled on `std::io::Error`
- **Decision.** `RustealError` is a boxed struct matched by `kind()`, with the object, the engine's reason, the caller's location, contexts and a source; fallible APIs are `#[track_caller]`.
- **Why.** An error must say what failed, on what, why Unreal refused and where in the user's code, and work with `?` and other error types.
- **Rejected.** The earlier enum of variants (a breaking change, `feat(core)!`: variants became constructors).
- **Implies.** A `#[ufunction]` may return `RustealResult<T>`, raising a Blueprint runtime error on `Err`; the C++ side keeps a per-thread reason (`b7fc0c7`, `f44d529`, `1f8be32`, `336bc90`). See [Errors](../architecture/errors.md).

### B3. Rust `bool`s are native bools
- **Why.** `FBoolProperty`'s default is a one-bit field; a `false` written by Python or `ProcessEvent` cleared only that bit, and Rust, reading the byte, saw `true` (`fe31dd3`).

### B4. `FName`s are packed and unpacked, never copied raw
- **Why.** In editor builds an `FName` carries a display index; raw copies showed `None` and could save it (`3c77034`, `eb43e93`).

### B5. `FKey` is an opaque reflected struct
- **Why.** The engine's `FKey` is 24 bytes or more; an 8-byte newtype crashed `TArray` operations (`f4a4b6b`).

### B6. Inheritance through `Deref`, upcasts checked at compile time
- **Decision.** Codegen emits each class's own members; inherited ones resolve through the parent. `Inherits<U>` checks `upcast_to` at compile time (`51fdb28`).
- **Rejected.** Copying every inherited member into each class (`88efc84`).

### B7. Two tiers of object lifetime
- **Decision.** `UObjectRef<T>` is a `Copy` handle with no ownership; `Pinned<T>` is a GC root.
- **Why.** No global registry or reference counting on the common path.
- **Implies.** Code calls `checked()` before using a handle; pins are released when their library unloads (`75dc959`).

### B8. Background work returns to the game thread; unloading waits for it
- **Why.** No thread may run code that is gone (`0f5f3c3`).

## Versions

### C1. One version for everything
- **Decision.** The seven crates, the two `.uplugin`s and the FFI table share `[workspace.package].version`; internal dependencies are `{ version, path }`.
- **Why.** One number to bump; it resolves locally in the workspace and from crates.io once published.
- **Implies.** Projects pin `"=x.y.z"`; the plugin refuses a library of another version; the CLI checks the project before working on it; plugin libraries must match exactly (tolerating minor differences would need a stable table layout); `tests/plugin_version.rs` fails CI when a descriptor drifts.

### C2. The integer form is derived, never a source
- **Decision.** `encode_version` = major·1 000 000 + minor·1 000 + patch, for the `.uplugin`'s integer `Version` and the table's `u32`.
- **Implies.** Minor and patch stay below 1000; the `const fn` refuses anything else at compile time.

### C3. Projects move forward whole
- **Decision.** `rusteal upgrade` rewrites the pins and replaces both plugins wholesale; mixed versions never run (`d26cb48`).

### C4. A project's glam follows Rusteal's
- **Decision.** New projects get the CLI's `GLAM_VERSION` (`rusteal-cli/src/new_cmd.rs`), and `rusteal upgrade` rewrites the project's `glam` to it next to the pins (`3461ef7`).
- **Why.** The generated bindings crate takes `glam` from the project's workspace while `rusteal-core` takes it from Rusteal's; with two versions they are two crates whose types do not mix, and the bindings stop compiling.
- **Implies.** Bumping `glam` in the workspace means bumping `GLAM_VERSION` with it. Projects that follow a checkout (`--runtime-path`) are not upgraded: change their `glam` by hand.

## Code generation

### D1. The bindings are generated into each game project and committed there
- **Why.** They depend on the project's engine version and modules; the Rusteal workspace builds, tests and lints without Unreal (`70add11`).
- **Implies.** Regeneration must be deterministic: module imports are sorted so a regeneration does not reshuffle files (`ca56d55`).

### D2. Generated files are written only when they change
- **Why.** Rewriting every file made UBT recompile every wrapper and Cargo rebuild the bindings; an unchanged full build went from minutes to seconds (`d8dd518`). `setup` keeps unchanged plugin files for the same reason (`601d8d4`).
- **Rejected.** Wiping the output first (`cbb0ca5`).

### D3. Clippy is off in generated code
- **Why.** Hundreds of generated functions made `cargo clippy` fail on every game crate (`7d8b9fb`).

### D4. Modules are bound selectively
- **Decision.** `classes = [...]` limits a large module (UnrealEd's 354 classes); editor-only functions are bound only in editor libraries; Rusteal's own modules are always bound (`1f23a9d`, `4b043e1`, `29a6986`).

### D5. UHT's gaps are fixed in the exporter, not blocklisted
- **Decision.** Enum values UHT leaves at -1 are evaluated, third-party constants looked up, structs reached by bound properties exported, protected members a Blueprint can reach exported (`b36344e`, `b416dfb`, `33d575c`, `f6496e6`, `b8d8159`).

## CLI

### E1. The CLI is also a Cargo subcommand
- **Why.** An IDE's Cargo run configuration starts `cargo rusteal watch` the same way on Linux, macOS and Windows (`a3ce116`).
- **Implies.** `default-run = "rusteal"`: without it `cargo run -p rusteal` was ambiguous and broke the release (`1e1c540`).

### E2. Platform behaviour is chosen at compile time
- **Decision.** The UBT and UAT scripts, the platform name, the library naming and the platform optimizations written all come from `#[cfg(target_os)]`.
- **Why.** A CLI only builds for its own platform; a Windows or macOS CLI never writes Linux settings.

### E3. Per-platform engine settings are written by the CLI, between markers
- **Decision.** `[optimizations.<platform>.<File>.<Section>]` in `rusteal.toml`, written into `Config/<Platform>/<Platform><File>.ini` by `new`, `watch` and `package`; the user's keys win; `--optimizations off` leaves them out (`916d531`).
- **Why.** Settings must reach every machine and every package without editing the engine install, which affects every project and is lost on upgrades. The default, `r.Vulkan.CpuWaitForFence=0`, avoids an NVIDIA Linux driver crash.
- **Rejected.** Writing them only where an NVIDIA GPU is detected: the file is versioned, machines would rewrite it back and forth, and the player's GPU is unknown when packaging. See [Platform optimizations](../internals/platform_optimizations.md).

### E4. `rusteal build` builds the Rust libraries only
- **Why.** A change to Rust needs steps 4 and 5; the full pipeline only when the engine, the modules or the blocklist change (`fbe3c92`).

### E5. `rusteal new` does not run `git init`
- **Why.** Versioning is the user's call; it writes a `.gitignore` so a later `git init` picks the right files (`46bdf93`).

### E6. A project holds no machine paths
- **Decision.** `rusteal.toml` holds what varies between projects; the layout is fixed by convention; the engine path is per machine, in the CLI's config directory (`7d28dde`, `83bf735`).

### E7. `.env` is for contributors only
- **Why.** A contributor's checkout setting must not change which engine a game builds with; the CLI never reads it (`7d4763b`).

### E8. Everything the CLI writes is embedded in it
- **Decision.** `cargo install rusteal` is the whole installation: plugins, templates, features.
- **Implies.** The published crate carries a snapshot of `ue_plugin/` (`ue_plugin_embed/`, written by `rusteal sync-plugin` before packaging), because `cargo package` cannot reach outside the crate (`3d380b9`, `e858c73`).

### E9. A crate is built by its own manifest, not `-p`
- **Why.** `-p inventory` collided with the `inventory` dependency; a game named Glam would collide with `glam` (`7604eb4`).

### E10. UBT runs with `-NoHotReload`
- **Why.** With the editor open, UBT named the binaries for a C++ hot reload and the link failed; Rusteal never hot reloads C++ (`7704dc2`).

## Hot reload

### F1. A reload loads a numbered link or copy of the deployed library
- **Why.** A build can always replace the deployed file; a hard link costs nothing whatever the size; Windows locks a loaded DLL, so it copies. macOS copies too (`e038281`): `dlclose` never unloads a Rust library there, and `dlopen` returns the image already loaded from the same inode, so a link to an unchanged file reloaded the old, initialized library and `rusteal_init` failed.
- **Implies.** On macOS each reload keeps the previous library mapped until the editor exits.
- **Implies.** The deploy step replaces the file (`.partial` then rename), never writes into it (`e8548e0`).

### F2. Classes are reinstanced with `FReload`, not Live Coding
- **Rejected.** Live Coding: Windows-only, and it patches UBT-built machine code, which a Cargo library is not.
- **Implies.** Reinstancing happens in the editor only; garbage collection is skipped during it to avoid a stall (`7081c51`).

### F3. Change is detected by a compile-time shape
- **Why.** A change to code alone keeps the class and costs nothing; only a structural change replaces it (`7081c51`).

### F4. Structs and enums are not hot reloaded; private fields reset
- **Why.** `FReload` reinstances classes; a changed struct or enum needs a restart, and Rust-private fields come back from `Default`.

### F5. Blueprints that only use a replaced class are recompiled too
- **Why.** The engine recompiles a class's children only; a Level Blueprint calling a removed function hit a fatal error on Play (`ff19417`).

### F6. `rusteal watch` polls
- **Why.** Small workspaces, the same behaviour everywhere, no dependency; the build runs as a child process so a failed build does not stop the watch (`656eaa1`).

## Templates

### G1. Each variant is a complete template
- **Decision.** `templates/<template>/<variant>/`, `base` the default; a new variant starts as a copy of the closest one, never layered (`da53d5e`).
- **Why.** Each variant is its own project, as the engine's Blueprint templates make them.
- **Implies.** Files repeat across variants (every `rusteal.toml.tera` carries the same blocklist and optimizations; tests keep them in agreement).

### G2. Ports keep the engine's names and move assets with redirects
- **Why.** The variants' Blueprints, StateTrees, EnvQueries and levels keep working (`docs/template-variants.md`).

### G3. World Partition actors are redirected to `/Script/Engine.Actor`
- **Why.** An actor descriptor stores its first native class; a Rust class is not native, so a redirect to it makes the level skip the actor.

### G4. Blueprint-only templates are reparented: logic to Rust, data stays
- **Decision.** Graphs removed; components, variables, enums and structs move to Rust under the same names; Anim Blueprints keep their AnimGraph (removing it crashes the compiler); the rebase runs headless in two passes (`459057b`, `c75bcd2`).

### G5. Features are plugins
- **Why.** Gameplay written once is reusable by any template, and by existing projects with `rusteal plugin new` (`9f44915`). Project features add files instead, such as the RustRover run configurations (`39aab8b`).

### G6. Levels that place many Blueprints keep their components in the Blueprints
- **Why.** A Rust component of the same name made the level discard each actor's values; the Intro level places 102 such actors (`dacf99b`).

### G7. The dev and release profiles
- **Decision.** `dev`: the game crate at `opt-level = 1` with full debug info, dependencies and bindings at `3` with `line-tables-only`. `release`: one codegen unit and thin LTO.
- **Why.** The levels follow Bevy's recommendations for games; full debug info made the library pass 100 MB, reloaded on every change. The bindings need their own `[profile.dev.package.bindings]`, since `"*"` does not cover workspace members.

### G8. Templates are named for what they are
- **Decision.** No aliases (`tps` became `third-person`); names still match loosely (`da582af`).

### G9. The templates are the reference code
- **Decision.** `examples/gem_collector`, the original demo, was removed in 0.7.0 (`bde75b8`): the templates cover the runtime API and every change keeps them building and playing.

## Build and release

### H1. Rust 1.88 is pinned, with no host triple
- **Why.** The checkout cannot use newer Rust by accident; 1.88 is the version where edition 2024's `let` chains, which the code uses, are stable. Without a triple the file works everywhere; on Windows the host must be MSVC, which the engine links with, and on macOS `aarch64-apple-darwin`.
- **Implies.** Packaging in the release pipeline uses `cargo +stable`, because the pinned Cargo cannot package crates whose internal dependencies are not published yet.

### H2. `Cargo.lock` is not versioned
- **Why.** Every build would rewrite the internal versions in it; a dependency release that breaks the workspace shows up in CI. `cargo install --locked` still works, since the published crate packs its own lock file.
- **Rejected.** Versioning it, tried and reverted the same day (`40f4f8a`, `fe7a952`).

### H3. The next version comes from crates.io and Conventional Commits
- **Decision.** The last release is the `rusteal` crate's `max_version` on crates.io, the one source that cannot drift; `BREAKING CHANGE:` bumps major, `feat` minor, the rest patch (`56d5ef5`).

### H4. Commits are counted from the published version's tag; the version never goes down
- **Why.** Counting only what `main` lacked lost the `feat` commits of a merge whose release had failed: 0.7.0 came out as 0.6.2 (`34556c9`).

### H5. The version is committed on the release pull request, and checked before the merge
- **Why.** A release that cannot publish must fail the pull request, not leave `main` ahead of crates.io; the `main` ruleset requires the `build` and `bump` statuses Prepare release posts (`8a2d092`, `4dd6ba3`).

### H6. The bot's version commits carry the CI skip marker
- **Why.** Runs triggered by the bot on a pull request wait for an approval and fail when it expires (`a48f7fb`). Never write the marker anywhere else, not even in a message about it.

### H7. Publishing is ordered, repeatable, and skipped when nothing changed
- **Decision.** Crates are published in dependency order, waiting for the index; an already published crate is skipped, so a failed run can be repeated; nothing is released when no package's content changed since the last tag (`aa55166`, `a2da834`).

### H8. `develop` is synced by its own fast-forward workflow
- **Why.** A failed sync never touches what the release did, and it can be re-run alone; it refuses to push when `develop` has commits `main` lacks (`aa55166`).

### H9. Release workflows ask for `contents: write`
- **Why.** The default token is read-only in new repositories, and pushing the tag failed (`68673dd`).

### H10. A vulnerable dependency fails the checks
- **Decision.** `cargo audit` runs in CI and in Prepare release, before it reports the statuses `main` requires (`7d873c2`).
- **Why.** A dependency with a RustSec vulnerability advisory must never reach a release; `Cargo.lock` is not versioned (H2), so the audit checks what a build resolves that day.
- **Implies.** Only vulnerabilities fail; unmaintained and yanked crates warn. A new advisory can turn a pull request red with no change of its own: update or replace the dependency. `cargo-audit` comes prebuilt through `taiki-e/install-action`, pinned to a commit.

## Process

### I1. The engine's templates set the priorities
- **Decision.** A feature earns its place because a template, or a real project, needs it (`docs/third-person-gaps.md`; see the [Roadmap](../roadmap.md)).

### I2. Tracking documents use stable ids
- **Decision.** `TP-GAP-NN` and `UE-DEP-NNN` are never renumbered; fixed entries keep their number.

### I3. Small commits, Conventional Commits subjects, no rationale comments in code
- **Decision.** One commit per fix; the subject's type drives the version (H3); the sources carry no comments, Rust follows rustfmt and C++ `.clang-format`. The reasons go in commit bodies, `docs/` and this chapter.

### I4. IDE-only projects are versioned but never shipped
- **Decision.** `rusteal.sln` and `ide/RustealRules` give the `*.Build.cs` files the context UBT gives them; without them a language server flags required `using` lines as unnecessary, and removing them breaks UBT (`f0e9d1d`).

### I5. Verified headless, then by hand
- **Decision.** Ports and hot reload are validated in a headless editor driven by Python, then by hand; the docs record what was tested and what was not (Windows hot reload, changed parent classes, packaging on macOS).

### I7. A game never needs `unsafe`, and is written in idiomatic Rust
- **Decision.** Two rules for every feature Rusteal adds: a game developer using it never has to write `unsafe` code, and the whole game can always be written in idiomatic Rust. Every game and plugin crate the templates create starts with `#![deny(unsafe_code)]`.
- **Why.** Rusteal's promise to a game developer is Rust's: memory safety checked by the compiler, and code that reads like any other Rust. An API that hands out raw handles, or needs `unsafe` to be used, moves the risk onto the game, where nobody audits it.
- **Implies.**
  - `unsafe` lives in Rusteal (`rusteal-core`, the generated bindings, the macros' output) behind safe functions; a raw handle from the engine becomes a typed reference through a checked call (`UObjectRef::from_handle`, `SubclassOf::from_class`), never through `from_raw` in game code.
  - Game-facing APIs take and return typed references and Rust types (`SpawnParams` with `Option<UObjectRef<Actor>>`, `CommandContext::world()`, `load_class::<T>()`), not `UObjectHandle`s, and log through the `log` crate.
  - The macros' output is accepted by `#![deny(unsafe_code)]` (`tests/manual_compiles.rs` compiles its game crate with it), and `game_developers_never_write_unsafe_code` (`rusteal-cli/src/templates.rs`) fails when a template's code uses `unsafe`, `from_raw`, raw handles or `ulog!`.
  - A feature that cannot meet the rules is not finished: either the safe API comes with it, or the gap is recorded in [Known issues](../contributing/known_issues.md).

### I6. The README presents, the handbooks document
- **Decision.** The README keeps the presentation (what Rusteal is, why use it, UE6, acknowledgments, license) and points to the two handbooks; using Rusteal is documented in the game developer's handbook, working on it in this one.
- **Why.** Nobody reads a 1600-line README; each handbook serves one reader.

## Recording a decision

Add a record here when a change:

- picks one approach among others a reader might expect (the rejected one goes in the record);
- adds a constraint others must keep (an ABI rule, a layout, an order);
- works around an engine or toolchain behaviour;
- changes the process (branches, releases, conventions).

Write it in the area it belongs to, with the next number, in the same pull request as the change, and name the commit or file it comes from.
