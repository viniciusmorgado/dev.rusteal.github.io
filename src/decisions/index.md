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
- **Decision.** The game's code is a `cdylib` the Rusteal plugin loads. A Rusteal plugin has its own library, workspace, bindings and C++ module, which registers the library (`RustealRegisterPluginLibrary`, `RustealRegisterPluginLibraryChecked` since 0.9); its C++ symbols are prefixed `<Plugin>_` (`8e66cd7`, `5c1ac09`).
- **Why.** Gameplay systems become reusable across projects. Classes, functions and pins remember the library that made them, and every call runs with that library current.
- **Implies.** A Rust class cannot derive from another library's Rust class. A runtime plugin may add an editor library (A19).

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

### A13. A library is a module of the runtime: its own state, global ids
- **Decision.** `entry!()` defines a `Module` per library; the callbacks the library returns enter it, and what Rust code registers while it is current (tick hooks, jobs, tasks, closures, pins, types, function callbacks, instance data) belongs to it. Registrations and load hooks name the crate that submitted them. Ids stay global and unique; a module's shutdown waits for and clears only its own state, through a start and stop state machine that makes a second shutdown a no-op and cleans up after a partial start.
- **Why.** In the editor each library is a separate dynamic library with its own copy of the runtime; in a static build every library shares one runtime. Keeping each library's state in its module serves both with one design: nothing changes for a dynamic library, and in a static build each library still ticks once per frame and shuts down alone.
- **Implies.** Rust code that runs while a library is current registers in that library, as the C++ side tags objects with the current library. Code on threads Rusteal did not start is attributed to the first library, the game's (`72a5172`).

### A14. A Rust type's id is the hash of its module path and name
- **Decision.** `#[uclass]` and the other macros compute the id at compile time from `module_path!()` and the name; a parent or a component contributes its own id through `UeClass::RUST_TYPE_ID` (0 for engine classes) instead of the macro hashing the name it was written with.
- **Why.** The short name collided across libraries (`Inventory::Item` and `Quest::Item`), and a parent written as a relative path hashed to the wrong id.
- **Implies.** Moving a class to another module gives it another id: a hot reload treats it as a new class (`d104817`).

### A15. Each library's bindings are their own package, with their own function table and a fingerprint
- **Decision.** The bindings crate is the package `<crate>-bindings`, imported as `bindings`; it owns the table of its C++ wrappers (`__RUSTEAL_FUNCS`), which `entry!()` binds when the library starts, checking the function count and a fingerprint of the functions that codegen writes on both sides and the plugin module passes (`RustealRegisterPluginLibraryChecked`).
- **Why.** Cargo refuses a crate graph with two path packages of the same name and version, which a static build of a game and its plugins is; positional ids read from one shared table would call another library's functions; and bindings and C++ wrappers from different code generation runs used to call the wrong function silently.
- **Rejected.** Looking the table up by library name through the API table (one more call, and names to keep in sync); a single fingerprint of the whole API table (it would not tell which side is stale).
- **Implies.** `rusteal build` rewrites older projects' dependency and profile; `rusteal setup` replaces C++ generated by an older Rusteal with stubs until codegen runs again (`452b687`, `3343c03`, `4e13003`).

### A16. A static build aggregates every library into one archive, built by the CLI, and checks its registrations
- **Decision.** `rusteal package --static` generates a Cargo workspace whose `staticlib` depends on the game's and the runtime plugins' crates, built with fat LTO and one codegen unit, and links it into the game through the `RustealStatic` External module; the editor stays dynamic. A manifest binary of the same crates lists every registration, and each library checks at start that it found all of its own.
- **Why.** One archive per library would carry one runtime and one standard library each, whose symbols collide in one executable. A linker keeps an archive's object only when it is referenced: with the Rust code in one object, the reference to `rusteal_static_modules` keeps every `inventory` registration, and the manifest turns a silently missing class into an error at start.
- **Rejected.** Explicit registration lists written by the game developer (one forgotten type is a missing class); linker flags such as `--whole-archive` (per platform, and not offered by UBT for every platform); patching UBT.
- **Implies.** Library crates are built as `rlib`s too. A crate with Rust types belongs to one library: shared crates are refused (`246f5ff`, `d2ff6c9`, `2c598a7`).

### A17. The New Project dialog is reached through a launcher plugin that ships no Rust
- **Decision.** An editor-only engine plugin, `RustealLauncher` (`ue_launcher/`), adds a **Rust** category of templates to the New Project dialog; the Rusteal plugin stays in each project, installed by the CLI.
- **Why.** The Rusteal plugin carries each project's generated wrappers, so it cannot be installed once in the engine. Fab builds a plugin from its sources with no `cargo` and accepts no executables, and a plugin on Fab cannot depend on another user-made plugin: a small C++ plugin that calls the CLI meets every rule.
- **Rejected.** A third button next to BLUEPRINT and C++ (two fixed slots, no extension point); replacing the project browser (an override only a loaded module can register).
- **Implies.** The launcher lives outside `ue_plugin/`, which is copied into every project; the CLI does the work (`a8c2b75`). See [The launcher](../architecture/launcher.md).

