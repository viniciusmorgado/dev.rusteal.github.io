# Architecture

The Rusteal repository holds two worlds that meet at runtime. On one side are the **Unreal Engine projects**: two UE plugins, written in C++ and C#, that live inside every game made with Rusteal. On the other side is a **Cargo workspace**: the Rust crates games depend on, the code generator, and the command-line tool that ties everything together.

Two files at the root of the repository list them:

- `rusteal.sln` is a .NET solution for C# language servers. It holds the exporter and an IDE-only project for the plugins' build rules, so editors resolve the engine's types in them.
- `Cargo.toml` is the Rust workspace, with every crate.

The C++ modules are in neither. The Unreal Build Tool compiles them inside a game project, and clangd understands them through the `compile_commands.json` that `cargo xtask dev-setup` writes (see [Setting up a checkout](../contributing/environment.md)).

This chapter walks through each project: what it is, who uses it, and when you would open it.

## The big picture

```
                         ┌─────────────── build time ───────────────┐
Unreal headers ──UHT──▶ RustealExporter (C#) ──▶ reflection JSON
                                                       │
                                                rusteal-codegen
                                         ┌─────────────┴─────────────┐
                                         ▼                           ▼
                               bindings crate (Rust)      C++ wrappers (Generated/)
                                         │                           │
                         ┌───────────────┼──────── runtime ──────────┼───────────────┐
                         ▼               ▼                           ▼
              game crate (cdylib) ──▶ librusteal.so ◀──tables──▶ Rusteal plugin (C++)
              rusteal-runtime                                        │
              rusteal-core                                     Unreal Engine
              rusteal-macros
              rusteal-ffi
```

The `rusteal` CLI drives the whole pipeline. `rusteal build` runs it in five steps:

1. **UE build.** The project is built and the exporter writes the reflection JSON.
2. **Codegen.** `rusteal-codegen` writes the bindings crate and the C++ wrappers.
3. **UE rebuild.** The engine compiles the wrappers into the plugin.
4. **Cargo build.** The game's library is compiled.
5. **Deploy.** The library is copied to where the plugin loads it.

## The Unreal Engine side

Everything under `ue_plugin/` is copied into a game project by `rusteal setup` (or `rusteal new`) and listed in its `.uproject`. There are **two separate plugins**, each with its own `.uplugin` descriptor. The engine loads them independently of each other.

### The Rusteal plugin (`ue_plugin/Rusteal`)

The runtime: the plugin that stays in the game, packaged builds included. It has two C++ modules.

#### The `Rusteal` module (Runtime)

The bridge between the engine and Rust. It is the largest C++ code base in the repository, and almost everything that happens at the boundary goes through it.

When the module starts, it:

1. finds the game's Rust library (`Plugins/Rusteal/Binaries/<Platform>/librusteal.so`) and every Rusteal plugin's library;
2. loads each one and checks its version;
3. hands each one the **API table**, a struct of function pointers to the module's services;
4. calls the library's `rusteal_init`.

A game packaged with `--static` has no library files: the Rust code is linked into the game, and the module finds each library's entry points by name in it instead (see [Static linking](boundary.md#static-linking)).

From then on, Rust calls the engine through that table, and the engine calls Rust through the callbacks the library returns.

Its sources, by role:

- **Module and library host.**
  - `RustealModule.cpp` starts and shuts down the module and keeps the registry of loaded Rust libraries. It also runs the core ticker and implements the `Rusteal.Reload` console command (hot reload).
  - `RustealLibraries.h` and `RustealLibrary.h` hold the *current library*. Before every call into Rust, the module selects the library that owns the class being called, so each library sees only its own objects.
- **The API table.** `Public/RustealApiTable.h` is the C++ side of the contract defined in `rusteal-ffi`. Each sub-table has its own implementation file:

  | File | Services |
  |---|---|
  | `RustealCoreApiImpl.cpp` | Object validity, names, classes and outers, `IsA`, `FName`, weak pointers. |
  | `RustealPropertyApiImpl.cpp` | Reading and writing properties through reflection. |
  | `RustealReflectionApiImpl.cpp` | Finding classes, structs, enums, properties and functions; calling functions by reflection; struct memory. |
  | `RustealContainerApiImpl.cpp` | `TArray`, `TMap` and `TSet` access. |
  | `RustealDelegateApiImpl.cpp` | Binding Rust code to engine delegates. |
  | `RustealLifecycleApiImpl.cpp` | Garbage collector roots and pinned objects, which keep objects alive for Rust. |
  | `RustealReifyApiImpl.cpp` | Creating Rust classes as real `UClass`es. |
  | `RustealWorldApiImpl.cpp` | Spawning actors; finding, loading and creating objects; data table rows. |
  | `RustealWidgetApiImpl.cpp` | UMG widgets. |
  | `RustealInputApiImpl.cpp` | Enhanced Input bindings. |
  | `RustealConsoleApiImpl.cpp` | Console commands and variables. |

