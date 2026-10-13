# Platform optimizations

Some engine settings belong to one platform: a workaround for a driver, a default that suits one RHI. Rusteal declares them per platform in the project's `rusteal.toml` and the CLI writes the ones of its own platform into the engine's per-platform config files. This chapter is how that works, why it is built this way, and what to keep in mind when adding one.

## The first one: Vulkan's fence on NVIDIA

The reason the mechanism exists. On Linux, with an NVIDIA card (seen with drivers 595.84 and 595.104.02, engines 5.8.2 and 5.8.3, an RTX 4060, COSMIC on Wayland with the editor under XWayland), the editor segfaulted one to three times a day inside the driver:

```text
vkWaitForFences
FVulkanFence / FVulkanSwapChain::AcquireImageIndex
FVulkanViewport::Present
```

Every one of those crashes came within two seconds of `AcquireNextImage() failed due to the outdated swapchain` in the log. The swapchain goes out of date when a window changes: a secondary window closes (Project Settings, Message Log, the PIE window), PIE stops, a window is resized. The engine then recreates the swapchain, passing the old one as `oldSwapchain`, and acquires the next image with a CPU fence, because `r.Vulkan.CpuWaitForFence` is `1` by default; `vkWaitForFences` crashes in the driver. It happened in the Project Browser too, with no project open and no DLSS, so neither a project nor Streamline (stubbed on Linux) was the cause.

`r.Vulkan.CpuWaitForFence=0` makes the engine wait on a GPU semaphore and never call `vkWaitForFences` there. Epic already ships it as the default on Android (`Engine/Config/Android/AndroidEngine.ini`: "Use semaphores instead of fences for image wait"). The variable is `ReadOnly`: the console refuses it, but it is read at RHI start-up from the `[ConsoleVariables]` section of the Engine config, so it has to be in an `.ini`, not set from code at run time. With it in place the crashes stopped.

A packaged game runs the same `VulkanRHI`, and its swapchain goes out of date on resize, fullscreen toggles, alt-tab and resolution changes: players on Linux with NVIDIA are exposed too. On Windows the projects use DirectX 12 and on macOS Metal, and the path never runs.