### A18. A Rust template is a stub the launcher sets up the first time it opens
- **Decision.** Each Rust template is a stub project that enables the launcher and names a Rusteal template and variant in `RustealLauncher.json`; when it opens, the launcher runs `rusteal new --in-place` and restarts the editor. Every variant is a template of its own.
- **Why.** `BuildPlugin` removes `EnabledByDefault`, so a plugin from Fab is not loaded in the project browser and no hook of `TemplateDefs.ini` can run there; the first open is the first moment its code runs. The dialog does not record the variant it shows.
- **Implies.** Requirements are checked in the launcher's window, not in the dialog; the setup starts on its own, since **Create** in the dialog already asked for it (`7f83c10`, `0803284`).

### A19. A plugin's editor library is a library of its own, named after its module
- **Decision.** A runtime plugin's `rusteal.toml` may have an `[editor]` section: a second crate in the plugin's workspace, with its own bindings (`Rust/editor-bindings`), C++ module (`<Plugin>Editor`, type `Editor`), symbol prefix (`<Plugin>Editor_`), library file (`librusteal_<Plugin>Editor`) and package (`/Script/<Plugin>Editor`). Its module registers it with `RustealRegisterPluginModuleLibraryChecked(<Plugin>, <Plugin>Editor, ...)`, which finds the plugin by one name and names the library by the other.
- **Why.** A gameplay system and its editor tools belong in one plugin, as C++ plugins put a runtime and an editor module side by side. Naming every part of a library after its C++ module keeps the two libraries apart everywhere the plugin's name was used, and leaves single-library plugins unchanged.
- **Implies.** The editor library cannot depend on the runtime crate: in a dynamic build its library would register the runtime crate's types a second time, so `rusteal build` refuses it; it reaches the runtime library's classes through reflection until libraries see each other's types (M6). The static build leaves the editor library out (`d25f423`, `c0dd537`, `9c2906f`, `e93efbb`).

### A20. An editor plugin for any project carries a copy of the runtime, renamed after it
- **Decision.** `rusteal plugin package --standalone` copies the Rusteal plugin's C++ into an editor plugin as `<Name>Rusteal` and `<Name>RustealEditor`, every `Rusteal` in names and code becoming `<Name>Rusteal`; the copy looks up the Rusteal types Rust asks for under those names, and the plugin's Rust library, unchanged, is prebuilt into `Source/ThirdParty/RustLibraries/<Platform>/`, without Rust sources.
- **Why.** Editor tools are meant for any project, Blueprint and C++ ones included, installed in the engine like plugins from Fab, which may not depend on a plugin made by someone else. Several such tools, and a Rusteal project's own plugin, can be loaded in one editor: the renaming keeps their C++ types, classes, symbols and console commands apart.
- **Rejected.** Rusteal installed in the engine as a shared dependency (Fab refuses it, and every tool would be tied to one Rusteal version); renaming only the types that clash (a missed one shows up only when two tools meet).
- **Implies.** Only editor plugins: a gameplay plugin goes into a game, so the game is a Rusteal one. The renaming is textual and must keep working: lowercase `rusteal` names the Rust side and stays as it is (`fcadc17`, `f736ee2`, `3564bcf`, `b60839f`). See [Standalone editor plugins](../architecture/standalone.md).

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

