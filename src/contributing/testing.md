# Testing a change

A change is done when the workspace checks pass, a project made from the checkout builds and plays with it, and the templates it can affect still do.

The workspace builds, tests and lints without the engine:

```bash
cargo build --workspace
cargo test --workspace
cargo clippy --workspace --all-targets --no-deps -- -D warnings
```

That is what CI runs on every pull request, on Linux and on macOS (Apple silicon): the Windows code paths (`#[cfg(target_os)]`) are only built on Windows, so run it there too when you change them. Run it before pushing. Where the engine-free tests are, and what each protects, is in [The CLI](../architecture/cli.md#tests), [Code generation](../architecture/codegen.md#tests) and [The runtime](../architecture/runtime.md#testing-the-runtime).

Anything that touches the engine is tested in a UE project of your choice,
driven by this checkout's CLI (`cargo run -p rusteal --`, from the repository
root). A new project that depends on this checkout:

```bash
cargo run -p rusteal -- new Probe --dir <dir> --runtime-path .
```

Any directory works, on the engine's volume or another one: the exporter builds into the project (decision D6).

After changing the runtime, the macros or a game crate:

```bash
cargo run -p rusteal -- build <dir>/Probe
```

After changing the UE plugins in `ue_plugin/`, reinstall them into the project
and build it:

```bash
cargo run -p rusteal -- setup <dir>/Probe
cargo run -p rusteal -- build <dir>/Probe --all
```

