# Known issues and sharp edges

Things that work as written but surprise. Each is a candidate for a fix; until then, know them before changing the code around them.

## The boundary

- **Function pointer order is not tested.** The contract tests check struct sizes and offsets, not the order or signatures of the table's function pointers. A field added on one side only, or in a different position, compiles and calls the wrong function. A field appended to a C++ sub-table but missing from its positional initializer is a null pointer.
- **Func ids are not bounds-checked.** The Rust side never reads `func_count`; the bindings and the C++ `Generated/` must come from the same codegen run.
- **The version is the only compatibility check.** Changing the table without changing the version (any work between releases) is not detected: rebuild the plugin and the library together.
- **`on_shutdown` runs twice per unload**: once as a callback, once from `rusteal_shutdown`. It must stay idempotent.
- **The game thread is a convention.** No API entry point asserts it; `GCurrentLibrary` is not thread-local. Engine calls from a background thread are undefined; use `task::spawn` and `run_on_game_thread`.
- **Validity checks differ by entry point.** Property and container functions only check for null; generated wrappers check `IsValid` only in Debug and Development. Rust callers must `checked()` first.
- **No `FWeakObjectPtr` size assert** on the C++ side, though weak pointers are copied as 8 bytes.

## Macros

- **Unknown `#[uclass]` and `#[component]` arguments are ignored**, and attributes on a `#[uclass]` struct are dropped, where `#[uproperty]` and `#[ufunction]` reject unknown arguments.
- **Type ids hash the type's name only.** Two Rust classes with the same name in one library collide.
- **Private fields are invisible to the garbage collector.** A `UObjectRef` kept in a private field does not keep its object alive.
- **`Pinned` is not reference counted.** Two `Pinned` of the same object share one root; dropping either unroots it.

## Idiomatic gaps (decision I7)

No game code needs `unsafe`. What is still less idiomatic than it could be:

- **Input actions and delegates are bound to a function by its name** (`bind_action(&me, action, ETriggerEvent::Started, "OnInputStarted")`, `_add_ufunction(target, "Function")`): a typo is found at run time, as in C++'s `BindAction`. Binding a method or a closure directly would catch it at compile time.
- **`RustealError::log` takes a level constant** (`LOG_WARNING`) instead of a `log::Level`.
- **`get_class()` and `get_outer()` still return raw handles**, kept for existing code; `class()` and `outer::<U>()` are the typed ones.
- **A `#[uclass]` struct's generated fields** (`__obj`, `__rust_data`) are public, for the macros' own code; their names keep them out of the way, but they are not hidden from the type.

## Static linking (consoles)

Each Rust library is a separate dynamic library today, and parts of the runtime depend on it; one aggregated static library, the route to consoles, breaks them (see [Consoles](../platforms/consoles.md#what-the-static-backend-has-to-change)):

- `entry!()` exports the same four symbols from every library.
- One API table per runtime instance (`rusteal-core/src/api.rs`'s `OnceLock`).
- Generated function ids are positions in each library's own table.
- Type ids hash the short name only (above).
- Registration relies on `inventory`'s link-time collection, which an archive does not guarantee to keep.
- Every generated bindings crate is the package `bindings 0.1.0`, so Cargo refuses a crate graph holding two of them.
- The handshake compares the Rusteal version, not the bindings that were generated.

## Hot reload

- **Structs and enums** keep their old layout until an editor restart.
- **Rust-private fields** start over from `Default` on every reload.
- **macOS keeps every reloaded library**: `dlclose` never unloads a Rust library there, so each reload leaves the previous one mapped until the editor exits; the load copy is a full copy, not a link (see [Hot reload](../architecture/hot_reload.md)).
- **Not tested**: hot reload on Windows, a changed parent class, components added or removed (see `docs/hot-reload.md`).

## Tooling

- **`rusteal-cli/ue_plugin_embed/`** is a local, possibly stale snapshot; builds in the workspace ignore it (they read `ue_plugin/`).
- **`setup.rs`, `global_config.rs` and `sync_plugin.rs`** have no unit tests; they are covered by making a project.
- **Packaging on macOS needs a workaround**: with Xcode 27, UE 5.8 starts the `xcodebuild` that finalizes the `.app` without a stdin and the build fails (`Failed to finalize the .app with Xcode`), for any C++ project. `rusteal package` works around it ([Platform optimizations](../internals/platform_optimizations.md#macos-xcodebuild-without-a-stdin)); running UAT by hand still fails. Reported to Epic: [the forum post](https://forums.unrealengine.com/t/5-8-mac-failed-to-finalize-the-app-with-xcode-on-xcode-27-xcodebuild-started-without-stdin-bad-file-descriptor-in-touch-ubt-generated-tiles/2839864).
- **CI runs on Linux only**: the Windows and macOS code paths are built and tested on those machines by hand.
- **macOS on Intel processors** has not been tested; macOS support is Apple silicon only.
