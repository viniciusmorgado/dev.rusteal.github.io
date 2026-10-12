# The Rust–C++ boundary

Everything that crosses between a Rust library and the engine goes through two tables of function pointers and seven callbacks, all `#[repr(C)]`. This chapter is that contract: how the plugin loads a library, what each side hands the other, how errors and panics cross, and what to keep in mind before changing any of it. "Private/" and "Public/" below are `ue_plugin/Rusteal/Source/Rusteal/Private/` and `.../Public/`.

## Loading a library (`Private/RustealModule.cpp`)

The `Rusteal` module is a `Runtime` module with the `Default` loading phase (`Rusteal.uplugin`). `StartupModule`:

1. **`FillApiTable`**: zeroes the static `GApiTable`, sets its `version` to the plugin descriptor's integer `Version` (`RustealPluginVersion`), and points its twelve sub-tables at the globals `GCoreApi`, `GPropertyApi`...
2. registers the component list resync and the reify delete listener;
3. adds the core ticker (`TickLibraries`, which calls every library's `on_tick`);
4. **`RustealFillFuncTable()`**, from the generated `RustealFillFuncTable.cpp`: the engine functions' table (below);
5. registers the game's library: the project's name, the package `/Script/Rusteal`, the file `<ProjectPluginsDir>/Rusteal/Binaries/<Platform>/<prefix>rusteal.<ext>` (`librusteal.so`, `librusteal.dylib`, `rusteal.dll`). `RustealLibraryFileName` names it: the prefix is `lib` on every platform but Windows, the names `rusteal build` deploys and `Rusteal.Build.cs` stages, and the extension the engine's `GetModuleExtension()`. It does not use `FPlatformProcess::GetModulePrefix()`: only the Unix platform overrides it with `lib`, it is empty on Mac, and the plugin looked for `rusteal.dylib` there.

A Rusteal plugin's own C++ module (from the plugin templates' `{{plugin}}Module.cpp.tera`) calls `<Plugin>_RustealFillFuncTable()` and `RustealRegisterPluginLibraryChecked("<Plugin>", <Plugin>_RustealGetFuncTable(), <Plugin>_RustealGetFuncCount(), <Plugin>_RustealGetFuncFingerprint())` when it starts (plugins made before 0.9 call `RustealRegisterPluginLibrary`, without the fingerprint, which still works), and `RustealUnregisterLibrary` when it stops: its library is `<Plugin>/Binaries/<Platform>/librusteal_<Plugin>.<ext>`, its package `/Script/<Plugin>`. The game's library loads first, each plugin's when its module starts.

**`RegisterLibrary`** refuses a name already loaded, makes or reuses the `FRustealLibrary` entry (heap-allocated, so its address is stable), deletes stale `_hot_` copies, **copies `GApiTable` into the library's own `Table`** and sets that copy's `func_table`, `func_count` and `func_fingerprint`, loads the library, and broadcasts `RustealOnLibraryRegistered`.

**`LoadLibraryFile`** looks up these exports (`Public/RustealApiTable.h` has their typedefs):

| Export | Signature | Required |
|---|---|---|
| `rusteal_version` | `uint32()` | no (missing reads as 0, which fails the check) |
| `rusteal_init` | `const FRustealRustCallbacks*(const FRustealApiTable*)` | yes |
| `rusteal_callbacks` | `const FRustealRustCallbacks*()` | no: lets callbacks fired during `rusteal_init` reach the library |
| `rusteal_shutdown` | `void()` | no, looked up when unloading |

`rusteal_version()` must equal the table's `version`, or the library is refused with a message asking for `rusteal build --all`. `rusteal_init` receives the library's own table and runs with the library current; a null return means failure, and the library is freed.