In the editor, each `build` reloads the library by itself, changed classes
included (see [Hot reload](https://viniciusmorgado.github.io/game.rusteal.github.io/iteration/hot_reload.html)); a change to the UE plugins needs an
editor restart.

A project made with `--runtime-path` follows that checkout: the CLI acts on it
only when built from the same checkout, so drive it with `cargo run -p rusteal --`
from there. After pulling, `setup` reinstalls the plugins at the checkout's
version.

## `--runtime-path`

It decides where the generated project takes Rusteal from:

| | without it | `--runtime-path <checkout>` |
|---|---|---|
| `Rust/Cargo.toml` says | `rusteal-runtime = "=x.y.z"` | `{ path = "<checkout>/rusteal-runtime" }` |
| Crates come from | crates.io | the local checkout |
| A change in the runtime reaches the game | on the next published version | on the next build |
| The project builds on another machine | yes | no (the path is this machine's) |

So: without it for a real game, with it while working on Rusteal — a change in
`rusteal-core` shows up in the project you test with on the next `build`,
with nothing to publish in between.

## The engine harness

What crosses the boundary (properties, functions called by reflection, engine functions, strings, interfaces, soft and weak references, delegates, tasks) is checked in a running game by one command:

```bash
cargo xtask engine-test
```

It builds this checkout's CLI, creates `target/engine-test/RustealTests` with `--runtime-path` (or updates it with `setup`), copies the checks from `tests/engine/src` into its game crate, regenerates the bindings and builds the project, then runs it as a game with no window (`-game -nullrhi`; `ProcessEvent` does not run functions in the editor's world). The console command `rusteal_tests.run` runs every check of `tests/engine/src/checks.rs` and writes a `PASS` or `FAIL` line for each to the results file, and the game quits; a run that takes more than ten minutes is killed. It prints the results and fails when one check does. The logs are in `target/engine-test/logs/`.

The project also has a Rust plugin, `RustealProbe`, made by `rusteal plugin new` and filled from `tests/engine/plugin/src`, for the checks of one library using another's classes. After the game, two more runs join the results:

- a Rust commandlet, `-run=VirtualsCommandlet`, whose exit code is checked;
- the plugin's editor library, `RustealProbeEditor` (from `tests/engine/plugin/editor/src`), whose `rusteal_editor_tests.run` and the Python scripts next to it (`py <file>`) check editor subsystems, utility objects, factories, Slate, the details panel and what editor Python reaches, in the same editor run as the reload;
- an editor with no window that hot reloads the plugin: `rusteal_tests.reload_prepare` creates a game class whose parent is the plugin's and replaces the plugin's library with a build of `tests/engine/plugin/reload/` (a changed class), `Rusteal.Reload RustealProbe` reloads it, and `rusteal_tests.reload_verify` checks the game's class was rebuilt on the new parent. Its objects are `UObject`s, since the editor's world absorbs actors' `ProcessEvent`.

```bash
cargo xtask static-test
```

It packages the project with `rusteal package --static` into `target/engine-test/static-package` and runs the packaged game with the checks that need a game and a plugin linked together (`RUSTEAL_TEST_FILTER`). It takes as long as a package does, cooking included: run it after a change to how libraries are linked.

A check is a function from a `Context` (the game's world) to an `Outcome`, listed in `ALL`; one that waits for the engine returns `Outcome::Pending` and is polled every tick. Reproduce a boundary bug here first, and keep the check once it passes.

## Checking the templates

A change to the code generation or to the runtime's API can break the templates' code. One command compiles all of it:

```bash
cargo xtask templates-check
```

It creates a project from every template and variant in `target/engine-test/templates`, and a plugin from every plugin template in one more project, generates their bindings from a reflection dump made with every feature, and runs `cargo check` on their Rust code. The dump is exported once by UBT, from a project with every feature enabled, into `target/engine-test/full-dump`; delete it after changing the engine or the exporter. Name templates to check only those: `cargo xtask templates-check third-person:combat plugin:tour`. Nothing runs in the engine: what the templates do is still checked by playing them.

## Testing in the editor, headless

What happens in the editor is tested with editor Python, in an editor with no window:

```bash
"<engine>/Engine/Binaries/Linux/UnrealEditor" "<dir>/Probe/Probe.uproject" \
  -nullrhi -unattended -nosplash -nosound -NoLiveCoding \
  -ExecCmds="py <dir>/test.py" -abslog="<dir>/test.log"
```

(`UnrealEditor-Cmd.exe` under `Engine/Binaries/Win64/` on Windows, `Engine/Binaries/Mac/UnrealEditor.app/Contents/MacOS/UnrealEditor` on macOS.) A test script:

- loads a map, places what it needs in the editor world (Python cannot spawn into the Play In Editor world), and saves;
- starts Play In Editor with `LevelEditorSubsystem.editor_request_begin_play()` and drives it from a Slate tick callback (`unreal.register_slate_post_tick_callback`), a generator that yields the number of ticks to wait between steps;
- calls the Rust classes' functions and reads their properties by their Unreal names (`actor.call_method("SnapTurn", (True,))`, `get_editor_property("MoveSpeed")`), since editor Python wraps a Rust class as its nearest engine class;
- writes `PASS`/`FAIL` lines to a results file, ends Play, and quits with `unreal.SystemLibrary.quit_editor()`.

A C++ helper library in the test project's own module, never in Rusteal, reaches what Python cannot: the Message Log's entries and tokens, Blueprint graph nodes, function calls on a token's link. The ports in [Porting an engine template](../templates/porting.md#validating) are validated this way.

## Memory errors on macOS

A crash in `malloc` or a jump to a nonsense address is often memory corruption from earlier. On macOS, Guard Malloc puts every allocation of the system allocator (the one Rust uses) against a protected page, so the overrun faults at the instruction that does it; the engine's own allocations are not affected, so the editor stays usable:

```bash
DYLD_INSERT_LIBRARIES=/usr/lib/libgmalloc.dylib \
  "<engine>/Engine/Binaries/Mac/UnrealEditor.app/Contents/MacOS/UnrealEditor" "<dir>/Probe/Probe.uproject" \
  -game -nullrhi -unattended -nosplash -nosound -stdout
```

The engine's crash handler prints only the top of the stack; while it hangs on the crash, `lldb --batch -p <pid> -o "thread backtrace all"` shows the whole stack of the faulting thread, through the signal frame. Stop a headless editor with `-ExecCmds=quit` or `kill -9`, not `SIGTERM`: on macOS a `SIGTERM` that arrives while the engine is still building distance fields (a first run, or `-ddc=Cold`) crashes the engine, with or without Rusteal.

## Testing a static build

A change to the runtime, the macros, the plugin's loading or the static backend is also tested packaged with `--static`, which runs the Rust code linked into the game, with every library sharing one runtime:

```bash
cargo run -p rusteal -- package <dir>/Probe --static
cargo run -p rusteal -- package <dir>/Probe --static --shipping --output <dir>/Probe/Packaged/Shipping
```

Run the packaged game (`Packaged/Linux/Probe.sh`, `-nullrhi` for no window). A Shipping build has no log, so a test game reports through its own Rust code, writing `PASS`/`FAIL` lines to a file and quitting when it is done. What is worth covering is what differs from the editor: a game with two or more Rust plugins, classes with the same name in two libraries, calls and delegates between libraries, each library's tick hooks and tasks, a Blueprint child of a Rust class placed in a level (cooked), and the absence of Rust dynamic libraries in the package (see [Consoles](../platforms/consoles.md#road-ahead) for the fixture used for the static backend).

## Before a pull request

- A game-facing API needs no `unsafe` and no raw handle from its caller, takes and returns typed references and Rust types, and its error is a `RustealResult`; the templates' sources stay free of `unsafe` (`game_developers_never_write_unsafe_code`).

- The workspace checks above pass.
- The test project builds with `rusteal build --all` and plays with no `LogRusteal` error or warning.
- A change to the runtime, the macros, codegen or the plugin: the templates it reaches still build and play (`rusteal new` of each, with `--runtime-path`). Codegen and plugin changes reach all of them.
- A change the game developer sees: the game developer's handbook says so. A decision: [Decisions](../decisions/index.md).