- **Rust classes in the engine.**
  - `URustealReifiedClass` is the `UClass` subclass behind every `#[uclass]`. It builds the class at runtime, constructs its components, and creates the Rust data of each object.
  - `URustealReifiedFunction` is the `UFunction` subclass behind every `#[ufunction]`. Its native entry point calls back into Rust, so the engine and Blueprints call it like any other function.
- **Delegates.** `RustealDelegateProxy` is a small `UObject` that receives a dynamic delegate's broadcast and forwards it to the Rust closure bound to it.
- **Subsystems.** `RustealSubsystems.h` declares the native parents that Rust subsystems derive from: game instance, world, engine and local player. The engine only creates subsystems from native classes, so these turn its virtual functions into events Rust overrides.
- **Contract tests.** `RustealContractTests.cpp` holds `static_assert`s that fail the C++ build if a handle or table layout drifts from its Rust counterpart.
- **Build rules.** `Rusteal.Build.cs` does three things:
  - declares the module's dependencies;
  - adds the engine modules the generated wrappers need, from `Generated/module_deps.txt`;
  - stages the Rust libraries in packaged games (`RuntimeDependencies`).

The **`Generated/`** directory does not exist in the repository. In a game project, codegen fills it with the C++ wrappers (one small `extern "C"` function per engine function) and the table that points at them.

#### The `RustealEditor` module (Editor)

Exists only in the editor and is left out of packaged games. It holds `URustealEditorSubsystem`, the native parent for editor subsystems written in Rust. Editor-type Rusteal plugins load this module before registering their classes.

Editor-only features of the runtime go here, so the `Rusteal` module keeps compiling for game targets.

### The RustealGenerator plugin (`ue_plugin/RustealGenerator`)

Acts only at build time; nothing in it runs while a game plays. Despite its name, it does not generate code: it **exports** the engine's reflection, which `rusteal-codegen` then turns into code. Its two parts are very different in nature.

#### `RustealExporter` (C#, a UHT plugin)

Where the work happens. `RustealExporter.ubtplugin.csproj` is not an engine module but a **plugin of the Unreal Header Tool**. The build tool compiles it into a .NET assembly and runs it inside UHT, while UHT parses the project's headers.

`RustealExport.cs` registers an exporter with `[UhtExporter]`. It walks every class, struct, enum, property and function UHT knows about, and writes to JSON:

- names and flags;
- property types and offsets;
- function signatures;
- inheritance and interfaces;
- the header each type lives in.

It also filters out the types the generated code could never use.

`RustealEnumValues.cs` recomputes the enum values UHT leaves unparsed. UHT stores `-1` for anything that is not a plain integer literal, such as `(1 << 0)`, but the bindings need the numbers.

When codegen needs a piece of reflection data it does not get yet, this is the project to change.

#### The `RustealGenerator` module (C++, a stub)

Empty on purpose: an `IMPLEMENT_MODULE` and a dummy `UCLASS`. The exporter declares `ModuleName = "RustealGenerator"`, and that module must appear in UHT's manifest for UHT to know where the exporter's output goes. This module only exists to be that anchor; there is nothing to change in it.

### `RustealRules` (`ide/RustealRules`, IDE only)

The plugins' `*.Build.cs` files are C# that the Unreal Build Tool compiles itself, outside any project. A language server would therefore analyze them with no references: nothing resolves, and every `using` looks unused.

`RustealRules.csproj` links those files and references the build tool's assemblies, giving editors the context UBT gives them. Nothing builds or ships this project; it only matters to your editor.

## The Rust side

The Cargo workspace has eight members. The first five make up the runtime a game links against; the other three are tools.

