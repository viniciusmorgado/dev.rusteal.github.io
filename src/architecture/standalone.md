# Standalone editor plugins

`rusteal plugin package <Name> --standalone` (`rusteal-cli/src/standalone.rs`) turns an editor plugin into one that works in any project, Blueprint and C++ ones included, installed in the engine as plugins from Fab are. How it is used is in the game developer's handbook, [Editor tools for any project](https://viniciusmorgado.github.io/game.rusteal.github.io/plugins/standalone.html).

## Why the runtime is copied, and renamed

An editor plugin needs the Rusteal plugin's C++: the API table, the classes Rust creates, library loading. A project without Rusteal has no such plugin, and a plugin on Fab cannot depend on a plugin made by someone else. So the plugin carries a copy of it, as modules of its own.

The copy is **renamed** with the plugin's name as a prefix: two standalone plugins, or one and a Rusteal project's own plugin, may be loaded in one editor. Unrenamed, they would define the same C++ types (UHT refuses them, and the dynamic linker binds exported symbols of the same name to whichever library came first) and the same class names, which `FindFirstObject` (how Rust finds a class by its short name) would resolve to either copy.

## What the package is made of

From the plugin, in `<output>/<Name>-Source` (what a store that builds plugins itself takes) and, built by `BuildPlugin`, in `<output>/<Name>`:

1. **The plugin's files**, without `Rust/`, `rusteal.toml`, `Binaries/`, `Intermediate/`, `Saved/` and its `FilterPlugin.ini`.
2. **Its C++ module**, renamed (`rename_tree`): its `Module.cpp`, its `Build.cs` and its generated wrappers name Rusteal's types, headers and functions, which become the copy's.
3. **The runtime** (`vendor_runtime`): `Source/Rusteal` and `Source/RustealEditor` from the CLI's embedded copy of the plugin, renamed to `Source/<Name>Rusteal` and `Source/<Name>RustealEditor`, with stub generated code (it hosts no game library: `<NAME>RUSTEAL_HOSTS_GAME_LIBRARY=0`) and without the static-link module.
4. **The descriptor**: the two runtime modules first, all of type `Editor`; no dependency on `Rusteal` or `RustealGenerator`, `EnhancedInput` instead, which the runtime uses.
5. **The Rust library**: the plugin's own, built with the release profile by the project, copied into `Source/ThirdParty/RustLibraries/<Platform>/`. It is the same library the plugin loads in its project: the Rust code still names `RustealEditorSubsystem`, and the copy of the runtime finds it under its own name (below).

## The renaming

`rename` replaces `Rusteal` with `<Name>Rusteal` and `RUSTEAL` with `<NAME>RUSTEAL`, in file names and in C++, C#, header and text files. Lowercase `rusteal` is left alone: it names the Rust libraries' files (`librusteal_<Library>`) and the functions they export (`rusteal_init`), which the copy loads as the original does. The module's API macro, `UCLASS` names, generated headers, console commands (`<Name>Rusteal.Reload`), log categories and message log follow.

Rust finds a type by its short name, through `reflection_find_class`, `reflection_get_static_class`, `reflection_find_struct` and `reflection_find_enum` (a parent class in `#[uclass(parent = ...)]` too). The copy is compiled with `<NAME>RUSTEAL_VENDORED_PREFIX=<Name>` and `<NAME>RUSTEAL_VENDORED_ORIGINAL=Rusteal`, which `OwnTypeName` (`RustealReflectionApiImpl.cpp`) uses to look up a name that starts with `Rusteal` as `<Name>Rusteal…`: its own types, never the ones of a Rusteal project's plugin loaded in the same editor. The defines are added after the renaming, so their values stay as written. It also gets `<NAME>RUSTEAL_PLUGIN_VERSION`, the version it was copied from, since there is no `Rusteal` plugin to read it from.

## Where the Rust library lives

A plugin's library is looked for in `Binaries/<Platform>/` first, then in `Source/ThirdParty/RustLibraries/<Platform>/` (`RustealRegisterPluginModuleLibraryChecked`). Fab takes a plugin's sources with no `Binaries/` and builds it itself, so a prebuilt library goes into the sources, where `BuildPlugin` packages it by default.

## Testing it

Unit tests cover the renaming, the lookups and the copy's layout. The full check, from the plugin's project:

1. `rusteal plugin package <Name> --standalone`: `BuildPlugin` must build the renamed copy with no errors.
2. The package in `Engine/Plugins/Marketplace`, enabled in a Blueprint project, a C++ project made from the engine's Blank template, and a Rusteal project; two standalone plugins at once. Each library logs `Rust side initialized` with its own prefix.
3. The Blueprint project packaged: nothing of the plugin or of Rust in the game.
