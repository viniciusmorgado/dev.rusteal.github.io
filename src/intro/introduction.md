# Introduction

Rusteal integrates the [Rust programming language](https://rust-lang.org/) into Unreal Engine, so you can write both gameplay code and plugins in Rust.

## How it Works

Rusteal lets Rust code take part in Unreal Engine as a first-class language: it calls the engine's API, and it defines classes the engine, the editor and Blueprints treat as their own. This chapter explains how. The design comes from [Uika](https://github.com/VioletHelianthus/uika), and its core idea is simple: **Rusteal does not describe the engine by hand, it reads the engine's own reflection.**

## The engine already describes itself

Every class, struct, enum, property and function that Unreal exposes to Blueprints is declared with macros such as `UCLASS`, `USTRUCT`, `UPROPERTY` and `UFUNCTION`. During the build, the Unreal Header Tool (UHT) reads those declarations and produces the reflection data that the editor, Blueprints, serialization and networking all rely on.

That reflection data is a complete, machine-readable description of the engine's scripting API. Rusteal uses it as its single source of truth: whatever Blueprints can reach, Rust can reach, for the exact engine version installed and the engine plugins the project enables (Enhanced Input, StateTree, Niagara, Gameplay Abilities, and so on).

## From reflection to bindings

Rusteal turns that description into Rust code in three stages.

```
Unreal headers ──UHT──▶ RustealExporter ──▶ reflection JSON
                                                  │
                                         rusteal-codegen
                                         ┌────────┴────────┐
                                         ▼                 ▼
                              Rust bindings crate    C++ wrappers
                              (typed Rust API)       (one per engine function)
```

1. **Export.** `RustealExporter` is a plugin for the UHT itself, written in C#. It runs inside the engine's build and writes every reflected type to JSON: names, flags, properties, function signatures, inheritance and the header each type lives in.
2. **Codegen.** `rusteal-codegen` reads that JSON and generates two things:
   - a **Rust bindings crate**, with a Rust type for every engine class, struct and enum, and a typed method for every function;
   - a set of **C++ wrappers**: one small `extern "C"` function per engine function, which calls the real C++ function natively.
3. **Compile.** The C++ wrappers are compiled into the Rusteal plugin by the Unreal Build Tool, and the game's crate is compiled by Cargo against the bindings.

Because everything is generated, the bindings follow the engine: a new engine version or a newly enabled engine plugin is a regeneration away, not a rewrite.

## Crossing the boundary

The Rust code is compiled into a shared library (`librusteal.so`, `librusteal.dylib`, `rusteal.dll`) that the Rusteal plugin loads at startup. The two sides talk through a single, versioned contract defined in `rusteal-ffi`: plain `#[repr(C)]` types and **tables of function pointers**.

- **The API table** holds the plugin's own services: object lifetime, properties, delegates, containers, input, worlds, and the reification of Rust classes. The plugin hands it to the library when it loads it.
- **The function table** holds a pointer to every generated C++ wrapper. Calling `actor.k2_get_actor_location()` in Rust looks up the wrapper in that table and calls it; the wrapper casts the object and calls `AActor::K2_GetActorLocation` directly, with no interpreter and no `ProcessEvent` in between.
- **Properties** are reached through reflection: the property is found once by name, and its offset is reused afterwards.

Engine errors become Rust values: a call on an object the garbage collector has destroyed returns an error instead of crashing, and Rust code handles it with `Result`, like any other failure.

Since every call goes through these tables, iterating on Rust code never requires compiling C++: only Cargo runs.

## Rust classes the engine sees

Calling the engine is half of the story. The other half is the engine calling Rust.

A Rust struct marked with `#[uclass]` becomes a real `UClass`. When the library loads, Rusteal **reifies** it: it creates the class at runtime, with its parent (`Actor`, `Character`, `GameModeBase`, a subsystem…), its `#[uproperty]` fields as real properties, its `#[component]`s as default subobjects, and its `#[ufunction]`s as real functions whose native entry point calls back into Rust.

To the engine, that class is indistinguishable from one declared in C++ or in a Blueprint:

- it appears in the editor and can be placed in a level;
- a Blueprint can derive from it and set its properties as class defaults;
- its functions can be called from Blueprints, overridden as events (`ReceiveBeginPlay`, `ReceiveTick`…), or bound to delegates and input actions;
- it can be saved, configured from ini files, and cooked into a packaged game.

The Rust data of each object lives on the Rust side, next to the engine object, and is created and dropped with it.

## Hot reload

Because the engine only knows the library through its tables, the library can be swapped while the editor runs. `Rusteal.Reload` unloads it, loads the new build, and points every reified function at the new code: a change to a function body is live in seconds, without restarting the editor or touching C++.

## Games and plugins

The same machinery builds **Unreal Engine plugins** as well as games. A project has one Rust library for the game and one more for each Rusteal plugin, and the Rusteal plugin hosts all of them side by side:

```
Plugins/Rusteal/Binaries/<Platform>/librusteal.so                  → /Script/Rusteal
Plugins/Inventory/Binaries/<Platform>/librusteal_Inventory.so      → /Script/Inventory
Plugins/LevelTools/Binaries/<Platform>/librusteal_LevelTools.so    → /Script/LevelTools
```

Each library has its own bindings crate, its own C++ wrappers and its own tables, and its classes live in its own package, exactly as a C++ plugin's do. When the engine calls into Rust, the Rusteal plugin first selects the library that owns the class, so each library sees only its own objects and can be rebuilt and reloaded on its own.

A plugin is one of two kinds:

- a **runtime** plugin ships with the game: gameplay systems such as an inventory, built on actors, components and subsystems;
- an **editor** plugin only runs in the editor: tools built on editor subsystems, menus, asset validators and the editor scripting API, with bindings to the editor's own modules.

A plugin carries its Rust code, bindings and binaries in its own directory, so it can be copied into another Rusteal project or kept in a repository of its own.

## What it adds up to

Reflection gives Rusteal the whole engine API without writing it by hand, reification lets Rust classes stand next to C++ and Blueprint ones, and the function tables keep the boundary fast and swappable. The result is a workflow where Rust code is compiled by Cargo alone, reloaded in seconds, and shipped as games or as plugins.
