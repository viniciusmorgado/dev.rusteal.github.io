# Hot reload

A Rust library is swapped while the editor runs: save a file, `rusteal watch` builds and deploys it, the editor reloads it, and the objects in the level carry on with the new code. This chapter is how the pieces fit, with the files to read for each. The repository's [`docs/hot-reload.md`](https://github.com/viniciusmorgado/rusteal/blob/main/docs/hot-reload.md) keeps the longer notes: shapes in detail, the timings, and what was and was not tested.

## The loop

```
save a .rs file
  └─ rusteal watch (polls every 400 ms, settles 300 ms)
       └─ rusteal build: cargo rustc (cdylib), then deploy
            └─ copy to librusteal.so.partial, rename over librusteal.so
                 └─ RustealAutoReload (DirectoryWatcher, quiet for 0.5 s)
                      └─ RustealReloadLibrary(Name)
                           ├─ 1. drop the Rust data of the library's objects
                           ├─ 2. unload, load librusteal_hot_<N>.so, rusteal_init
                           ├─ 3. reinstance the replaced classes (FReload)
                           ├─ 4. stop here if the load failed
                           └─ 5. construct the Rust data again
```

## Registering and loading a library (`RustealModule.cpp`)

- `StartupModule` fills the API table, registers a delete listener and the core ticker (`TickLibraries`, which calls each library's `on_tick`), and registers the game's library: the project's name, the package `/Script/Rusteal`, the file `<Project>/Plugins/Rusteal/Binaries/<Platform>/librusteal.<ext>`. A Rusteal plugin's C++ module registers its own with `RustealRegisterPluginLibrary` (`<Plugin>/Binaries/<Platform>/librusteal_<Plugin>.<ext>`, package `/Script/<Plugin>`).
- `RegisterLibrary` deletes the stale load copies (`<stem>_hot_*.<ext>`) a crash may have left, and broadcasts `RustealOnLibraryRegistered`, which the editor module listens to.
- `LoadLibrary` increments the library's reload count and makes the **load copy** `<stem>_hot_<N>.<ext>`: a hard link on Linux, a copy where that fails, on Windows (a loaded DLL's file stays locked) and on macOS (below), the deployed file itself if both fail; a packaged game (`!WITH_EDITOR`) loads the deployed file in place, with no load copy. It checks the copy's `rusteal_version` against the plugin's descriptor `Version`, calls `rusteal_init(&Table)` with that library current (`FRustealLibraryScope`) and keeps the callbacks it returns.
- `UnloadLibrary` calls the library's `on_shutdown` and `rusteal_shutdown`, forgets its pinned objects and console commands, frees the handle, and deletes its load copy. On macOS the library stays mapped: `dlclose` never unloads a Rust library there (an image with thread-local variables stays loaded), so each reload leaves the previous library in memory until the editor exits.

Loading a copy is what lets the build replace the deployed file at any time. It is also why the deploy step must replace the file (`.partial` then rename, a new inode) and never write into it: the load copy may be a hard link to the old inode, which the running editor has mapped.

macOS is why the load copy is not a link everywhere. The library a reload unloads is still loaded there, and `dlopen` hands back an image already loaded from the same file, the same inode, instead of loading it again. A hard link to an unchanged deployed file would reload the old library, already initialized: `rusteal_init` fails on `init_api called more than once`. A reload after a build was spared, the deploy writes a new inode, but `Rusteal.Reload` without a build was not; a copy gives every reload a new file.

## Reloading (`ReloadLibrary`)

Each phase is timed and logged:

1. `drop_rust_instance` on every object of a `URustealReifiedClass` owned by the library. The `#[uproperty]` fields live in the objects' memory and stay; the Rust-private fields are dropped and come back from `Default`.
2. Unload, then load the new copy, whose `rusteal_init` registers every class again.
3. `RustealReifyReinstanceReplacedClasses()`.
4. If the load failed, stop and log why.
5. `construct_rust_instance` on every object, saying whether it is a class default object.