### B9. The runtime takes threads, time and memory from the engine
- **Decision.** `task::spawn` runs on the engine's task system, `task::is_game_thread` and `time::Instant` ask the engine, through the API table's `platform` sub-table; a static build allocates through `FMemory` (`EngineAllocator`, on unless `[static] engine_allocator = false`). The `std` versions remain only as a fallback for tests without the engine, and a test keeps them there. Game code may still use `std`; `[consoles] check = true` lists where it does, without refusing anything.
- **Why.** The engine is already ported to every console it supports, Rust's standard library is not: what the runtime takes from the engine needs no port. The engine's tasks also show in Unreal Insights and respect the engine's thread budget, and its allocator puts Rust's memory in the engine's memory tools.
- **Rejected.** Forbidding `std` threads and crates such as `rayon` in game code (most games never ship on consoles, and a platform's `std` port may support them); a Cargo feature to block them (features add, they cannot forbid a dependency).
- **Implies.** A task that never ends holds one of the engine's workers. Locks and thread-locals are still `std`'s (`849381b`, `66d20a2`, `bdcec59`, `8ec7933`, `1ac7df7`).


### B10. A panic is reported before it aborts
- **Decision.** Where Rust is built with `panic = "abort"` (`cfg(panic = "abort")`), the panic hook reports the panic, its message and location, to the engine's log before the process ends; with unwinding, `ffi_boundary` still catches it and the game goes on.
- **Why.** Consoles may require aborting on a panic, and an abort with no message leaves a crash report that says nothing about Rust (`9e72b64`).

### B11. Locks and thread-locals come from one module
- **Decision.** `rusteal-core` takes `Mutex`, `RwLock`, `Once`, `OnceLock` and `thread_local!` from `sync.rs` only, and a test keeps it so; they are still `std`'s.
- **Why.** They are the `std` services a console's port of `std` is most likely to have, so replacing them now would be work for nothing; keeping them in one place makes the replacement one file if a platform lacks them (`e7d9d69`).

### B12. `rayon` runs on the engine's threads, one pool per library
- **Decision.** `task::spawn_thread` starts dedicated engine threads (`FRunnableThread`); the runtime's `rayon` feature builds `rayon` pools whose spawn handler uses it, one per library, created on first use with the engine's worker count and joined when the library shuts down.
- **Why.** Games use `rayon`; its global pool starts the operating system's threads and never stops them, so a hot reload unloads the code they run. Engine threads are what a console provides and the engine profiles; a pool that belongs to its library ends with it.
- **Rejected.** Running `rayon` workers as tasks of the engine's task system (they are loops that last as long as the pool, which would hold the task workers); one pool for every library in a static build (each library's shutdown has to wait for its own work only).
- **Implies.** The global pool stays usable and stays unsafe across a hot reload; the books steer away from it (`14e91ea`, `6de3453`, `6f2585f`, `b936a28`, `be61ffe`).
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

### C5. A breaking change ships as a feature, with a note `rusteal upgrade` prints
- **Decision.** An incompatible API change goes in under a `feat:` subject, never a breaking-change one, until 1.0; its note is added to `MIGRATIONS` (`rusteal-cli/src/upgrade_cmd.rs`) under the version it ships in, and to the game developer's handbook ([Versions and upgrades](https://viniciusmorgado.github.io/game.rusteal.github.io/getting_started/versions.html)). `rusteal upgrade` prints the notes of the versions it crosses before it builds (`150d3ed`).
- **Why.** The release pipeline turns a breaking-change subject into a major bump, 1.0.0, which is a milestone of its own. Game code cannot be rewritten reliably, but the compiler points at every line once the note says what changed.

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

