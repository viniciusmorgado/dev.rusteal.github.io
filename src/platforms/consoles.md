# Consoles

Rusteal builds for Linux, Windows and macOS (Apple silicon); consoles are not supported yet, and everything that could be prepared without an SDK is done (see [Road ahead](#road-ahead)). This chapter is how Rusteal gets to PlayStation 5 and Xbox Series X|S (and to any platform whose Unreal support is private), what can be prepared now in the public repository, and what can only be done once a project has console access. It comes from two research reports made on 11 October 2026 from public sources and from Rusteal's own code, without access to any console SDK: what they could confirm is marked as such, and what depends on the SDK is listed as open.

The goal behind the plan is concrete: a game made with Rusteal ships on PC, does well, and its publisher gets it console access. From that moment the publisher wants a release date, so the work left after access has to be as small and as predictable as possible.

## Conclusions

What the research and the review of Rusteal's own code concluded, on 11 October 2026:

1. **No Rust support from the platform holders is needed.** Rust produces native code; what matters is whether that code links and runs in the platform's environment. Looking for an official "Rust SDK" for a console is the wrong question.
2. **Linking Rust into a console build needs no engine fork.** A static library consumed by an external module, built by Rusteal's CLI before UBT runs, uses the engine's documented mechanisms for every platform. Patching UBT is justified only by a limitation measured with the SDK in hand.
3. **The artifact is a `staticlib` per target.** Whether a console links it into the executable (route S) or turns it into a dynamic library (route D), the console build starts from it. Some consoles do load dynamic libraries, but that does not remove the need for a Rust target the platform's linker accepts.
4. **The biggest obstacles are in Rusteal, not in the consoles.** The runtime relies on each library being a separate dynamic library: fixed export names, one table per runtime instance, positional function ids, short-name type ids, link-time registration, identical bindings package names. An aggregated static build breaks all of them, and all of them can be fixed on desktop, in public, without an SDK.
5. **The largest console-specific unknown is the Rust target and its standard library**, above all on PS5. Making the runtime take its services from the engine (allocation, tasks, time), which is already ported to every console, shrinks that unknown to "the crates compile and link for the target".
6. **The repository stays public; console specifics do not.** Rusteal keeps a platform-agnostic structure and an extension point; each console's target, flags, build rules and CI live in a private platform pack.
7. **The aim is a short, predictable path after access.** Everything that does not need the SDK is done before it, so that once a publisher obtains access, the remaining work is the toolchain bring-up and validation on the devkit, not a redesign.

## What console access is

Three separate permissions, none of which replaces the others:

| From | What it gives | What it does not give |
|---|---|---|
| The platform holder (Sony, Microsoft) | Developer status for the organization, the SDK, devkits, the submission process | Epic's console code |
| Epic | Access to the engine's **Platform Extensions** for the approved platforms, through the Epic Developer Portal, after the platform holder's approval | The SDK, which Epic does not redistribute |
| The publisher | The relationship, the title, often the request for access | Permissions for each organization and person, which are granted by the parties above |

The console build of Unreal is the engine built from source with the platform extension added, against the SDK version that extension supports; Epic's AutoSDK layout and Turnkey help install and verify it. That is the normal way any studio builds for a console; it is not a fork of the engine.

What Rusteal adds on top must not depend on anything else from that list. In particular, **no Rust support from the platform holders is needed or expected**: Rust produces native code, and the question is only whether that code links and runs in the platform's environment.

## Public structure, private platform packs

Rusteal stays open source; console code cannot be. The line between them:

| Public, in this repository | Private, per platform, under the agreements |
|---|---|
| The static backend, the module contract, the services the runtime takes from the engine | The platform's Rust target definition (target JSON, sysroot), flags and `build-std` settings |
| The CLI's notion of a target platform and its extension point for platform definitions | The definition files the CLI loads for each console |
| The external module that consumes a static library | The `Build.cs` branches and staging rules for the console |
| The desktop fixture and its tests | Patches to the Rust standard library, if a console needs them |
| This chapter | CI with the SDK and the devkits, and the platform's own documentation |

A private **platform pack** per console holds the right-hand column, and is given only to developers who show they have signed Epic's console access agreement and the platform holder's. Its licensing is a separate decision.

## How Rust reaches a console build

The engine's build is not changed. The parts:

1. **Rust compiles to a static library.** For each target and configuration, Cargo builds a `staticlib`: an archive of native objects the platform's linker can consume like any C++ library.
2. **An external module consumes it.** A module of type `ModuleType.External` adds the archive with `PublicAdditionalLibraries`, and its include paths and system libraries; the module that needs the Rust code depends on it. This is the documented way to integrate a third-party library, and it works the same way for every platform.
3. **The CLI orchestrates.** `rusteal` already runs steps before and after the engine's build (see [The CLI](../architecture/cli.md)). For a console it runs Cargo for the target, checks the result against a manifest, and only then runs UBT and UAT. In CI the same sequence can be a BuildGraph graph (`Spawn` the Rusteal step, then `Compile` the game).
4. **UBT links, with the platform's linker.** The game's binary contains the engine, the plugins' C++ and the Rust archive; cooking, staging and packaging are the engine's and the platform extension's, unchanged.

**Why not patch UBT.** UBT's documented hooks (external modules, `PreBuildSteps`, BuildGraph) are enough to link a library built beforehand. A patch would only be justified by a need measured with the SDK in hand: Rust actions inside UBT's own graph and distributed executor, for instance. It would have to be kept up with every engine version, and it would not help if the platform rejected the Rust objects themselves, which is a toolchain question.

**Why a static library and not loose objects or bitcode.** A `staticlib` carries every object a crate graph needs, in the form the platform's linker expects; loose objects (`--emit=obj`) do not include the dependencies, and an `rlib` is Rust's internal format. Bitcode would tie the Rust compiler's LLVM version to the platform compiler's. Cross-language LTO is out of scope until a first build works.

## Two routes from the same archive

| Route | What ships | Rusteal changes | Depends on |
|---|---|---|---|
| **S, static** (the default) | One aggregated Rust archive linked into the game's executable | The module contract below | Only the platform's linker accepting the archive |
| **D, dynamic** | One dynamic library per Rusteal library, produced by the platform's linker from its archive | Little: the libraries stay isolated as today | The platform allowing game code in dynamic libraries, and the engine's `FPlatformProcess::GetDllHandle` working there |

Unreal builds consoles monolithically, and that is the path platform certification knows best, so **S** is the one to prepare; **D** stays a fallback where a platform supports it. Microsoft documents DLLs for Xbox titles; for PS5, nothing is publicly verifiable. Either way the console build starts from the same `staticlib` per library.

The editor keeps the dynamic libraries and hot reload: the static backend is for packaged games only, built from the same crates.

## What the static backend changed

Each library (the game's and each Rusteal plugin's) was a `cdylib` loaded on its own, and several parts of the runtime relied on that isolation. In one aggregated archive they would have collided; each was changed on desktop, in this repository (milestone 1, the module contract, and milestone 2, the static backend):

| Before | In one archive | Now |
|---|---|---|
| `entry!()` exported the same four symbols (`rusteal_init`...) from every library | Duplicate symbols | `entry!()` defines `__RUSTEAL_MODULE` and `__RUSTEAL_EXPORTS`; the dynamic exports exist only without `static-link`; the archive exports `rusteal_static_modules` alone |
| `rusteal-core/src/api.rs` kept one table pointer in a `OnceLock`, set once | The second library to initialize panicked | The services' table is shared; each library has a `Module` context, and its own function table lives in its bindings crate |
| The generated functions' ids were positions in one table | Index 7 of plugin A is not index 7 of plugin B | Each bindings crate reads its own `__RUSTEAL_FUNCS`, bound and checked at start |
| A type's id hashed its short name | `Inventory::Item` and `Quest::Item` collided | Ids hash the module path and name; parents and components give theirs through `UeClass::RUST_TYPE_ID` |
| Registration relied on `inventory`'s link-time collection | Nothing guaranteed the linker kept every registration | One object (fat LTO, one codegen unit), registrations tagged with their crate, and a manifest each library checks at start |
| Every generated bindings crate was the package `bindings 0.1.0` | Cargo refused the crate graph | The package is `<crate>-bindings`, imported as `bindings` |
| The handshake compared the Rusteal version | Two generations of bindings were not told apart | The function table's count and fingerprint are checked when a library starts |
| `on_shutdown` ran twice per unload | No library handle to rely on | One start and stop state machine per module, safe twice and after a partial start |

The C++ side kept its per-library tables and package paths (`/Script/<Plugin>`): cooked Blueprints refer to Rust classes by them. A library is found by name in the archive instead of by file. How it all works is in [Static linking](../architecture/boundary.md#static-linking).

## Depend on the engine, not on the operating system

The Rust standard library talks to the operating system: threads, time, locks, thread-local storage, the allocator, unwinding. On a console those parts of `std` are the hardest to port. Unreal, on the other hand, is already ported to every console it supports. The runtime can take its services from the engine through the API table instead of from `std`:

| Service | Before | Now (milestone 4) |
|---|---|---|
| Allocation | The system allocator | In a static build, `rusteal_runtime::EngineAllocator`, a `GlobalAlloc` over `FMemory::Malloc`, `Realloc` and `Free` through C symbols the plugin defines, available before the first Rust allocation; on by default, `[static] engine_allocator = false` turns it off (for a game with its own global allocator). The editor's dynamic libraries keep the system allocator. |
| Background work | `task::spawn` on `std::thread` | The engine's task system (`UE::Tasks::Launch`, background priority), through the API table's `platform` sub-table |
| Time | `std::time::Instant` | `FPlatformTime::Seconds`; `rusteal_runtime::runtime::time::Instant` gives it to game code |
| Game thread | A recorded `ThreadId` | `IsInGameThread` |
| Logging | The `log` crate and `ulog!` already went to the engine | (unchanged) |

`rusteal-core` keeps `std` versions of these only in `std_fallback.rs`, for its tests without the engine; a test fails if any other source file reaches `std::thread` or the `std` clock. Locks (`std::sync`) and thread-locals are still `std`'s: they are the next candidates, if a console's port of `std` lacks them.

With those, the runtime needs little more than `core`, `alloc` and the synchronization primitives on a console, and a port of `std` becomes a smaller and optional piece of the platform pack. Game code can keep using `std` on PC; on a console the rule is that it uses Rusteal's services rather than the operating system directly. `[consoles] check = true` in `rusteal.toml` makes every build list the game's uses of `std::thread` and the `std` clock, and the dependencies known to start threads or use the OS's I/O (`rayon`, `tokio`...), as warnings: nothing is refused, since a dependency may work once a platform's `std` is ported.

**Panics.** Consoles' C++ is usually built without exceptions, and unwinding may not be available to Rust there. If a console needs `panic = "abort"`, `ffi_boundary`'s `catch_unwind` no longer turns a panic into a logged error: the process ends, through the platform's crash reporting. Expected failures are already `RustealResult`s; the change is a documented policy per platform, not a different API.

## The Rust toolchain per console

The open part of the plan, and the one each platform pack carries:

- **A target.** No public Rust target names a console, and none is assumed. A console's target starts from the nearest public one (`x86_64-pc-windows-msvc` for Xbox, `x86_64-unknown-freebsd` for PS5) as a custom target, then matched to the platform's ABI: calling convention, relocation model, TLS model, object format, unwinding.
- **The standard library.** Building `core` and `alloc` for a custom target is routine with `build-std`; `std` needs an implementation of its system layer for the platform, which is why the runtime leans on the engine for services. On Xbox, the Game OS exposes a partition of the Windows API: imports outside it show up at link time, and the C runtime must match the engine's.
- **Proof before scale.** The first console experiment links one function with no heap, threads or I/O into a minimal Unreal project that already runs in C++ alone, then adds allocation, a callback from the engine, registration, tasks.

## Road ahead

The port has two phases. The first is done now, on desktop and in public, before any console access: it is where most of the work is, and doing it first removes it from the critical path of a console release. The second starts the day a publisher obtains access, and is kept as short as possible by the first.

### Before access: on desktop, in public

All four were done in October 2026 (Rusteal 0.9) and validated on Linux; Windows and macOS are untested.

| Milestone | Work | Done when |
|---|---|---|
| 1. Module contract | Contexts per library, qualified type ids, explicit registration, unique bindings package names, bindings fingerprint, one shutdown state machine; the dynamic backend keeps working | A fixture with the game and two Rusteal plugins passes: calls from A to B and back reach the right library, two classes with the same short name coexist, a library's tasks and delegates keep their owner |
| 2. Static backend on desktop | `rusteal package --static`: one aggregated `staticlib`, an external module consuming it, a manifest of the artifact; the editor stays dynamic | The fixture, packaged in Development and Shipping on Linux and Windows, starts from a clean machine with no Rust dynamic library, cooks and loads a Blueprint derived from a Rust class |
| 3. Target platforms in the CLI | Host and target separated; unknown targets refused instead of falling back to `.so`; platform definitions loaded from outside the repository | A platform definition file for desktop drives the same build as the built-in one |
| 4. Engine services | The allocator, tasks and time through the API table | The runtime builds without the `std` features it no longer needs, and the desktop fixture behaves the same |

How they were validated: the fixture is a project with the game and two runtime plugins whose Rust code runs a scenario by itself and writes `PASS`/`FAIL` lines: two classes named `Item` in two libraries, a call from the game into a plugin's function and back, each library's tick hooks counted over thirty frames (once per frame each), a plugin's background task, a closure of the game on a plugin's actor's delegate and the other way round, a Blueprint child of a Rust class placed in the level, the game thread and the engine's clock. It passed in the editor's game mode (dynamic libraries), packaged with `--static` in Development and in Shipping (no Rust dynamic library in the package), and packaged from a definition file `rusteal platform show Linux` printed. The projects of the interface and error tests and every template were built and played again on top of it.

### After the four: closing what needs no SDK

Four more steps, done in October 2026 on Linux, take the desktop work as close to a console build as a machine without an SDK can:

| Step | What it showed |
|---|---|
| Linux ARM64, cross-compiled | A static package for `LinuxArm64` from an x64 machine, with the engine's own ARM64 toolchain: the executable is `aarch64` with the Rust code in it. It also showed that a library crate with a `cdylib` crate type made Cargo link a dynamic library for the target as well, with the host's linker: library crates are now plain `rlib`s, and `rusteal build` makes the editor's dynamic library with `cargo rustc --crate-type cdylib` (decision E13). |
| A simulated platform pack | A definition file outside the repository, as a console's will be: a target JSON (a copy of the Linux target with `panic-strategy: abort`), a dated nightly toolchain, `build_std = ["std", "panic_abort"]`, `-Zjson-target-spec` (which recent nightlies require for a JSON target) and `-Cpanic=abort`. The fixture packaged from it passes every check. |
| Panics that abort | With `panic = "abort"` a panic ends the process; the hook reports it, with its location, to the log first (decision B10). |
| Locks, thread-locals and `rayon` | The runtime's `std` locks and thread-locals are in one module (B11); `rayon` runs on the engine's threads, one pool per library, safe across a hot reload (B12). |

Windows remains to be validated (`rusteal package --static` on Win64, the closest desktop to the Xbox toolchain), on a Windows machine.

### After access: with the SDK and the devkits

In the private platform pack, on top of the four milestones above:

| Step | Done when |
|---|---|
| 5. C++ baseline | A minimal Unreal project builds and runs on the devkit with the platform extension, with no Rust |
| 6. Smallest Rust slice | One function from Rust links and runs in that project |
| 7. The fixture | The desktop fixture runs on the devkit: allocation, callbacks, registration, cooked content, tasks, shutdown |
| 8. The game | The title's own build, its certification requirements, its crash and debugging tools |

Without the first phase, steps 5 to 8 would also carry the module contract, the static backend and the runtime services, under the publisher's date; with it, the work left is the toolchain bring-up and validation. A firm estimate needs the SDK, but the schedule risk is concentrated in one item: each console's Rust target.

A failure at each step has a known owner: a context, registration or cook problem on PC is Rusteal's; a symbol or service missing at the console's link is the platform pack's; a cold start failing after a good build points to registration order or runtime services; a project that works for its author but not from a clean checkout is a reproducibility problem, to fix before calling anything shippable.

## Open until there is access

- Whether each platform accepts objects from the Rust compiler as they are, and which target definition matches its ABI.
- Which parts of `std` and which C runtime the Rust objects can use on each console.
- Whether PS5 allows game code in dynamic libraries (route D).
- Debugging and profiling Rust code with the consoles' tools: symbols, mixed C++ and Rust stacks, crash dumps.
- The exact build, packaging and submission steps of each platform extension, which are not public.

## Sources

The two research reports behind this chapter (11 October 2026), and the public documentation they rely on, among it:

- Epic: [Getting started with console development](https://dev.epicgames.com/community/learning/knowledge-base/B9xk/getting-started-with-console-development-for-unreal-engine-and-epic-online-services), [Integrating third-party libraries](https://dev.epicgames.com/documentation/unreal-engine/integrating-third-party-libraries-into-unreal-engine), [Module properties](https://dev.epicgames.com/documentation/en-us/unreal-engine/module-properties-in-unreal-engine), [Build tool target reference](https://dev.epicgames.com/documentation/unreal-engine/unreal-engine-build-tool-target-reference?lang=en-US), [BuildGraph](https://dev.epicgames.com/documentation/en-us/unreal-engine/buildgraph-for-unreal-engine), [AutoSDK](https://dev.epicgames.com/documentation/en-us/unreal-engine/using-the-autosdk-system-in-unreal-engine).
- Microsoft: [Developing a new title with the GDK](https://learn.microsoft.com/en-us/gaming/gdk/docs/gdk-dev/get-started/overviews/developing-new-titles-on-gamecore?view=gdk-2604), [vcpkg on Xbox](https://learn.microsoft.com/en-us/vcpkg/users/platforms/xbox).
- Rust: [Linkage](https://doc.rust-lang.org/reference/linkage.html), [Custom targets](https://doc.rust-lang.org/rustc/targets/custom.html), [build-std](https://doc.rust-lang.org/cargo/reference/unstable.html#build-std), [FFI in the Rustonomicon](https://doc.rust-lang.org/nomicon/ffi.html).