```
rusteal-ue-flags ◀── rusteal-ffi ◀── rusteal-core ◀──┐
       ▲                                             ├── rusteal-runtime ◀── game crate
       │                              rusteal-macros ┘
       └──────── rusteal-codegen ◀── rusteal (CLI)

xtask: contributor tooling, not published
```

All published crates share one version, and so do the UE plugins. The library and the plugin each check the other's version and refuse to work together if they differ.

### `rusteal-ue-flags`

The engine's reflection flag constants (`EPropertyFlags`, `EFunctionFlags`, `EClassFlags` and others), mirrored from the engine's headers. No dependencies, no logic.

Two crates use it: the runtime, which writes flags when it creates Rust classes, and the code generator, which reads them from the JSON. Touch it when a flag Rusteal needs is missing.

### `rusteal-ffi`

The **contract between Rust and C++**, and nothing else. It has no dependency besides the flags and defines only `#[repr(C)]` types:

- `handles.rs`: opaque handles to engine objects (`UObjectHandle`, `UClassHandle`, `FPropertyHandle`…);
- `api_table.rs`: the API table and its sub-tables, the Rust side of `RustealApiTable.h`;
- `callbacks.rs`: the callbacks the library hands back to the plugin (create and drop Rust data, invoke a Rust function, tick…);
- `reify_types.rs`: the descriptions of Rust classes, properties and functions sent to the reify API;
- `error.rs`: the error codes that cross the boundary;
- `version.rs`: the version both sides check;
- `contract_tests.rs`: compile-time assertions on sizes and offsets, mirroring `RustealContractTests.cpp`.

Every change here is made on both sides at once: the table in `api_table.rs` and in `RustealApiTable.h`, with both sets of contract tests in agreement.

### `rusteal-core`

The **safe runtime**. All `unsafe` calls through the API table are confined to this crate, so generated code and game code only see safe types. Its build script reads `rusteal-ffi/src/api_table.rs` and generates one thin dispatch function per table entry.

Its main modules:

- **Object references.**
  - `object_ref.rs`: `UObjectRef<T>`;
  - `weak_ptr.rs` and `soft_ref.rs`: weak and soft references;
  - `subclass_of.rs`: `SubclassOf<T>`;
  - `pinned.rs`: `Pinned<T>`, which keeps an object alive for Rust.
- **Data.**
  - `struct_ref.rs`: engine structs;
  - `containers.rs`: `UeArray`, `UeMap`, `UeSet`;
  - `fname.rs`;
  - `ue_math.rs`: rotators, transforms, colors and bounds, on top of `glam`.
- **Calling the engine.**
  - `dynamic_call.rs` calls a function by name, for what the bindings do not cover;
  - `world.rs` spawns actors and queries the world.
- **Rust classes.**
  - `reify_registry.rs` registers every `#[uclass]` and creates its `UClass` through the reify API. It keeps each object's Rust data and dispatches the engine's calls to Rust methods.
  - `delegate_registry.rs` does the same for closures bound to delegates.
- **Services.**
  - `console.rs`: commands and variables;
  - `task.rs`: `spawn`, `run_on_game_thread`, `on_tick`;
  - `input.rs`, `widget.rs` and `logging.rs`.
- **Errors.**
  - `error.rs` defines `RustealError` and `RustealResult`;
  - `ffi_guard.rs` catches panics at the boundary, so a panic in game code is logged instead of unwinding into C++.

### `rusteal-macros`

The procedural macros that make a Rust type an Unreal type:

- `#[uclass]`: a struct becomes a `UClass`, with its parent, its `#[uproperty]` fields and its `#[component]`s;
- `#[uclass_impl]` and `#[ufunction]`: methods become `UFunction`s, overrides of engine events, or console commands;
- `#[ustruct]`: a struct becomes a `UScriptStruct`;
- `#[udelegate]`: a multicast delegate declared on a Rust class.

The macros generate typed accessors for each property, plus the registration `rusteal-core` collects (through the `inventory` crate) when the library loads. A typo in a macro argument is a compile error, not a silent no-op.

### `rusteal-runtime`

**The one dependency a game crate needs.** It re-exports `rusteal-core`, `rusteal-ffi`, the flags and the macros, and provides `rusteal_runtime::prelude`.

It has two macros:

