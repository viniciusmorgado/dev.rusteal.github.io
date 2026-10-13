# Code generation

The bindings are not written by hand: the engine describes itself through UHT's reflection, an exporter writes that description to JSON, and `rusteal-codegen` turns the JSON into a Rust crate and the C++ wrappers it calls. This chapter follows that pipeline, its rules, and what to do when it leaves something out.

```
UHT ──▶ RustealExporter (C#) ──▶ rusteal_{classes,structs,enums}.json
                                         │  copied to Intermediate/Rusteal/uht/
                                  rusteal-codegen
                     ┌───────────────────┴───────────────────┐
                     ▼                                       ▼
          Rust/bindings/ (Rust crate)        Plugins/Rusteal/Source/Rusteal/Generated/
          func_ids.rs, one module per        RustealFunc_<module>_<Class>.cpp,
          engine module, manual/, prelude    RustealFuncIds.h, RustealFillFuncTable.cpp,
                                             module_deps.txt
```

## The exporter (`ue_plugin/RustealGenerator/Source/RustealExporter/`)

`RustealExport.cs` is a plugin of the Unreal Header Tool: `[UnrealHeaderTool] static class RustealExport` with `[UhtExporter(Name = "Rusteal", ModuleName = "RustealGenerator")]` on `Generate`. `RustealExporter.ubtplugin.csproj` builds it for .NET into the plugin's own `Binaries/DotNET/UnrealBuildTool/Plugins/RustealExporter/`, a path relative to the `.csproj` (decision D6), with `EngineDir` taken from the per-machine `RustealExporter.ubtplugin.csproj.props` (`rusteal setup` writes it in a project, `cargo xtask dev-setup` in the checkout). The `RustealGenerator` module is an empty editor module: it only exists so UHT's manifest has a module for the exporter's output.

It walks every module of the UHT session and writes three indented JSON files next to the module's UHT output: `rusteal_classes.json`, `rusteal_structs.json`, `rusteal_enums.json`. Step 1 of `rusteal build` copies them to `<project>/Intermediate/Rusteal/uht/`.

**What it exports:**