`Rusteal.Reload [Names...]` (an `FAutoConsoleCommand` of the runtime module) reloads the named libraries, or all of them.

## Kept or replaced: the shape

`#[uclass]` and `#[uclass_impl]` compute a **shape** at compile time (`rusteal-macros/src/shape.rs`): an FNV-1a hash over the sorted declarations, the class's arguments, its `#[uproperty]` and `#[component]` fields and its `#[ufunction]` and `#[udelegate]` signatures. Method bodies, doc comments, private fields and the order of declarations are left out.

`CreateClassImpl` (`RustealReifyApiImpl.cpp`) finds the existing class by name:

- **same shape and parent**: the class is kept, with its new `RustTypeId` and `Library`. `AddFunctionImpl` keeps each existing `URustealReifiedFunction` and only gives it the new `CallbackId` and `Library`, so a call goes to the new code. This is the common case, a change to a function's body, and it costs nothing.
- **changed, with a reinstancer**: `RetireClass` cuts the old class and its functions from the library (`Library = nullptr`, so they do nothing), renames it and its stub Blueprint to `RUSTEAL_<Name>` in the transient package, and roots it; a new class is created and the pair is queued for reinstancing.
- **changed, without a reinstancer** (a `-game` process): the class is kept as it was, with a warning that only the editor can apply the change.

Structs and enums are not hot reloaded: `create_struct` and `create_enum` return the existing object and warn when its fields or values changed, which takes an editor restart.

## Reinstancing (`RustealEditorModule.cpp`)

The editor module registers `ReinstanceClasses` with `RustealSetClassReinstancer`:

1. `FReload Reload(EActiveReloadType::Reinstancing, "RUSTEAL", *GLog)`;
2. `NotifyChange(New, Old)` for each replaced class, then `Reinstance()`: the engine moves every object, Blueprint child and default object to the new class, as it does for Live Coding;
3. `CompileBlueprintsUsing`: the engine recompiles the replaced class's children, and Rusteal also recompiles the loaded Blueprints that only use it (a Level Blueprint calling one of its functions), which would otherwise call a function that no longer exists;
4. `Finalize(false)`, with garbage collection skipped, so a reload does not stall the editor.

`FReload` lives in UnrealEd, which is why reinstancing is the editor module's job and a `-game` process has none.

## Reloading on deploy (`RustealAutoReload.cpp`)

The editor module watches each library's deploy directory with the engine's `DirectoryWatcher`, for the libraries already registered (`RustealForEachLibrary`) and the ones registered later (`RustealOnLibraryRegistered`). Only a change to the deployed file itself counts: `.partial` and `_hot_` files are ignored. Once the file has been quiet for half a second (`SettleSeconds`), the ticker reloads that library and shows an editor notification, for 3 seconds on success and 6 on failure. `Rusteal.AutoReload 0` turns it off.

## Limits

| Change | Result |
|---|---|
| A function's body, a private field's use | The class is kept; the functions point to the new code. |
| A `#[uproperty]`, `#[component]`, `#[ufunction]` or `#[udelegate]` added, removed or changed | The class is replaced and reinstanced, its Blueprint children and users recompiled. |
| A `#[ustruct]` or `#[uenum]` changed | Kept as it was, with a warning: restart the editor. |
| A function a Blueprint uses removed | The Blueprint's node breaks, as it would with C++. |
| Private Rust fields | Reset to `Default` on every reload. |
| The C++ plugin, the bindings, `rusteal.toml` | Not reloadable: `rusteal build --all` and an editor restart. |

## Why not Live Coding

Live Coding is Windows-only and patches the machine code UBT compiled; a library Cargo built has nothing for it to patch. What Rusteal needs from it is the reinstancing, and that part, `FReload`, is exported from UnrealEd (`UNREALED_API`) and usable on every platform. The function table makes the code swap itself trivial: since every reified function calls through its library and callback id, loading the new library and pointing the functions at it is the whole swap.