### D6. The exporter builds into its plugin, not the engine
- **Decision.** `RustealExporter.ubtplugin.csproj` has a relative `OutputPath`, `..\..\Binaries\DotNET\UnrealBuildTool\Plugins\RustealExporter\`, the plugin's `Binaries/` (`ue_plugin/RustealGenerator/Source/RustealExporter/`); it was `$(EngineDir)\Binaries\...`.
- **Why.** UBT records a UBT plugin's assembly relative to its `.csproj` and combines the two again on each build; when that relative path climbs to the root, the project on another volume than the engine (`/Volumes/...` and `/Users/...` on macOS) or under `/tmp` on Linux, the result loses its leading `/` and UBT stops at step 1 (`... RustealExporter.dll is not an absolute path`). In the engine, every project also shared one `RustealExporter.dll`, whatever its Rusteal version, and an engine install can be read-only.
- **Implies.** Each project builds its own exporter (`Plugins/RustealGenerator/Binaries/`, ignored by the project's `.gitignore`); a `RustealExporter` directory left in an engine's `Binaries/DotNET/UnrealBuildTool/Plugins/` by older versions is unused and can be deleted.

### D7. A buffer the engine writes a struct into has the struct's size
- **Decision.** For an out parameter or a return value of struct type, the generated binding allocates `rusteal_core::struct_size_by_name("<Struct>")` bytes, the engine's `GetStructureSize()`, looked up once per call site (`param_helpers::struct_buffer_expr`); the C++ wrapper copies `sizeof(F<Struct>)` into it.
- **Why.** The buffers were 256 bytes whatever the struct: `FHitResult` is larger, and every sweep (`K2_SetRelativeLocation`, `K2_SetActorLocation`, ...) wrote past the end of the buffer into the heap. macOS's allocator caught it, a crash in `malloc` or a bad pointer later; the other platforms corrupted memory silently. Found with Guard Malloc ([Testing a change](../contributing/testing.md#memory-errors-on-macos)).
- **Implies.** Never size a buffer the engine writes into from a guess; the codegen test `struct_out_buffers_take_the_engine_size` checks the out parameters.

### D8. Strings cross whole and as UTF-8
- **Decision.** A string the engine returns is written with `RustealWriteUtf8` (`RustealUtf8.h`): UTF-8, and when it is longer than the buffer the binding gave, kept aside under the buffer's address for `take_pending_utf8`, which `rusteal_core::take_utf8` calls (`78cd7f9`). Strings from Rust are converted from exactly the bytes passed (`RustealUtf8ToFString`, `1dc64d3`).
- **Why.** The wrappers converted to ANSI and cut at 512 bytes, which could split a character; the reverse conversion read past the end of Rust's bytes. Both found by the engine harness.

### D9. Engine interfaces are named by their `UInterface`, and cross as objects
- **Decision.** The exporter names an interface by its `UInterface` (`BlendableInterface`), not the native class (`IBlendableInterface`); the wrappers build a `TScriptInterface` with `RustealToScriptInterface` (`RustealInterface.h`), which keeps the object and sets the native pointer when there is one; `FScriptInterface`, the generic `IInterface`, is `UObjectRef<Object>` (`2ffcf6b`).
- **Why.** The native name matched no class, so every engine function and property with an interface was dropped. The engine's own `TScriptInterface(UObject*)` clears the object when it does not implement the interface natively, which breaks Blueprint-implemented interfaces and `IInterface`.

### D10. Engine soft references cross as paths, weak ones as objects
- **Decision.** An engine `TSoftObjectPtr<T>` is `SoftObjectRef<T>` and crosses the boundary as its path, a string; `TWeakObjectPtr<T>` is `TWeakObjectPtr<T>` and crosses as the resolved object, through `ObjectPointer` (`d5122d2`).
- **Why.** As `UObjectRef<T>`, an unloaded soft reference read as null and lost its path, and a function taking one could only be given a loaded object; a weak one read as a strong handle that could dangle.
- **Implies.** Functions with a soft reference and a container are skipped. Arrays, maps and sets of soft references were `UeArray<UObjectRef<T>>` of the loaded objects; since `610aa47` their elements cross as paths too, `UeArray<SoftObjectRef<T>>`.

### D11. Engine functions with no export macro are left out of every project
- **Decision.** `ENGINE_UNEXPORTED_FUNCTIONS` (`rusteal-codegen/src/config.rs`) joins every project's blocklist, as Rusteal's internal classes do (`2ffcf6b`, `bad00f6`). Likewise `ENGINE_PLUGIN_PACKAGES` names the plugin of an engine package a project maps without one, such as `GameplayAbilities` (`d61aba5`).
- **Why.** A project's `[codegen.blocklist]` is written once by `rusteal new` and never migrated: a function newly bound by a fix would fail to link in every existing project.

### D12. A struct the engine gives Rust is constructed in Rust's buffer, and owned
- **Decision.** Whatever writes an engine struct into a buffer Rust owns (a function's return or out parameter, a property, an array element, a delegate parameter) constructs it there through its `UScriptStruct` (`InitializeStruct`, then `CopyScriptStruct`), and Rust takes it with `OwnedStruct::from_constructed`, which destroys it on drop; `clone()` copies it through the struct too (`5f16556`). A struct passed by reference is used in place, as `&mut OwnedStruct<T>`. The `UScriptStruct` is `TBaseStructure<T>::Get()`, or, for the engine's `NoExportTypes` (`FBox`, `FIntPoint`...), the one found by its path (`RustealFindScriptStruct`), so every reflected struct is an `OwnedStruct` (`984b2c0`).
- **Why.** The wrappers `memcpy`'d a struct and destroyed the original, leaving Rust with its freed arrays; a copy into zeroed memory was never destroyed; structs with no `StaticStruct` crossed as raw `*const u8`, 190 functions in the default modules, their buffers leaked.
- **Rejected.** A placement move (`new (Buf) T(MoveTemp(Value))`): some structs' move constructors are not exported (`FImportanceTexture`).

### D13. `FText` is the engine's own, held by Rust
- **Decision.** `Text` owns a heap `FText` the engine allocated (`new FText`), and crosses as its pointer; arguments take `impl Into<Text>` (`b8a0fe6`).
- **Why.** Through `FString`, a text lost its namespace and key, so it could not be translated.
- **Implies.** An `FText` parameter with a default value is a required one; the engine's text gatherer does not read Rust (roadmap, M8).

### D14. An engine function that takes a delegate takes a closure, through a proxy kept by a registry
- **Decision.** The closure is registered, `make_delegate` binds an `FScriptDelegate` to a new `URustealDelegateProxy`, and the wrapper passes it as `FRustealDelegateArg`, which converts to whatever dynamic delegate type the function takes (`1a44e6d`). Every proxy is kept by an `FGCObject` registry, holds its owner weakly, frees its closure when destroyed, and goes inert when its library unloads (`5542357`, `b7ff413`).
- **Why.** Nothing referenced the proxies: the first garbage collection destroyed them, and delegates stopped calling Rust, silently. Naming each delegate's C++ type would need the exporter to give it; the conversion operator needs none, and is limited to `FScriptDelegate`-derived types so that overloads taking a static delegate stay unambiguous.
- **Implies.** For a static function the proxy's owner is the transient package and its world is `GWorld`: it lives until the library unloads.

### D15. A container element crosses in a buffer of its size
- **Decision.** `ContainerElement` gives each value's `write_size()` and each type's `read_size()`; an element is written into a buffer of its size, and read into one of `read_size()`, read again when the engine reports more (`ReadElement` reports the full size and writes only what fits) (`1a38c3e`).
- **Why.** Elements went through a fixed 4096-byte stack buffer with no bounds check: a longer string, or a larger struct, wrote past it, and the engine cut strings to it.

### D16. Rust maps and sets are reified with their key and value properties
- **Decision.** `RustealReifyPropExtra` carries a second element type, `value_prop_type`, with its handles in `value`, a pointer to another extra; a map's key and a set's element are the first one, and get `CPF_HasGetValueTypeHash` (`9a79775`). Keys are the hashable kinds: numbers, objects, classes, `FName` and `String`.

### D17. A Rust function's parameters cross as properties of its parameter buffer
- **Decision.** `DynamicValue` (`write_to`, `read_from`) reads and writes a value of any type a property holds in any container of it; a function's parameter buffer is one, with each parameter's `FProperty` found once and kept beside its offset. The function thunk, `DynamicCall::set_value`/`value`, Blueprint event dispatch and `#[udelegate]` all use it for strings, texts, names, containers, soft and weak references and interfaces (`4cde66e`, `406c8ea`, `f013a67`). Function parameters take Rust's `Vec`, `HashMap` and `HashSet`, copied, where properties hold `UeArray`, `UeMap` and `UeSet`.
- **Why.** These types own memory the engine manages: copying their bytes in and out of a call, as scalars are, would share or leak it. The property API that already reads and writes them on objects works on any container.
- **Rejected.** `UeArray` and friends as parameters (a handle into a buffer that is freed when the call returns).