Reported to Epic: [Linux Vulkan editor crash (SIGSEGV) in the NVIDIA driver in FVulkanSwapChain::AcquireImageIndex](https://forums.unrealengine.com/t/linux-vulkan-editor-crash-sigsegv-in-nvidia-driver-in-fvulkanswapchain-acquireimageindex-waitforfence-after-out-of-date-swapchain-under-xwayland-workaround-r-vulkan-cpuwaitforfence-0/2839784).

## macOS: `xcodebuild` without a stdin

Not a setting but a workaround in the CLI, and the same kind of problem: the engine on one platform, and a fix Rusteal applies until Epic's lands.

On macOS 27 with Xcode 27 (seen with engines 5.8.2 and 5.8.3), a game target fails at its last step, for any C++ project, the engine's blank template included:

```text
Run custom shell script 'Touch UBT generated tiles'
** BUILD FAILED **
ERROR: Failed to finalize the .app with Xcode. Check the log for more information
```

Only Apple platforms have that step: after linking, the `ApplePostBuildSync` action runs `xcodebuild` on a stub Xcode project to assemble and sign the `.app`. The UBT log shows nothing more; the Xcode activity log (`~/Library/Developer/Xcode/DerivedData/<Project>_Mac_<Target>-*/Logs/Build/*.xcactivitylog`) has `Failed to launch task /bin/sh: ... Bad file descriptor`: Xcode could not start the scheme's pre-action script.

- UE 5.8 always runs a build's actions through the UBA executor (`ExecutorFactory`; `-NoUBA` only turns detouring off). The `xcodebuild` the action starts has no file descriptor 0: `lsof` shows only 1 and 2. Its parent, `UnrealBuildTool -Mode=ApplePostBuildSync`, has a stdin, which its child does not inherit.
- The same `xcodebuild` command fails the same way when run by hand with stdin closed (`<&-`), and succeeds with stdin from a terminal, `/dev/null` or a pipe. The action, run by hand outside the executor, succeeds. Older Xcode versions presumably tolerated the closed stdin; Xcode 27 does not.
- The receipt (`Binaries/Mac/<Target>.target`) is only written when the build succeeds, so `BuildCookRun -skipbuild` after a hand-run finalize stops at `Missing receipt`, and a second build runs the action again and fails again.

`rusteal package` works around it (`rusteal-cli/src/xcode_shim.rs`). With Xcode 27 or newer, it points `DEVELOPER_DIR` at a mirror of the active Xcode's `Developer` directory: symbolic links to everything, but for `usr/bin/xcodebuild`, a script that runs the real one with `/dev/null` as its stdin. The engine finds Xcode with `xcode-select -p`, which follows `DEVELOPER_DIR`, so every tool it runs goes through the mirror, and only `xcodebuild` changes. Two constraints shape it:

- the mirror is inside a directory named `XcodeShim.app`: `xcrun` refuses a developer directory that is not inside an application bundle;
- it lives in the system's temporary directory, one per run, removed at the end: UBT builds the `xcodebuild` command line without quoting its path, so a path with a space (`Unreal Projects`) would break it.

Remove it once the engine gives `xcodebuild` a stdin. Probably the same issue: [UE5.8 Unable to build Game on macOS 27](https://forums.unrealengine.com/t/ue5-8-unable-to-build-game-on-macos-27/2746676/6). Reported to Epic: [[5.8][Mac] "Failed to finalize the .app with Xcode" on Xcode 27: xcodebuild started without stdin](https://forums.unrealengine.com/t/5-8-mac-failed-to-finalize-the-app-with-xcode-on-xcode-27-xcodebuild-started-without-stdin-bad-file-descriptor-in-touch-ubt-generated-tiles/2839864).

Two other parts of `rusteal package` are Mac-specific, and not workarounds:

- `-package` is passed to `BuildCookRun` (`HostPlatform::uat_package_flags`). On Mac, `-archive` copies `Binaries/Mac/<Game>.app`, and that bundle only receives the cooked data and the staged Rust libraries (`Contents/UE/...`) in the package step; without it the archived game was the bare executable. On Linux and Windows `-archive` copies the staged directory, and nothing changes there.
- A packaged game loads its libraries in place, without the `_hot_` load copy (`#if WITH_EDITOR` in `RustealModule.cpp`): it never reloads, and on Mac the copy would be written into the signed bundle.

## Where an `.ini` setting can go

| File | Read by | Notes |
|---|---|---|
| `Engine/Config/ConsoleVariables.ini`, `[Startup]` | the editor and Development builds | Compiled out of Shipping and Test (`#if !DISABLE_CHEAT_CVARS`): never for something the game needs. |
| `Engine/Config/Linux/LinuxEngine.ini`, `[ConsoleVariables]` | every project and every game packaged with that engine install, and the Project Browser | Lost when the engine is reinstalled or upgraded (a 5.8.4 install replaces `Engine/Config`). |
| `<Project>/Config/Linux/LinuxEngine.ini`, `[ConsoleVariables]` | that project's editor and game, on Linux only | Versioned with the project; survives engine upgrades; this is the one Rusteal writes. |

The engine one, as it was added on the development machine, in the section the engine's file already has:

```ini
[ConsoleVariables]
; larger timeout since drivers may take longer time
g.TimeoutForBlockOnRenderFence=60000
memory.MemoryPressureCriticalThresholdPercent=10
; Use semaphores instead of fences for swapchain image wait (avoids NVIDIA driver crash in vkWaitForFences after an out-of-date swapchain)
r.Vulkan.CpuWaitForFence=0
```

and the project one, in a project that had no `Config/Linux` before:

```ini
[ConsoleVariables]
; Use semaphores instead of fences for swapchain image wait (avoids NVIDIA driver crash in vkWaitForFences after an out-of-date swapchain)
r.Vulkan.CpuWaitForFence=0
```

Both are needed on a machine that also shows the crash in the Project Browser: the engine's file covers what runs before a project is loaded; the project's file is what goes with the game when it is packaged elsewhere (another machine, CI) or with an engine that does not have the change. Startup confirms either in the log:

```text
LogConfig: Set CVar [[r.Vulkan.CpuWaitForFence:0]]
```

An engine-level change is per machine and invisible to the project: keep it out of what Rusteal manages, and repeat it after every engine upgrade.

## Declared in `rusteal.toml`

```toml
[optimizations.linux.Engine.ConsoleVariables]
"r.Vulkan.CpuWaitForFence" = 0
```

`[optimizations.<platform>.<File>.<Section>]`, with the platforms `linux`, `windows` and `mac` and any config file name (`Engine`, `Game`, `Input`, ...). The section name is the `.ini` section, quoted when it has dots or slashes (`"/Script/Engine.RendererSettings"`). Values are TOML strings, integers, floats or booleans, written as the engine reads them (`True`, `False`).

The default table is `rusteal-cli/optimizations.toml`, embedded in the CLI (`optimizations::DEFAULTS`). It has nothing for `windows` or `mac` yet. Every template's `rusteal.toml.tera` ends with the same text, so a new project shows its optimizations and the game developer edits them in place. `every_template_carries_the_default_optimizations` fails if a template drifts from the defaults: change `optimizations.toml` and the templates together. A project whose `rusteal.toml` has no `[optimizations]` table, every project made before this existed, gets the defaults; an empty `[optimizations]` table means none. Unknown platforms and non-scalar values are refused when the file is read.

## Written by the CLI, chosen at compile time

`rusteal-cli/src/optimizations.rs`. The platform is not detected at run time: `optimizations::HOST` is a `#[cfg(target_os = ...)]` constant, as `build_cmd::HostPlatform::CURRENT`. The Linux CLI only ever writes `optimizations.linux` into `Config/Linux/`, the Windows CLI only `optimizations.windows` into `Config/Windows/`, the macOS CLI only `optimizations.mac` into `Config/Mac/`. A Windows or macOS machine never touches the Linux settings, and the engine stages `Config/Linux` for Linux builds only, so a Windows or macOS package carries none of them.

The settings are the same on every Linux machine, whatever its GPU. Writing them only where an NVIDIA card is found was rejected: the file is versioned, and machines that disagree would rewrite it back and forth in every commit; the player's GPU is unknown at package time anyway; and the setting costs nothing on other drivers.

`apply(root, switch)` goes over the `.ini` files of `Config/<Platform>/` and those the table names:

1. strip the block between `BEGIN` and `END` (two comment lines: `; Rusteal optimizations, ...: begin` and `; Rusteal optimizations: end`), keeping the user's text around it;
2. with `Switch::On`, render the table as its own sections inside a new block, leaving out every key the user's text already sets in that section (the user's value wins, and the key is not written twice);
3. write the user's text and the block, or delete the file when nothing is left of it, which is the case of a file Rusteal created itself.

A file with no Rusteal block and nothing to add is never rewritten, not even to normalize its line endings. Writing a separate section block at the end of a file that already has `[ConsoleVariables]` is valid because the engine merges repeated sections: `FillFileFromBuffer` (`ConfigCacheIni.cpp`) calls `FindOrAddSectionInternal` for every section header. The user's own lines must stay outside the markers: whatever is between them is rewritten.

## When it runs

| Command | Call |
|---|---|
| `rusteal new` | `optimizations::report("new", ...)` once the project files exist, before the plugins are installed. |
| `rusteal watch` | `report("watch", ...)` before the first snapshot: each developer, on each machine, gets the settings by starting the watch. |
| `rusteal package` | `Snapshot::take`, `report("package", ...)`, `BuildCookRun` (`build_cmd::run_cmd_code`, which returns the exit code instead of exiting), `Snapshot::restore`, then the exit code. |

`--optimizations <on|off>` (`Switch::parse`: `on`/`off`, `1`/`0`, `true`/`false`; `on` by default) is on the three commands. `watch --optimizations off` removes the block until the next `watch`; `package --optimizations off` ships without it and restores the project as it was, so the developer's editor keeps whatever state it had. A package interrupted with `Ctrl+C` skips the restore; the next `rusteal watch` rewrites the block.

`rusteal build` does not write them: the build must not change versioned files behind the user's back, and the editor reads the `.ini` when it starts, which `new` or `watch` has already prepared.

## Adding one

1. Find the root cause and check the setting on the platform's editor and on a packaged game: the engine log shows `LogConfig: Set CVar [[name:value]]` when a console variable is applied at start-up.
2. Check whether it can be set at run time instead. A setting the game code can change after start-up belongs in Rust, behind `#[cfg(target_os = "...")]`, where the game crate is compiled per platform; the `.ini` route is for what the engine reads before Rust is loaded (`ReadOnly` console variables, RHI and platform settings).
3. Add it to `rusteal-cli/optimizations.toml` with a comment saying what it fixes, and append the same lines to every `rusteal-cli/templates/*/*/rusteal.toml.tera`; `cargo test -p rusteal` checks they agree.
4. Document it in the game developer's handbook (`cli/optimizations.md`) and in this chapter.

Existing projects pick a new default up only if their `rusteal.toml` has no `[optimizations]` table; the others list their own and add it by hand, which the release notes must say.