- **Names**: `name` without the `A`/`U`/`F` prefix, `cpp_name` with it, `package` the module's short name.
- **Classes**: not deprecated, exported from their module (`MinimalAPI` or `RequiredAPI`), and either Blueprint types (up the super chain, `BlueprintType`/`BlueprintSpawnableComponent` win and `NotBlueprintType` stops) or with exposed members (a `BlueprintVisible`/`BlueprintAssignable` property, a `BlueprintCallable`/`BlueprintEvent` function). A class another exported type reaches (its super, a property's class) is exported too, with `reached: true`: properties but no functions.
- **Structs**: Blueprint types or with exposed fields, plus the ones reached by supers and properties. `has_static_struct` is false for `NoExport` structs, except a fixed list (`Vector`, `Rotator`, `Transform`...).
- **Enums**: unless `NotBlueprintType`. UHT stores `-1` for values that are not plain literals; `RustealEnumValues.cs` reads the header and evaluates the expression (literals, operators, earlier values, constants from the engine's third-party headers), and gives up on bodies with preprocessor lines.
- **Properties**: private and protected ones only when Blueprint-visible or editable; not deprecated, not editor-only. With their type, flags, `BlueprintGetter`/`BlueprintSetter`.
- **Functions**: plain functions, not network, private or delegate ones, protected ones only when `BlueprintCallable`, not deprecated, `Native` or `BlueprintEvent`. The return value is the last parameter; defaults come from `CPP_Default_<Param>` metadata.

When codegen needs a piece of reflection the JSON lacks, the exporter is what to change.

## `rusteal-codegen`

`run_generate(root)` generates the game's library from `<root>/rusteal.toml`; `run_generate_plugin(root, plugin)` a plugin's, from its own `rusteal.toml` but the **project's** JSON. Both call `generate_library`:

1. `config.for_library(kind)` (below);
2. read the three JSON files; build the `CodegenContext`;
3. `with_unlisted_classes`: a module mapping with `classes = [...]` keeps those classes and their ancestors within the module, and blocks the rest;
4. `filter::apply_filters`;
5. `build_func_table`: the ids of every wrappable function (see [The Rust–C++ boundary](boundary.md));
6. the Rust side: modules, `manual/`, the prelude, `Cargo.toml`;
7. the C++ side: wrappers, ids, the table;
8. `module_deps.txt` and the `.uplugin`'s engine plugin dependencies;
9. `verify_output`: contiguous ids, non-empty `lib.rs`, `func_ids.rs`, `RustealFuncIds.h` and `RustealFillFuncTable.cpp`, a `mod.rs` per enabled module; otherwise it exits with an error.

**Every file goes through `write_if_changed`**: unchanged output keeps its timestamp, so a full build with nothing new does not make UBT recompile the wrappers nor Cargo the bindings. Stale files are removed one by one, never by wiping the directory.

### `rusteal.toml` (`config.rs`)

```toml
[project]                 # [plugin] in a plugin, with kind = "runtime" | "editor"
crate = "my-game"

[codegen]
features = ["core", "engine", "input", "slate", "umg", "enhanced-input"]

[codegen.modules]
Engine = { module = "engine", feature = "engine" }
Niagara = { module = "niagara", feature = "niagara", plugin = "Niagara" }
UnrealEd = { module = "unreal_ed", feature = "editor", classes = ["EditorActorSubsystem"] }

[codegen.blocklist]
classes = ["MovieSceneNiagaraSystemSpawnSection"]
structs = ["ConstraintInstanceAccessor"]
functions = ["AnimMontage.IsValidAdditiveSlot"]
```

- The key of `[codegen.modules]` is the UHT module (the JSON's `package`); `module` is the Rust module; `feature` the Cargo feature it is generated under; `plugin` the engine plugin it comes from, added to the `.uplugin`; `classes` limits a large module to what is needed.
- A module is generated when its feature is in `features`. An unmapped package is never generated.
- `for_library` always adds Rusteal's own package (`Rusteal` → `rusteal`, on the first feature), `RustealEditor` → `rusteal_editor` for editor libraries, and blocks Rusteal's internal classes (`RustealReifiedClass`, `RustealReifiedFunction`, `RustealDelegateProxy`).
- Blocklist functions are `"Class.Function"`, the class without its prefix and the function's Unreal name.

### Filters (`filter.rs`)

- **Blocked classes** disappear; blocked structs are left out of what properties and parameters may use.
- **Properties** stay when their type is supported, their access allows it, and every class, struct, enum and interface they name is available.
- **Functions** are dropped when blocked, editor-only in a runtime library, not native, a `K2_X` whose `X` exists, or with an unsupported, delegate, nested-container or unavailable parameter. A class from a non-public header loses its functions, and a function naming a type from one is dropped. Remaining duplicate names in a class are numbered (`Name_1`).

### Types (`type_map.rs`)

| Unreal | Rust |
|---|---|
| `bool`, integers, `float`, `double` | `bool`, `i8`…`u64`, `f32`, `f64` |
| `FString`, `FText` | `String` (UTF-8 pointer and length across the boundary) |
| `FName` | `FNameHandle` (`uint64_t` in C++) |
| enum | the generated enum, through its representation |
| object, soft object, weak object | `UObjectRef<Class>` |
| `TSubclassOf<T>` | `SubclassOf<T>` |
| interface | `UObjectRef<Interface>` |
| struct | opaque bytes; `OwnedStruct<FName>` in the API when the struct has a static struct |
| `TArray`, `TMap`, `TSet` | `UeArray`, `UeMap`, `UeSet` as properties; `&[T]` in and `Vec<T>` out as parameters; no nesting |

`LazyObjectProperty`, `SoftClassProperty`, `FieldPathProperty` and plain multicast delegates are unsupported. Parameters are in, out, or in-out by their flags (`OUT|CONST` is in, `OUT|REF` in-out); out scalars come back as a tuple, and a parameter with a parseable C++ default becomes an `Option`.

### Names (`naming.rs`)

- `to_snake_case`: `HTTPServer` → `http_server`, `K2_GetActorLocation` → `k2_get_actor_location`.
- A bool property loses its `b` (`bHidden` → `hidden`).
- `self`, `Self`, `super`, `crate` get a `_` suffix; other keywords become raw identifiers (`r#type`); a type file named after a keyword uses `#[path]`.
- Collisions: numbered functions, first property wins, a setter is dropped when a function of that name exists or the property has a `BlueprintSetter`.
- Enum variants drop `_MAX`, duplicates and the `Enum::` prefix; an enum with no variant left becomes a newtype.

### The bindings crate (`rust_gen/`)

`Rust/bindings/` (or `<Plugin>/Rust/bindings/`): package `bindings`, edition 2024, on `rusteal-core`, `rusteal-ffi` and `glam`; `[features]` with every enabled feature on by default, each listing the features its modules reference; a README saying the crate is generated and committed with the game.

- `src/lib.rs`: `func_ids`, one `#[cfg(feature)]` module per engine module, `manual`, `prelude`.
- One file per type. A class is a unit struct with `UeClass` (`static_class` through reflection), `HasParent`, `Inherits` for every ancestor, and a `<Name>Ext` trait for `Checked<Name>` and `Pinned<Name>` with its property accessors (found by name once, `OnceLock`), delegate accessors and instance functions; static functions are inherent. Inherited members come from the parent's trait, not copies. Generated modules start with `#![allow(clippy::all)]`.
- A struct is an opaque `F<Name>` with a `F<Name>Ext` trait for `UStructRef<F<Name>>`; an enum is `#[repr]` with `UeEnum`.
- `src/prelude.rs` re-exports a fixed table per feature (`core`, `engine`, `umg`, `input`, `enhanced-input`).

**`manual/`** (`rusteal-codegen/manual/*.rs`) holds hand-written extensions for what reflection cannot describe: vector, quaternion, rotator, transform and color math (`core`), `FKey` (`input`), world, collision, net quantize and data table helpers (`engine`), widget helpers (`umg`), Enhanced Input helpers (`enhanced-input`). `build.rs` embeds them; codegen writes them into every bindings crate's `src/manual/`, each gated by its feature.

### The C++ side (`cpp_gen/`)

Written into the host module's `Generated/`:

- `RustealFunc_<module>_<Class>.cpp`: one `extern "C"` wrapper per function. A protected function goes through `struct FAccess : UClass { using UClass::Func; }`, a `BlueprintNativeEvent` calls `Func_Implementation`, an interface function casts to the interface, a static one calls the class. Strings are pointer and length (out strings, a buffer and its length), structs are copied as bytes, containers are passed as base and property.
- `RustealFuncIds.h` and `RustealFillFuncTable.cpp` (see [The Rust–C++ boundary](boundary.md)).
- `module_deps.txt`: `Core` and every enabled package except the host module, one per line; `Rusteal.Build.cs` adds them as public dependencies, a plugin's `Build.cs` as private ones.

`rusteal setup` writes stubs of these files before codegen ever ran, so the plugin compiles from the start.

## A plugin's library

Same pipeline, with `[plugin]` instead of `[project]`, the project's JSON as input, `<Plugin>/Rust/bindings` and `<Plugin>/Source/<Plugin>/Generated` as output, and every C++ symbol prefixed with `<Plugin>_`. An editor library (`kind = "editor"`) binds `RustealEditor` and keeps the editor-only functions. A plugin has one library, runtime or editor, not both.

## When something is missing or does not link

- **A function is silently absent**: a filter dropped it (type unsupported, not native, non-public header...). The Rust generator writes `// Skipped: Class.Func (...)` where it falls back. Either make the type supported, add a helper in `manual/`, or call it with `DynamicCall`.
- **A wrapper compiles but does not link**: the function has no `*_API` export in its module; the exporter only checks exports per class. Add `"Class.Function"` to `[codegen.blocklist] functions`, sorted, in every template's `rusteal.toml.tera` (`rusteal-cli/templates/*/*/` and `rusteal-cli/plugin_templates/*/`), and run `rusteal build --from 2` in the test project.
- **A wrapper does not compile**: block it the same way, with a comment saying why.

## Binding a new engine module

1. Add `Package = { module = "...", feature = "...", plugin = "..." }` to `[codegen.modules]`, and the feature to `features`, in the templates' `rusteal.toml.tera`.
2. `rusteal build --all` in a test project: the exporter sees every module of the session, codegen writes the module, the Cargo feature's dependencies, `module_deps.txt` and the `.uplugin` plugin entry.
3. Blocklist what does not link.
4. Optionally, helpers in `manual/` and prelude entries (`rust_gen/prelude.rs`).

## Tests

`rusteal-codegen/tests/manual_compiles.rs` runs codegen on a fixture of real UHT output (`tests/fixtures/uht/`) and `cargo check`s the result, without an engine:

| Test | Checks |
|---|---|
| `manual_compiles_against_generated_bindings` | The bindings and `manual/` compile with `core`, `engine`, `input`, `slate`, `umg`. |
| `enhanced_input_bindings` | The C++ wrappers' interface casts, `FAccess`, container returns and includes; the module is removed when its feature is. |
| `uclass_types_compile` | A game crate using every macro compiles against the bindings. |
| `regenerate_fixture` (ignored) | Rewrites the fixture from a project's JSON: `RUSTEAL_UHT_DIR=<project>/Intermediate/Rusteal/uht cargo test -p rusteal-codegen --test manual_compiles regenerate_fixture -- --ignored`. |

Unit tests cover the config, names, filters, properties and the table's symbol prefixes. Run them with `cargo test -p rusteal-codegen`.