### D18. A `BlueprintNativeEvent` is dispatched through the engine
- **Decision.** The macro renames the user's body to `<name>_implementation`, registers it as the event's native function, and gives the method's own name to a dispatcher that calls the event through `ProcessEvent`, so a Rust or Blueprint override runs when there is one (`5d04da6`).
- **Why.** Calling the Rust body directly would skip the children's overrides, which is the point of the event; C++'s `_Implementation` convention is the one Unreal developers know.

### D19. A Rust class's functions are registered after its Rust parent's
- **Decision.** The reify registry orders the function blocks by the classes' dependency order, as it does the classes (`378cb32`).
- **Why.** An `Override` copies its parameters from the parent's function; registered first, it found a function with none.

### D20. A `#[ufunction]` with no receiver is static
- **Decision.** No `self` means `FUNC_Static`, and the thunk calls `Type::method(args)`; events refuse it. Function metadata (`CallInEditor`, `category`, `meta(...)`, the doc comment as `ToolTip`) goes through `set_function_metadata` (`e7fdc27`).
- **Why.** It is how Rust spells an associated function, and a function library is a class of them.

### D21. Single-cast and sparse delegates are reified like the multicast one
- **Decision.** `add_delegate` takes a kind: a single-cast delegate is an `FDelegateProperty` executed with `execute_delegate`; a sparse one an `FMulticastSparseDelegateProperty` whose signature is a `USparseDelegateFunction` naming the class and the delegate, with its offset registered with `FSparseDelegateStorage` when the class is finalized. A single-cast binding is undone only by its own `DelegateBinding` (`unbind_delegate` takes its callback id) (`f013a67`).
- **Why.** The engine resolves a sparse delegate's owner from that offset, by class name; with none registered it asserts. A binding dropped after another replaced it unbound the newer one.
- **Implies.** Reified classes are not `CLASS_Native`, so `FSparseDelegateStorage::ResolveSparseDelegate` (used only by C++ accessors) does not find them; the property paths Blueprints and Rust use do not need it.

### D22. An `Override` duplicates the parameters of the event it overrides
- **Decision.** `CopyParamsFromParentFunction` copies each parameter with `FField::Duplicate`, inner properties included (`ac543d8`).
- **Why.** It rebuilt them from a list of types and skipped the others (arrays, maps, sets, soft and weak references, interfaces), so the override's layout no longer matched the event's.

### D23. Every attribute refuses the arguments it does not know
- **Decision.** `#[uclass]`, `#[component]`, `#[uproperty]`, `#[ufunction]`, `#[udelegate]`, `#[uclass_impl]` and `#[class_defaults]` fail with the list of what they take (`c7d30d5`).
- **Why.** Ignored arguments compiled and did nothing: `Blueprintable` or `Replicated` looked like they worked.

