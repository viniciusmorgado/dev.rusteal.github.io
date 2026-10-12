# Known issues and sharp edges

Things that work as written but surprise. Each is a candidate for a fix; until then, know them before changing the code around them.

## The boundary

- **Function pointer order is not tested.** The contract tests check struct sizes and offsets, not the order or signatures of the table's function pointers. A field added on one side only, or in a different position, compiles and calls the wrong function. A field appended to a C++ sub-table but missing from its positional initializer is a null pointer.
- **The API table has no fingerprint of its own.** A library checks the version and its bindings' fingerprint; changing the API table without changing the version (any work between releases) is not detected: rebuild the plugin and the library together.
- **Plugins made before 0.9 register without a fingerprint** (`RustealRegisterPluginLibrary`): their library checks the function count only. Switching the module to `RustealRegisterPluginLibraryChecked`, as the templates do, turns the check on.
- **The game thread is a convention.** No API entry point asserts it; `GCurrentLibrary` is not thread-local. Engine calls from a background thread are undefined; use `task::spawn` and `run_on_game_thread`.
- **Validity checks differ by entry point.** Property and container functions only check for null; generated wrappers check `IsValid` only in Debug and Development. Rust callers must `checked()` first.
- **No `FWeakObjectPtr` size assert** on the C++ side, though weak pointers are copied as 8 bytes.

## Macros

- **Unknown `#[uclass]` and `#[component]` arguments are ignored**, and attributes on a `#[uclass]` struct are dropped, where `#[uproperty]` and `#[ufunction]` reject unknown arguments.
- **Private fields are invisible to the garbage collector.** A `UObjectRef` kept in a private field does not keep its object alive.
- **`Pinned` is not reference counted.** Two `Pinned` of the same object share one root; dropping either unroots it.

## Idiomatic gaps (decision I7)

No game code needs `unsafe`. What is still less idiomatic than it could be:

- **Input actions and delegates are bound to a function by its name** (`bind_action(&me, action, ETriggerEvent::Started, "OnInputStarted")`, `_add_ufunction(target, "Function")`): a typo is found at run time, as in C++'s `BindAction`. Binding a method or a closure directly would catch it at compile time.
- **`RustealError::log` takes a level constant** (`LOG_WARNING`) instead of a `log::Level`.
- **`get_class()` and `get_outer()` still return raw handles**, kept for existing code; `class()` and `outer::<U>()` are the typed ones.
- **A `#[uclass]` struct's generated fields** (`__obj`, `__rust_data`) are public, for the macros' own code; their names keep them out of the way, but they are not hidden from the type.

## Static linking

`rusteal package --static` links every library into the game (see [Static linking](../architecture/boundary.md#static-linking)):

- **Validated on Linux only.** The Windows (`.lib`, MSVC's runtime) and macOS paths are written but have not been packaged.
- **No hot reload** of a static library, by design; `Rusteal.Reload` says so.
- **One `log` logger and one panic hook** for every library: the maximum log level is shared.
- **Threads Rusteal did not start** run with no module current, and are attributed to the game's library: a closure a `rayon` worker registers belongs to the game.
- **A crate used by two libraries is refused**, since its types would be registered by both; move shared code without Rust types into its own crate, used by both, or the types into one library.
- **Editor plugins are left out** of the archive (they are not part of a packaged game).
- **Plugins made before 0.9** stage their dynamic library anyway: their `Build.cs` adds it directly instead of through `Rusteal.AddRustLibrary`.
- **The manifest binary is built for the host.** A registration behind a `cfg` of the target platform would differ between the host's list and the target's archive.
- **The engine allocator is global.** A game that sets its own `#[global_allocator]` must turn it off (`[static] engine_allocator = false`).

## Hot reload

- **Structs and enums** keep their old layout until an editor restart.
- **Rust-private fields** start over from `Default` on every reload.
- **macOS keeps every reloaded library**: `dlclose` never unloads a Rust library there, so each reload leaves the previous one mapped until the editor exits; the load copy is a full copy, not a link (see [Hot reload](../architecture/hot_reload.md)).
- **Not tested**: hot reload on Windows, a changed parent class, components added or removed (see `docs/hot-reload.md`).

## Tooling

- **`SIGTERM` can crash the editor on macOS** while the engine builds distance fields (a first run, `-ddc=Cold`): an engine issue, with or without Rusteal. Closing the editor or the game normally does not send it; scripts should stop a headless editor with `-ExecCmds=quit` or `kill -9`.
- **Strings the engine returns are cut at 512 bytes** in the generated bindings (`StringUtf8` buffers); the C++ wrapper respects the buffer's length, so a longer string is truncated, never overrun.
- **`rusteal-cli/ue_plugin_embed/`** is a local, possibly stale snapshot; builds in the workspace ignore it (they read `ue_plugin/`).
- **`setup.rs`, `global_config.rs` and `sync_plugin.rs`** have no unit tests; they are covered by making a project.
- **Packaging on macOS needs a workaround**: with Xcode 27, UE 5.8 starts the `xcodebuild` that finalizes the `.app` without a stdin and the build fails (`Failed to finalize the .app with Xcode`), for any C++ project. `rusteal package` works around it ([Platform optimizations](../internals/platform_optimizations.md#macos-xcodebuild-without-a-stdin)); running UAT by hand still fails. Reported to Epic: [the forum post](https://forums.unrealengine.com/t/5-8-mac-failed-to-finalize-the-app-with-xcode-on-xcode-27-xcodebuild-started-without-stdin-bad-file-descriptor-in-touch-ubt-generated-tiles/2839864).
- **CI does not run on Windows**: the Windows code paths are built and tested on a Windows machine by hand; Linux and macOS have CI jobs.
- **macOS on Intel processors** has not been tested; macOS support is Apple silicon only.