On the Rust side, `rusteal_runtime::entry!()` (short for `entry!(bindings)`, the crate holding the library's bindings) defines two statics at the crate's root and, in a dynamic library, the four exports:

- `__RUSTEAL_MODULE`, a `rusteal_core::Module`: the library's crate name, its source root (for error locations) and its bindings' function table;
- `__RUSTEAL_EXPORTS`, a `RustealModuleExports` (`version`, `init`, `shutdown`, `callbacks`), whose functions and seven callbacks are generated in the crate, each entering the module before it calls into the runtime;
- `rusteal_init`, `rusteal_shutdown`, `rusteal_version` and `rusteal_callbacks`, which forward to `__RUSTEAL_EXPORTS`. With `rusteal-runtime`'s `static-link` feature they are not generated (see [Static linking](#static-linking)).

`init` (`rusteal-runtime/src/lib.rs`) checks the table and its version again, stores the table pointer (`rusteal_core::api`, kept from the first library: the services are the same for every library), starts the module (refused if it is already running), records the game thread, installs the panic hook and the `log` logger (once per runtime), **binds the bindings' function table** (below), registers the module's types (see [Reification](reification.md)), runs its `on_load!` hooks and returns its callbacks. A failure after the start, or a panic, shuts the module down again before returning null.

**Versions** are encoded as `major * 1_000_000 + minor * 1_000 + patch` (`rusteal_ffi::encode_version`, a `const fn` that refuses minor or patch ≥ 1000 and pre-release suffixes at compile time). Exact equality is the only compatibility check: the table has no size or field count.

**`UnloadLibrary`** calls `on_shutdown` and the `rusteal_shutdown` export (both run the module's shutdown; the module's state makes the second one a no-op, and a shutdown after a partial start clears what was registered), forgets the library's pinned objects and console commands, frees the handle and deletes its load copy. On macOS freeing the handle does not unload the library: `dlclose` never unloads a Rust library there (an image with thread-local variables stays loaded), so its code stays mapped until the editor exits. `ShutdownModule` removes the ticker and unloads every library in registration order. Reloading is in [Hot reload](hot_reload.md).

## The current library (`Private/RustealLibrary.h`)

`FRustealLibrary` holds a library's name, package, file paths, reload count, handle, callbacks and its own `FRustealApiTable`. The **current library** is a plain static (`GCurrentLibrary`), set by the RAII `FRustealLibraryScope`. Every call from C++ into Rust goes through `RustealCallLibrary(Library, Fn)`, which skips an unloaded library and sets the scope: tick, construct, drop, invoke a function, a delegate, a console command, a pinned notification.

The API implementations Rust calls read `RustealCurrentLibrary()` to tag what they create with its owner: reified classes and functions, delegate proxies, console commands, pinned objects. Later, a call into one of those goes back to the library stored on it.

The Rust side mirrors it with **modules** (`rusteal-core/src/module.rs`). Each library's callbacks enter its `Module` (a thread-local "current module", restored on return), and what Rust code registers while a module is current belongs to it: tick hooks, game-thread jobs and background tasks (`task.rs`), delegate and console closures (`delegate_registry.rs`), pins (`pinned.rs`), types, function callbacks and instance data (`reify_registry.rs`). Ids stay global and unique (type ids, function callback ids, delegate ids), so libraries that share one runtime cannot collide; a module's shutdown waits for its own tasks and clears only its own entries. In a dynamic library there is one module per runtime, so this changes nothing; in a static build every library shares one runtime (see [Static linking](#static-linking)), and the modules are what keep them apart. Code running on a thread Rusteal did not start, with no module current, is attributed to the first module (the game's). `RustealCurrentPackage()` gives the package classes, structs and enums are created in: the current library's, `/Script/Rusteal` by default, created rooted with `PKG_CompiledIn`.

## The API table

Two hand-written mirrors, field by field in the same order: `rusteal-ffi/src/api_table.rs` (`#[repr(C)]` structs of `unsafe extern "C" fn` fields) and `Public/RustealApiTable.h` (C structs of function pointers). `RustealApiTable` is `version: u32`, twelve sub-table pointers, `func_table`, `func_count` and `func_fingerprint`, then the `platform` sub-table, appended last.

| Sub-table | Services |
|---|---|
| `core` | Validity, name, class, `IsA`, outer, `FName`, weak pointers, `take_error_message`, `is_child_of` |
| `logging` | `log(level, utf8)`, `report(RustealErrorReport*)` |
| `lifecycle` | GC roots, pinned objects |
| `property` | Typed get and set (bool, integers, floats, string, name, object, enum, struct), soft object paths |
| `reflection` | Finding classes, properties, structs, functions and enums; parameter buffers and `call_function`; offsets and sizes; struct init, destroy and copy; delegate signatures |
| `container` | `TArray`, `TMap`, `TSet` operations and bulk copies |
| `delegate` | Bind, unbind, multicast add, remove and broadcast, `add_function` |
| `reify` | Creating classes, structs and enums, and adding their members (see [Reification](reification.md)) |
| `world` | Spawning (deferred too), actors of a class, finding, loading and creating objects, the world, data table rows |
| `widget` | Creating widgets, root widget, widget tree |
| `input` | Enhanced Input action binding, touch interface |
| `console` | Commands and variables |
| `platform` | The engine's services the runtime uses instead of the operating system's: `launch_task(run, data)` (a background task of the engine's task system), `seconds`, `sleep`, `is_game_thread` (`RustealPlatformApiImpl.cpp`) |

Each C++ sub-table is a global filled by **positional** aggregate initialization (`FRustealCoreApi GCoreApi = {&IsValidImpl, ...}` in `RustealCoreApiImpl.cpp`, and likewise in each `Rusteal*ApiImpl.cpp`).

**`rusteal-core/build.rs`** copies `rusteal-ffi/src/api_table.rs` into `rusteal-core/api_table.rs` (gitignored, packaged with the crate, since a published crate cannot read `../rusteal-ffi`), parses it with `syn`, and writes `$OUT_DIR/ffi_dispatch.rs`: one `#[inline(always)] pub unsafe fn <table>_<field>(...)` per function pointer of every `Rusteal<Name>Api` struct, calling `((*crate::api::api().<table>).<field>)(...)`, plus `NativePtr` and unaligned `native_mem_read`/`native_mem_write`. Raw byte pointers become `NativePtr`; pointers to other `#[repr(C)]` structs named `Rusteal*`, `F*` or `U*` are written fully qualified (`*const rusteal_ffi::RustealReifyPropExtra`). A parameter of a type it does not know makes the generated wrapper fail to compile.

**Contract tests** check layouts at compile time: `rusteal-ffi/src/contract_tests.rs` (`const` asserts, always compiled, so `cargo build` fails) and `Private/RustealContractTests.cpp` (`static_assert`s): handles are 8 bytes, `RustealErrorCode` 4, `RustealReifyPropExtra` 40, `RustealConsoleArgs` 24, `RustealErrorReport` 72, `RustealApiTable` 136, `RustealModuleExports` 32, `RustealStaticModule` 24, with their field offsets. They do **not** check the order or the signatures of the function pointers: that is on the person changing the table.

### Adding an API function

1. Add the field to its sub-table in `rusteal-ffi/src/api_table.rs`, with types `build.rs` maps.
2. Add the function pointer at the same position in `Public/RustealApiTable.h`.
3. Implement it in the matching `Private/Rusteal*ApiImpl.cpp`, and add `&Impl` at the same position of the positional initializer. A field appended to the struct but missing from the initializer is silently null.
4. Return failures with `RustealFail(code, "reason")`, so Rust gets the engine's reason (below).
5. Add size and offset asserts on both sides for any new shared struct.
6. Call it from `rusteal-core` as `crate::ffi_dispatch::<table>_<field>(...)`, behind a safe function; nothing outside `rusteal-core` calls the dispatch.
7. Rebuild the plugin and the library together: the table changed but the version did not, so an old plugin with a new library is not detected.

## The callbacks (`rusteal-ffi/src/callbacks.rs`)

`RustealRustCallbacks`, mirrored in `RustealApiTable.h` and in the same order, is what `rusteal_init` returns. Each Rust implementation (`rusteal-runtime/src/lib.rs`) runs inside `ffi_boundary`.

| Callback | Called by |
|---|---|
| `construct_rust_instance(obj, type_id, is_cdo)` | `RustealClassConstructor`, once per reified class of the object's chain, root first; and after a hot reload. |
| `drop_rust_instance(obj, type_id, data)` | The UObject delete listener, once per library of the chain; and before a hot reload. |
| `invoke_rust_function(callback_id, obj, params)` | `URustealReifiedFunction::execCallRustFunction`. |
| `invoke_delegate_callback(callback_id, params)` | `URustealDelegateProxy::ProcessEvent`, and console commands (with `RustealConsoleArgs`). |
| `on_shutdown()` | `UnloadLibrary`: waits for background tasks, clears the registries. |
| `notify_pinned_destroyed(obj)` | The pinned objects' delete listener. |
| `on_tick(dt)` | The core ticker, every frame, for every library: runs the queued game-thread jobs and tick hooks. |

## The engine functions' table

Codegen gives every bound engine function an id: `build_func_table` (`rusteal-codegen/src/lib.rs`) sorts the functions it can wrap by module, class and name and numbers them from 0; `verify_output` refuses gaps. The ids are constants on both sides, named after the class and function in SCREAMING_SNAKE case: `RustealFuncIds.h` (`namespace RustealFuncId`) and the bindings' `func_ids.rs`.

- **The wrapper**, in `Generated/RustealFunc_<module>_<Class>.cpp`, is `extern "C" uint32_t <prefix>Rusteal_<Class>_<Func>(void* Obj, ...)`: it casts the object and calls the C++ function natively, returning `RustealErrorCode`. It checks `IsValid` and returns `ObjectDestroyed` only in Debug and Development builds. The prefix is empty for the game and `<Plugin>_` for a plugin, so plugins' symbols never clash.
- **The table**, `RustealFillFuncTable.cpp`, fills `GRustealFuncTable[FUNC_COUNT]` by id; `RegisterLibrary` puts it in the library's table, with `FUNC_COUNT` and `FUNC_FINGERPRINT`.
- **The fingerprint**: codegen hashes every entry of the table (id, module, class, function, flags, and each parameter's name, type, flags and class, struct, enum or interface), and writes the same value to `RustealFuncIds.h` (`FUNC_FINGERPRINT`, returned by `<prefix>RustealGetFuncFingerprint()`) and to the bindings' `func_ids.rs` (`FINGERPRINT`).
- **The binding**: each bindings crate owns its table, `pub static __RUSTEAL_FUNCS: rusteal_core::FuncTable` in its `lib.rs`, created with its `FUNC_COUNT` and `FINGERPRINT`. `init` binds it to the library's table: the counts must be equal, and the fingerprints too when the plugin gave one (zero means an old plugin module that did not); otherwise the library is refused with a message asking for `rusteal build --all`.
- **The call**, in the generated binding method: `const FN_ID: u32 = <id>`, `crate::__RUSTEAL_FUNCS.get(FN_ID)` (which panics past the table's end or before it is bound) transmuted to the wrapper's `extern "C"` type, called, its code checked with `ffi_infallible` (a `debug_assert`). Instance methods live on a `<Class>Ext` trait implemented for `Checked<Class>` and `Pinned<Class>` only, so the object was checked before the call.

Ids are positions: they change whenever a bound function is added or removed anywhere. Since each bindings crate reads its own table, the ids of two libraries never mix, even in one runtime; and since the table is checked when the library starts, bindings and C++ wrappers from different code generation runs are refused instead of calling the wrong functions.

## Handles and errors

**Handles** (`rusteal-ffi/src/handles.rs`): `UObjectHandle`, `UClassHandle`, `FPropertyHandle`, `UFunctionHandle`, `UStructHandle` are `#[repr(transparent)]` pointers; `FNameHandle` is a `u64` (comparison index low, number high); `FWeakObjectHandle` is `{object_index: i32, object_serial_number: i32}`, the bytes of an `FWeakObjectPtr` (`{-1, 0}` when null).

**Error codes** (`rusteal-ffi/src/error.rs`, `#[repr(u32)]`, mirrored by `ERustealErrorCode`): `Ok` 0, `ObjectDestroyed` 1, `InvalidCast` 2, `PropertyNotFound` 3, `FunctionNotFound` 4, `TypeMismatch` 5, `NullArgument` 6, `IndexOutOfRange` 7, `InvalidOperation` 8, `InternalError` 9, `BufferTooSmall` 10.

**The engine's reason** (`Private/RustealErrors.cpp`): `RustealFail(code, message)` stores `{code, message, GFrameCounter}` in a thread-local and returns the code. Rust's `check_ffi` turns a code into a `RustealError` and asks `core.take_error_message` for the reason, which answers only for the same code in the same frame; the message is cleared once fully read. See [Errors](errors.md).

**Strings** cross as UTF-8 pointer plus `u32` length, never NUL-terminated. A string coming back is written into the caller's buffer, with the full length returned, so the caller can retry with a bigger one.

**Validity.** `UObjectRef::is_valid` is `::IsValid` (not null, not garbage); `checked()` gives a `Checked<T>` or an `ObjectDestroyed` error. On the C++ side the property and container entry points only check for null, the delegate ones call `IsValid`, and the generated wrappers check only in Debug and Development: a Rust caller checks first.

## Panics (`rusteal-core/src/ffi_guard.rs`)

`ffi_boundary(default, f)` runs `f` under `catch_unwind`: a panic is reported (error level, as a script error, "Rust panic: ..." with the file, line and column the panic hook recorded) and the call returns `default`. The hook is chained to the one installed before. All seven callbacks use it; `init` and `shutdown` use `catch_unwind` directly (a panic in `init` returns null and the library is refused). Background tasks, game-thread jobs and tick hooks catch their own panics (`task.rs`). The callbacks are `extern "C"`, not `"C-unwind"`: a panic must never reach C++, and these guards are what ensure it.

## Threads

Nothing on the C++ side asserts the game thread; `task::is_game_thread` asks the engine (`IsInGameThread`). Engine calls belong on the game thread by convention: Rust code runs there when the engine calls it (functions, events, ticks, delegates), and `task::spawn` brings a background result back there through the queue `on_tick` drains. Handles are `Send + Sync`; `UObjectRef`, `Checked`, `Pinned`, `SubclassOf` and `TWeakObjectPtr` are `Send` but not `Sync`. `GCurrentLibrary` is not thread-local; the last error and the pending script error are.

## Static linking

A packaged game can link every Rust library into its executable instead of loading them (`rusteal package --static`, see [The CLI](cli.md#plugins-and-packaging-srcplugin_cmdrs-srcpackage_cmdrs)); the editor always loads them dynamically, with hot reload. The pieces:

- **The aggregate.** The CLI writes a Cargo workspace in `<project>/Intermediate/Rusteal/Static/` whose `archive` package is a `staticlib` depending on the game's crate and every runtime plugin's crate, with `rusteal-runtime`'s `static-link` feature, built with fat LTO and one codegen unit. Its `lib.rs` is one `rusteal_runtime::static_modules!` call, which generates `rusteal_static_modules(count)`, the archive's only export: an array of `RustealStaticModule { name, name_len, exports }`, one per library, named as C++ registers them (the project's name for the game, the plugin's name for a plugin), pointing at each crate's `__RUSTEAL_EXPORTS`. The same call tells each `Module` which crates it owns, since registrations from every library now share one `inventory`, and which registrations it must find.
- **Linking.** `Plugins/Rusteal/Source/ThirdParty/RustealStatic/` is an External module (`ModuleType.External`): it adds `lib/<Platform>/<archive>` with `PublicAdditionalLibraries` and the system libraries rustc asks for (`--print native-static-libs`) with `PublicSystemLibraries`, both read from `lib/<Platform>/link.txt`, which the CLI writes. `Rusteal.Build.cs` depends on it, and defines `RUSTEAL_STATIC=1`, for non-editor targets while `Generated/static_link.txt` holds `1` (an external dependency of the module, so UBT notices the switch); the CLI sets it for the packaging and back to `0` after. `Rusteal.AddRustLibrary`, which the plugin templates' `Build.cs` call, then stages no dynamic library.
- **Loading.** With `RUSTEAL_STATIC`, `LoadLibrary` finds the library's entry in `rusteal_static_modules` by name, checks its version and calls its `init`, exactly as for a loaded file; `UnloadLibrary` calls its `shutdown`; a reload is refused.
- **Registrations that must not get lost.** `inventory` registers types with static constructors in each object file. A linker keeps an object of an archive only when something references it; fat LTO and one codegen unit make the Rust code a single object, which the reference to `rusteal_static_modules` keeps. To check it rather than trust it, the CLI first builds and runs the workspace's `manifest` binary, which lists every registration (`kind:crate:name`, impl blocks and load hooks included) the way an executable sees them, and writes each library's list into `static_modules!`. When a module starts, it compares that list with what it actually found, and refuses to start, naming what is missing.
- **Memory.** Unless `rusteal.toml` says `[static] engine_allocator = false`, the archive sets `rusteal_runtime::EngineAllocator` as Rust's global allocator: every Rust allocation goes through `FMemory`, which the engine has ported to each platform, through four C functions the plugin defines only with `RUSTEAL_STATIC` (`RustealPlatformApiImpl.cpp`). The link resolves them, so they work before any library starts. The manifest binary keeps the system allocator.
- **Ownership.** `cargo metadata` gives the crates each library depends on through local paths (not the Rusteal crates nor bindings): those crates are the library's, so a helper crate of the game registers its types in the game's package. A crate reached by two libraries is refused, since its types would belong to both.

## Before changing the boundary

- **Order is the ABI.** `api_table.rs`, `RustealApiTable.h`, the positional `G*Api` initializers and `FillApiTable`, and `RustealRustCallbacks` with its C++ mirror and the callbacks `entry!()` generates, must all agree. Only layouts are tested.
- **Shared structs** are `#[repr(C)]`, named `Rusteal*`, `F*` or `U*`, with contract asserts on both sides. Handles stay 8 bytes; enums `repr(u32)`; `bool` is one byte on both sides.
- **Versions**: the runtime check reads the `.uplugin`'s integer `Version`; the CLI compares `VersionName`. The release writes both (see [Releases and versions](../contributing/releases.md)).
- **Generated files** (`rusteal-core/api_table.rs`, `Generated/`) are build outputs: change `rusteal-ffi/src/api_table.rs` and the code generator instead.