### D24. Async tasks run on a per-library executor, on the game thread
- **Decision.** `executor.rs` keeps one executor per `ModuleId`, in a game-thread `thread_local!`, polled in `task::tick` after the jobs. Futures are not `Send`; the waker is, and pushes the task's id into a global `READY` map of that module, so an engine callback or a worker thread can wake it. A task has an optional owner, kept as a weak handle and checked before each poll and in `sweep_dead_owners` every tick: a dead owner drops the future at its `.await`. `task::shutdown` drops a module's tasks before the library unloads. `JoinHandle<T>` is a future giving `Option<T>` (`None` once cancelled); dropping it detaches; `AbortHandle` is `Send`, through a global set of aborted ids (`e48b9c6`, `3a7f638`).
- **Why.** Engine calls belong to the game thread, so the futures that make them do too; requiring `Send` would forbid `UObjectRef`s and the class values game code holds. Owner cancellation is what makes a task safe to write without checks after every `.await`: it never resumes on a destroyed object. A task left running past an unload would poll code that is gone.
- **Rejected.** `tokio` or another runtime on the game thread (a second loop beside the engine's, and futures crossing threads); polling every task every frame (wakers make an idle task free).

### D25. Timers own their closures through the engine's delegate
- **Decision.** `timer::set_timer` goes through `RustealTimerApiImpl.cpp`: an `FTimerDelegate::CreateLambda` that holds a shared `FRustealTimerCallback`, whose destructor releases the Rust closure. The engine destroys the delegate when the timer is cleared, when a one-shot timer has fired, or when its world goes away, so the closure is freed with it. `TimerHandle` keeps the context as a weak handle and the engine's `FTimerHandle` as a `u64`; dropping it does not clear the timer, as in Blueprint. `timer::sleep` is a one-shot timer that wakes a future, in world time; `sleep_real` uses the engine's clock. `k2_set_timer_delegate` and `k2_set__for_next_tick_delegate` are hand-written in `manual/timer_ext.rs` on the same path (`ead385a`, `8db536d`).
- **Why.** The generated `k2_set_timer_delegate` bound a delegate proxy that lived until the library unloaded: each call leaked a proxy and a closure.

### D26. An engine latent function is awaited through a proxy object
- **Decision.** A function with an `FLatentActionInfo` parameter generates `fn x(..) -> impl Future<Output = Outputs>` without it; the future is a `latent::LatentCall`, started on its first poll. The C++ wrapper creates a `URustealLatentProxy` (`Create`, `MakeLatentInfo`, `LatentDone`, `Cancel`) whose `FLatentActionInfo` points the engine's `ExecutionFunction` at the proxy; the function's outputs live on the heap (`TSharedRef`) until `OnComplete` converts them. Dropping the future calls `RemoveActionsForObject` on the proxy in every world. A latent function with a container parameter is not bound (`71967fa`, `211372b`).
- **Why.** The engine writes a latent function's outputs when its action finishes, frames after the call returns: a parameter buffer on the stack would be gone by then. The proxy gives every call its own callback target, so concurrent calls never share a UUID.

### D27. An `async #[ufunction]` takes `self` by value and returns nothing
- **Decision.** The macro registers a hidden `LatentInfo` parameter (the struct found with `reflection_find_struct("LatentActionInfo")`) and the `Latent`/`LatentInfo` metadata; the thunk passes the future to `latent::start_latent_function`, which spawns it owned by the object and adds a `FRustealLatentAction` (`FPendingLatentAction`) that triggers the node's output when the task finishes. An abandoned action (the object destroyed, the node aborted, PIE stopped) cancels the task and triggers nothing. `&self`, `&mut self`, out parameters and a return value are refused; `RustealResult<()>` logs its error as a warning (`c80346d`).
- **Why.** A future lives across frames while the engine runs other code on the object: a borrow of the class held across an `.await` would alias the one a tick or an overlap takes. `self` by value is the handle; owner cancellation (D24) keeps it valid. A latent node's outputs are copied when the node starts, so a value computed later cannot reach them.
- **Lesson.** Rust 2021's disjoint closure capture moved only `finish.completed` into an `async move` block that set it, so the struct's `Drop` saw the old value and never fired the node; moving the whole struct first (`let mut finish = finish;`) fixed it.

### D28. A delegate's `next()` binds once and unbinds when it fires
- **Decision.** Every generated multicast wrapper, and every `#[udelegate]` (`<name>_next()`), has `next()`, a future of the next broadcast's parameters, built on `event::once()`: the binding sends to a `OnceSender` and is removed after its first call; a sender dropped unfired (the object destroyed, the library unloaded) gives the receiver an error (`0cec3f7`). Blueprint async actions use it: factory, `completed().next()`, `activate()`, `.await`.
- **Why.** The binding starts when `next()` is called, before what makes the delegate fire, so no broadcast is missed; an error instead of a future that never resolves keeps a task from waiting forever.

### D29. Native engine events and HTTP are hand-written shims
- **Decision.** `RustealEventsApiImpl.cpp` binds a curated list of C++-only delegates (`FCoreDelegates`, `FWorldDelegates`, `FGameModeEvents`, `UPathFollowingComponent::OnRequestFinished`) by a kind number to a lambda holding the library and a callback id, keeps each `FDelegateHandle` per library and removes them in `ForgetLibrary`; Rust exposes them as `NativeEvent<A>` with `add` and `next`. `RustealHttpApiImpl.cpp` sends requests through `FHttpModule`, its completion handing the status, headers and body to the Rust future by the request's id; `Rusteal.Build.cs` adds `HTTP` and `AIModule` on a line of their own (`646aff8`).
- **Why.** Native delegates are not reflected, so code generation cannot see them; a short, reviewed list is cheaper than a generator for them. The separate `Build.cs` line keeps the standalone vendoring's match on `Add("Projects")`.

### D30. `method(self, Self::x)` makes a method a callback
- **Decision.** `method.rs`'s `Method<T, Args, Marker>` trait, implemented for `fn(&mut T, ..)` (`ByMut`) and `fn(&T, ..)` (`ByRef`) with up to nine parameters, turns a method into a `Box<dyn FnMut>` that finds the object again from its handle (`RustObject::from_object_handle`, implemented by every `#[uclass]`) before each call, and does nothing once it is gone. `bind_action`, timers and delegates' `add` take it as any closure; `bind_action_by_name` and `add_ufunction` keep the by-name path for functions a Blueprint defines (`9e8aadc`, `7fea2c8`).
- **Why.** A name is checked when the game runs, and a renamed method failed silently; the compiler checks a method path. The method no longer needs to be a `#[ufunction]`, so input handlers stop showing up in Blueprints.
- **Implies.** `bind_action` changed its signature: a `feat` with a `rusteal upgrade` note, never a breaking-change commit (decided for M1's API changes as well).

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

### E11. On Mac with Xcode 27, `rusteal package` gives `xcodebuild` a stdin
- **Decision.** UAT runs with `DEVELOPER_DIR` pointing at a mirror of Xcode's `Developer` directory whose `xcodebuild` runs the real one with `/dev/null` as its stdin (`rusteal-cli/src/xcode_shim.rs`).
- **Why.** UE 5.8 starts the `xcodebuild` that finalizes the `.app` without a stdin, and Xcode 27 then fails the scheme's pre-action: no Mac game could be packaged, Rusteal or not.
- **Rejected.** Running the finalize step by hand and continuing with `-skipbuild` (the receipt is never written); disabling UBA (5.8 always uses its executor); patching the engine install.
- **Implies.** Remove it once the engine gives `xcodebuild` a stdin ([reported to Epic](https://forums.unrealengine.com/t/5-8-mac-failed-to-finalize-the-app-with-xcode-on-xcode-27-xcodebuild-started-without-stdin-bad-file-descriptor-in-touch-ubt-generated-tiles/2839864)). See [Platform optimizations](../internals/platform_optimizations.md#macos-xcodebuild-without-a-stdin).

### E12. Target platforms are definitions: built in for desktop, files for the rest
- **Decision.** A packaging target is a `Platform`: the engine's name for it, the Rust target, a toolchain, `build-std`, flags, environment, the archive's name, system libraries and extra `BuildCookRun` arguments. Linux, LinuxArm64, Win64 and Mac are built in; any other comes from a TOML file, given by path or found by name in the user's config directory or `RUSTEAL_PLATFORM_PATH`. A platform other than the host's is packaged with `--static` only; an unknown one is refused, and the plugin no longer gives a `.so` name to platforms it does not know.
- **Why.** Console specifics cannot be in a public repository, but the code that uses them can: a private platform pack is a definition file and its toolchain, with nothing to patch in Rusteal. Host and target are separate questions: the host runs UBT and the editor; the target is what the game is built for.
- **Implies.** `rusteal platform show` prints a definition, which is how a pack starts (`5507bed`).


### E13. Library crates are plain Rust libraries; `rusteal build` makes the dynamic library
- **Decision.** A game's or plugin's crate declares no `crate-type`; step 4 runs `cargo rustc --lib --crate-type cdylib`, and `rusteal package --static` rewrites an older `crate-type` with `cdylib` to `["rlib"]`.
- **Why.** A dependency's `cdylib` is linked as a dynamic library for whatever target the static build compiles for: for Linux ARM64 or a console, with a linker the host does not have, and the build failed. `--crate-type` builds the editor's library from the same crate, including in projects that still declare one (`a604d23`).
### E14. `rusteal new --in-place` fills a stub, and only success removes its marker
- **Decision.** With `--in-place`, `rusteal new` sets up the project in the stub's own directory: it keeps what the open editor holds (`Saved/`, `Intermediate/`, `DerivedDataCache/`), clears the stub and anything a stopped conversion left, refuses anything else, and removes the marker after the build. `--engine` gives the engine without the configuration file, which a process without a terminal cannot be asked for.
- **Why.** The editor that runs the CLI has the project open, and a setup can fail or be canceled; with the marker kept and the stub's `.uproject` restored by the launcher, opening the project again retries it (`fd4ed8f`, `94e74cd`).

### E15. `rusteal launcher install` builds the launcher with the engine's `BuildPlugin`
- **Decision.** The CLI embeds the launcher's sources (`LAUNCHER_FILES`), builds them with the engine's `BuildPlugin` and installs the result in `Engine/Plugins/Marketplace/RustealLauncher`.
- **Why.** Linux has no Epic Games Launcher and its editor cannot install Fab plugins; building with the user's own engine needs no binaries in the repository and no change to the release workflows (`2e22292`).

### E16. A precompiled plugin is built by the engine's `BuildPlugin`, with the project's Rusteal plugin as its dependency
- **Decision.** `rusteal plugin package --precompiled` builds the plugin's Rust libraries with the release profile, then runs `BuildPlugin` with `-Dependencies=<project>/Plugins/Rusteal/Rusteal.uplugin`, which copies the Rusteal plugin into the host project `BuildPlugin` makes. The plugin's `Config/FilterPlugin.ini` adds the Rust libraries, the Rust workspace without `target/` and `rusteal.toml` to the package. A plugin whose `.uplugin` says `"Installed": true` is left alone by `rusteal build` and `rusteal generate`; `rusteal plugin merge` joins the per-platform files of packages made on several platforms.
- **Why.** `BuildPlugin` builds a plugin in a project of its own, where the Rusteal plugin it depends on was missing; passing it as a dependency needs no Rusteal installed in the engine. The sources still travel, since a static build compiles the plugin's crates into the game's archive.
- **Implies.** A precompiled plugin works only in projects at the same Rusteal and engine version, as any plugin does. Each platform's binaries are built on that platform (`f559969`, `ef88af5`).

### E17. `rusteal build` goes on when the code generated before no longer compiles
- **Decision.** When the first UBT pass fails and the reflection dump was written, the build prints so, generates the code again and builds it in step 3; it stops at once only when it was asked for step 1 alone (`23d570d`).
- **Why.** After a CLI update, the first pass compiles the wrappers an older version generated; when a fix changes them they may not compile, and the build stopped before the step that regenerates them, leaving the project stuck.

### E18. `rusteal build` exports the reflection again when the bindings need a new engine plugin
- **Decision.** When step 2 adds plugins to a descriptor (a feature whose modules come from an engine plugin: `state-tree`, `niagara`, `chaos-vehicles`...), it touches `RustealGeneratorDummy.h`, runs step 1 again and generates from the new dump (`bad00f6`).
- **Why.** UHT runs only when a project header changes: enabling the plugin did not make it run, so the dump never had the plugin's types and the feature's bindings stayed empty, build after build.

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
- **Why.** The levels follow Bevy's recommendations for games; full debug info made the library pass 100 MB, reloaded on every change. The bindings need their own `[profile.dev.package.<crate>-bindings]`, since `"*"` does not cover workspace members.

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

### H11. The launcher has its own version
- **Decision.** `RustealLauncher.uplugin` starts at `0.1.0` and is not bumped by the release workflows; the launcher accepts any CLI whose `rusteal new` has `--in-place` and `--engine`.
- **Why.** Every Fab submission goes through review: a lockstep version would force one for each Rusteal release, while the launcher changes rarely.

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
- **Decision.** Ports and hot reload are validated in a headless editor driven by Python, then by hand; the docs record what was tested and what was not (Windows hot reload, changed parent classes).

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

### I8. The launcher's sources carry a copyright header
- **Decision.** Each `.cpp` and `.h` of `ue_launcher/` starts with `// Copyright <year> Vinicius Donatto Morgado.`
- **Why.** Fab requires a copyright notice with the publisher's name in every source file; it is the one exception to the rule of no comments in the sources (I3).

## Recording a decision

Add a record here when a change:

- picks one approach among others a reader might expect (the rejected one goes in the record);
- adds a constraint others must keep (an ABI rule, a layout, an order);
- works around an engine or toolchain behaviour;
- changes the process (branches, releases, conventions).

Write it in the area it belongs to, with the next number, in the same pull request as the change, and name the commit or file it comes from.