- `entry!()` generates the library's module and its entry points, which the plugin looks up by name in a dynamic library: `rusteal_init`, `rusteal_shutdown`, `rusteal_version` and `rusteal_callbacks`;
- `on_load!` runs code on every load, hot reloads included.

The engine's types are not here. They come from the project's own generated bindings crate.

### `rusteal-codegen`

The **code generator**. It reads the exporter's JSON and the project's `rusteal.toml`, and writes a library's bindings:

- `schema.rs` deserializes the JSON.
- `config.rs` reads `rusteal.toml`: which engine modules to bind, under which Cargo feature, and what to block.
- `filter.rs` decides what can be generated.
- `type_map.rs` maps engine types to Rust types.
- `naming.rs` turns Unreal names into Rust names.
- `rust_gen/` writes the **bindings crate**: one Rust type per class, struct and enum; a typed method per function; accessors per property; and the crate's `Cargo.toml`, with one feature per engine module.
- `cpp_gen/` writes the **C++ wrappers** and the function table the plugin hands to the library.
- `manual/` holds hand-written extensions copied verbatim into every bindings crate: math conversions, collision helpers, world and widget helpers, and Enhanced Input helpers. Put any API that reflection cannot describe here, such as a C++ template.

Generation only rewrites files whose contents change, so an unchanged engine does not trigger rebuilds in UBT or Cargo. `tests/manual_compiles.rs` generates bindings from fixtures and compiles them, catching codegen regressions without an engine.

### `rusteal` (`rusteal-cli`)

The **command-line tool**, published as `rusteal`. It is what game developers install, and it orchestrates everything above:

- `rusteal new` creates a UE project from a template and builds it;
- `rusteal setup` installs the two plugins into an existing project;
- `rusteal build` runs the five-step pipeline, or part of it with `--step` or `--from`;
- `rusteal generate` runs codegen alone;
- `rusteal package` builds every library in release and cooks the game;
- `rusteal upgrade` moves a project to the CLI's version;
- `rusteal plugin new` and `rusteal plugin package` create and distribute Rusteal plugins.

Its build script **embeds the UE plugins and the templates into the binary**, so the CLI carries everything it writes into a project. In the workspace it reads `ue_plugin/` directly; the published crate falls back to a snapshot produced by `rusteal sync-plugin`.

- `templates/` holds the game templates: blank, first person, third person, top down, vehicle, intro, AR and VR, with their variants (see [Template directories](../templates/directories.md)).
- `plugin_templates/` holds the plugin templates: blank, editor and inventory, and the gameplay features packaged as plugins: mannequin, checkpoints, interactables and tour (see [Features](../templates/features.md)).
- `project_features/` holds the features that add files to a project instead of a plugin: rustrover.

### `xtask`

Contributor tooling, never published and never used by games. `cargo xtask dev-setup` prepares a checkout for working on Rusteal itself:

1. reads the engine path from `.env`;
2. asks for the UE project Rusteal is developed against;
3. writes `compile_commands.json` for clangd, re-targeted at `ue_plugin/`;
4. writes the exporter's `.csproj.props`;
5. restores `rusteal.sln` with the `dotnet` on the `PATH` (a .NET SDK 10 or newer).

## What a game project contains

The generated code never enters the repository. It lives in each game project, next to the game's own crate:

```
MyGame/
├── MyGame.uproject
├── rusteal.toml                         which engine modules to bind
├── Plugins/
│   ├── Rusteal/                         copied from ue_plugin/Rusteal
│   │   ├── Source/Rusteal/Generated/    C++ wrappers (codegen)
│   │   └── Binaries/<Platform>/librusteal.so
│   ├── RustealGenerator/                copied from ue_plugin/RustealGenerator
│   └── Inventory/                       a Rusteal plugin, with its own Rust/ and library
└── Rust/
    ├── Cargo.toml                       workspace: the game crate and the bindings
    ├── bindings/                        bindings crate (codegen)
    └── my_game/                         the game crate (cdylib), on rusteal-runtime
```

Each Rusteal plugin repeats that shape inside its own directory: a C++ module with its own `Generated/` wrappers, a `Rust/` workspace with its own bindings crate, and its own library.

## Examples

The repository has no examples directory: `examples/gem_collector`, the original demo, was removed in 0.7.0. The reference code for the runtime API is the templates in `rusteal-cli/templates/`, which every change has to keep building and playing.
