# Unreal Engine and Rust

## History of Rust and Unreal Engine

Over the years, many attempts have been made to make Rust a viable language for Unreal Engine. This chapter is a brief history of those attempts, and of how the community tested and validated different approaches until Rusteal.

### Linking Rust into a plugin (2015)

The earliest public experiment is [ue4-static-plugin](https://github.com/shadowmint/ue4-static-plugin), a sample UE4 plugin that builds a Rust library with Cargo and links it statically, calling it through `extern "C"` functions. It offers no bindings to the engine: Rust is a library that C++ calls into. What it proved is that the two toolchains can live in the same build.

### Rust as an Unreal module (2020)

In January 2020, a [proof of concept](https://users.rust-lang.org/t/proof-of-concept-rust-code-in-unreal-engine/36744) by ejmahler went further and made a Rust crate a module of the UnrealBuildTool. C++ modules depended on it as on any other module, headers were generated with `cbindgen`, and the Rust code was packaged and debugged with the game. The integration lived inside the engine's own source, so it was shared only with those who had access to Epic's repository, and require engine re-compilation.

### unreal-rust (2022)

The best-known attempt is [unreal-rust](https://github.com/MaikKlein/unreal-rust) by Maik Klein, an engineer at Embark Studios, who built it in his spare time and [announced it in 2022](https://forums.unrealengine.com/t/announcing-unreal-rust-a-rust-integration-for-unreal-engine-5/641990) for Unreal Engine 5.0. It was a plugin that required no engine changes, and it took an opinionated path: gameplay was written on an Entity Component System of its own, next to Unreal's actors, with hand-written bindings for the essentials (sounds, spawning actors, pathfinding, physics), hot reload, and Rust components that could be added to actors in the editor.

The project drew thousands of stars and showed how much interest there was in Rust for Unreal. It described itself as a proof of concept, though, and its last commit dates from October 2022. Forks such as [elodin-sys/unreal-rust](https://github.com/elodin-sys/unreal-rust), which started on macOS support, and [open-dx/unreal-rs](https://github.com/open-dx/unreal-rs) carried it a little further before stopping as well.

> **A note on Embark.** [Embark Studios](https://www.embark-studios.com/) is an interesting case in this story. The studio ships its games, *The Finals* and *ARC Raiders*, on Unreal Engine, and it is also one of the major supporters of Rust for games and graphics, with more than forty open source Rust projects of its own (tracked in [rust-ecosystem](https://github.com/EmbarkStudios/rust-ecosystem)), among them:
>
> - [rust-gpu](https://github.com/EmbarkStudios/rust-gpu), which makes Rust a first-class language for GPU shaders;
> - [kajiya](https://github.com/EmbarkStudios/kajiya), a real-time global illumination renderer written in Rust and Vulkan;
> - [puffin](https://github.com/EmbarkStudios/puffin), an instrumentation profiler for Rust;
> - [cargo-deny](https://github.com/EmbarkStudios/cargo-deny), a Cargo plugin for managing large dependency graphs;
> - [physx-rs](https://github.com/EmbarkStudios/physx-rs), Rust bindings for NVIDIA PhysX;
> - [texture-synthesis](https://github.com/EmbarkStudios/texture-synthesis), example-based texture generation.

### Smaller efforts (2024–2026)

Other projects explored the same space with different scopes. [unreal_rs](https://github.com/ProjectKML/unreal_rs) worked on Rust bindings for Unreal Engine 5 between late 2024 and early 2025. [UERust](https://github.com/kyunghoon/uer), in 2026, chose to be "plumbing" rather than a framework: a command-line tool that generates the FFI layer from a configuration file and reloads the Rust library in the editor.

### Uika (2026)

[Uika](https://github.com/VioletHelianthus/uika), by [VioletHelianthus](https://github.com/VioletHelianthus), took a different approach from everything before it. Instead of writing bindings by hand or building a framework beside the engine, it reads the engine's own reflection:

- a plugin for the Unreal Header Tool exports every reflected class, struct, enum and function to JSON;
- a code generator turns that data into Rust bindings and the C++ wrappers they call;
- Rust structs become real `UClass`es, which the editor, Blueprints and the rest of the engine see as their own;
- the Rust code compiles to a library loaded by a small C++ plugin, with every call going through a function pointer table, so iterating on Rust needs no C++ build, and the library can be hot reloaded.

Uika targeted Unreal Engine 5.7 and was released in February 2026. A few months later, its author announced that the project would no longer be maintained.

### Rusteal

Rusteal is a hard fork of Uika, and it stands on the foundation VioletHelianthus designed: the reflection-driven code generator, the exporter plugin, the reification of Rust structs as `UClass`es, and the runtime. From there, Rusteal develops the Unreal bindings independently, with the goal of making Rust a language you can ship games and plugins with.

### Building on what came before

The earlier attempts are not dead ends: they validated techniques that Rusteal can reuse. The clearest case is static linking, the approach of the first experiments, and it is the path to consoles.

During development, Rusteal loads the Rust code as a dynamic library, which is what makes hot reload possible. Consoles generally do not allow a game to load native code at runtime, and their toolchains are C++ only: no platform holder ships a Rust compiler. Both facts point to the same answer. The Rust code does not need to be built by the console's toolchain, only linked by it.

The consoles run on architectures and systems Rust already targets: the PlayStation's operating system derives from FreeBSD on x86-64, and the Xbox runs on Windows. Rust can compile the game's crate for the closest of those targets and stop before linking, emitting object files or LLVM bitcode, the same intermediate form the consoles' Clang-based compilers work with. The Unreal Build Tool then takes those files as part of the Rusteal plugin's module and links the engine, the plugin and the Rust code into one static game binary with the platform's own linker, as it does for any monolithic build.

Rusteal's design already fits this path. Every call between the engine and Rust goes through a function pointer table, so with statically linked Rust the plugin fills the table by calling the Rust entry point directly instead of looking it up in a loaded library, and the generated bindings stay the same. Two points need care on each platform: the calls the Rust standard library makes into the operating system must resolve against the platform's C library, and bitcode has to match the LLVM version of the platform's compiler, which object files avoid.

This may also need a small change to the engine's build tooling for each console. Console code is covered by the platform holders' agreements and cannot be published openly, so the project would provide that change, with its documentation, only to developers who show they have signed Epic's console access agreement and the platform holder's.
