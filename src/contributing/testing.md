# Testing a change

A change is done when the workspace checks pass, a project made from the checkout builds and plays with it, and the templates it can affect still do.

The workspace builds, tests and lints without the engine:

```bash
cargo build --workspace
cargo test --workspace
cargo clippy --workspace --all-targets --no-deps -- -D warnings
```

That is what CI runs on every pull request, on Linux only: the Windows and macOS code paths (`#[cfg(target_os)]`) are only built on those machines, so run it there too when you change them. Run it before pushing. Where the engine-free tests are, and what each protects, is in [The CLI](../architecture/cli.md#tests), [Code generation](../architecture/codegen.md#tests) and [The runtime](../architecture/runtime.md#testing-the-runtime).

Anything that touches the engine is tested in a UE project of your choice,
driven by this checkout's CLI (`cargo run -p rusteal --`, from the repository
root). A new project that depends on this checkout:

```bash
cargo run -p rusteal -- new Probe --dir <dir> --runtime-path .
```

Put it in a directory of your own, such as the one that holds your Unreal projects, not in `/tmp`: on Linux, UBT fails at step 1 for a project under `/tmp`, building the exporter's path without its leading `/` ("is not an absolute path").

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

## Before a pull request

- A game-facing API needs no `unsafe` and no raw handle from its caller, takes and returns typed references and Rust types, and its error is a `RustealResult`; the templates' sources stay free of `unsafe` (`game_developers_never_write_unsafe_code`).

- The workspace checks above pass.
- The test project builds with `rusteal build --all` and plays with no `LogRusteal` error or warning.
- A change to the runtime, the macros, codegen or the plugin: the templates it reaches still build and play (`rusteal new` of each, with `--runtime-path`). Codegen and plugin changes reach all of them.
- A change the game developer sees: the game developer's handbook says so. A decision: [Decisions](../decisions/index.md).
