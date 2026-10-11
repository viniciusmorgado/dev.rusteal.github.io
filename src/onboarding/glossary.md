# Glossary

| Term | Meaning |
|---|---|
| **API table** | `RustealApiTable`: the struct of function pointers the plugin hands each Rust library, the plugin's services (`rusteal-ffi/src/api_table.rs`, `RustealApiTable.h`). |
| **Bindings crate** | The `bindings` crate codegen writes into a game project's `Rust/bindings/` (or a plugin's): one Rust type per engine class, struct and enum, committed with the game. |
| **Callbacks** | `RustealRustCallbacks`: the seven functions a library returns from `rusteal_init`, which the plugin calls into Rust. |
| **CDO** | Class default object: the object holding a class's defaults, which `#[class_defaults]` edits. |
| **Checked** | `Checked<T>`: a `UObjectRef<T>` that was checked valid, the type generated instance methods are implemented for. |
| **Codegen** | `rusteal-codegen`: reflection JSON to bindings crate and C++ wrappers (step 2 of the build). |
| **Current library** | The library whose code is running, set before every call into Rust, so what the API creates is tagged with its owner. |
| **Deploy** | Step 5 of the build: copying a built library to where the plugin loads it, by `.partial` and rename. |
| **Exporter** | `RustealExporter`, the C# UHT plugin writing the reflection JSON. |
| **Feature** | A piece of gameplay any template can take (a plugin template with a `[feature]` table), or a set of project files (`project_features/`). Not to be confused with a Cargo feature, which gates an engine module in the bindings. |
| **Func id / function table** | The number codegen gives each wrapped engine function, and the table of wrapper pointers indexed by it. |
| **Game library** | The game's Rust `cdylib`, deployed as `Plugins/Rusteal/Binaries/<Platform>/librusteal.so`, classes in `/Script/Rusteal`. |
| **Hot copy** | `<stem>_hot_<N>.<ext>`: the numbered link or copy of a deployed library the plugin actually loads. |
| **Hot reload** | Swapping a library while the editor runs (`Rusteal.Reload`, automatic on deploy). |
| **Library** | A Rust shared library the plugin hosts: the game's or a Rusteal plugin's. |
| **`manual/`** | Hand-written extensions codegen copies into every bindings crate, for what reflection cannot describe. |
| **Pinned** | `Pinned<T>`: a GC root that keeps an object alive for Rust. |
| **Plugin library** | A Rusteal plugin's own library, `librusteal_<Plugin>.so`, classes in `/Script/<Plugin>`. |
| **Reify / reified class** | Creating a Rust type as a real Unreal type at load time; `URustealReifiedClass` is the class behind a `#[uclass]`. |
| **Reinstance** | Moving existing objects and Blueprints to a replaced class, with the engine's `FReload`. |
| **Rust data** | An object's Rust-private fields, kept by the registry outside the object's memory. |
| **Shape** | The compile-time hash of a class's declarations that decides, on hot reload, whether it is kept or replaced. |
| **Stub Blueprint** | `RS_<Class>`, the editor-only `UBlueprint` a reified class names as its origin. |
| **Template / variant** | A Rusteal template is an engine template with its gameplay in Rust; a variant (`combat`, `horror`...) is one of its genre starting points, a complete project of its own; `base` is the template alone. |
| **Test project** | A project made with `--runtime-path`, following a checkout, where engine-facing changes are tested. |
| **UHT / UBT / UAT** | The Unreal Header Tool (reflection), Build Tool (compiles C++), Automation Tool (cooks and packages). |
| **Wrapper** | A generated `extern "C"` C++ function that calls one engine function natively. |
