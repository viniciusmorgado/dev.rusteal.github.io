# Setting up a checkout

This chapter is for changing Rusteal itself: its crates, the `rusteal` CLI and the UE plugins in `ue_plugin/`. To make a game, none of it is needed; see the [game developer's handbook](https://viniciusmorgado.github.io/game.rusteal.github.io/).

## Prerequisites

Everything in the game developer's [Prerequisites](https://viniciusmorgado.github.io/game.rusteal.github.io/getting_started/prerequisites.html), plus:

| Dependency | Where from | What for |
|---|---|---|
| Git | the distribution's package (`git`); on Windows, [Git for Windows](https://git-scm.com/download/win); on macOS, Xcode's | cloning the repository |
| [clangd](https://clangd.llvm.org) (optional) | the distribution's package or LLVM (on macOS, Homebrew's `llvm`); Zed downloads it on its own | C++ support in the editor, for `ue_plugin/` |
| clang-format | the distribution's package or LLVM (on macOS, Homebrew's `llvm` or `clang-format`: Xcode has none) | formatting `ue_plugin/` with `.clang-format` |
| [.NET 10 SDK](https://dotnet.microsoft.com/download) (optional), on the `PATH` | Microsoft (the macOS Arm64 installer on a Mac), or the distribution's package (`dotnet-sdk-10.0`) | C# support in the editor, for the exporter in `ue_plugin/RustealGenerator/`; `dev-setup` restores the solution with it |

The checkout builds with the MSRV, Rust 1.88.0, whatever your default
toolchain is: `rust-toolchain.toml` pins it, and rustup installs it on the
first `cargo` command. The file names no host, so rustup completes it with its
default host; on Windows that must be `x86_64-pc-windows-msvc` (see the
Windows prerequisites above), on macOS `aarch64-apple-darwin`: an x86_64
rustup under Rosetta builds libraries the Apple silicon editor cannot load.

## The development environment

The settings that depend on the machine live in `.env` at the repository root,
which is not versioned; `.env.example` lists them:

| Variable | What it is |
|---|---|
| `RUSTEAL_DEV_ENGINE_ROOT` | The Unreal Engine root, the directory holding `Engine/` |

Linux and macOS:

```bash
git clone https://github.com/viniciusmorgado/rusteal.git
cd rusteal
cp .env.example .env    # then fill it in
cargo xtask dev-setup
```

Windows (PowerShell):

```powershell
git clone https://github.com/viniciusmorgado/rusteal.git
cd rusteal
Copy-Item .env.example .env    # then fill it in
cargo xtask dev-setup
```

Rusteal is developed against a UE project: the plugins in `ue_plugin/` are built
inside one, and the C++ editor support reads what that build generates (UHT's
`*.generated.h` headers and UBT's compile commands). `dev-setup` asks which one:

1. **An existing project.** It must have this checkout's plugins installed and
   built (`cargo run -p rusteal -- setup <project>`, then `build`); `dev-setup`
   only reads it, never changes it, and stops with those commands if they are
   missing. Plugin headers that differ from the checkout's are reported.
2. **A new blank project**, created at the path you give with
   `rusteal new --runtime-path` on this checkout, and built. The last component
   of the path is the project name.

To skip the question, pass the answer: `cargo xtask dev-setup --project <path>`
or `cargo xtask dev-setup --new <path>`.

Then it writes:

- **`compile_commands.json`** at the repository root, for clangd: the compile
  commands UBT gives for that project, pointed at `ue_plugin/`;
- **the exporter's `.csproj.props`**, so C# language servers resolve the
  engine's `EpicGames.*` assemblies, and a `dotnet restore` of `rusteal.sln`
  with the `dotnet` on the `PATH` (a .NET SDK 10 or newer; without one,
  `dev-setup` warns and goes on). The solution has two projects, both
  for IDEs only: the exporter (`ue_plugin/RustealGenerator/Source/RustealExporter/`),
  and `ide/RustealRules/`, which gives the plugins' `*.Build.cs` files the context
  UBT compiles them with. Without that project a language server analyzes those
  files with no references at all: nothing resolves, and every `using` is
  reported as unnecessary. UBT compiles the `Build.cs` files without implicit
  usings, so a `using System.IO;` there is required, and the project makes the
  editor agree.

Run it again after adding a source file to the plugin, or after changing a
header that declares a `UCLASS` or `USTRUCT` and rebuilding it in the project:
a generated header only matches the header it was made from.

`.env` is for working on Rusteal only:

| | Working on Rusteal | Making a game |
|---|---|---|
| Engine path from | `.env` | the `rusteal` CLI, which asks once and keeps it in `~/.config/rusteal/config.toml` (`~/Library/Application Support/rusteal/config.toml` on macOS, `%APPDATA%\rusteal\config.toml` on Windows) |
| Read by | `cargo xtask` only | the `rusteal` CLI |
| Writes the exporter's `.csproj.props` | `cargo xtask dev-setup`, in this checkout | `rusteal setup`, in the game project |

The `rusteal` CLI never reads `.env`, so it does not change which engine a game
builds with.
