# Releases and versions

Everything Rusteal ships carries one version: the seven crates (they inherit
`[workspace.package].version` through `version.workspace = true`), the two UE
plugins and the FFI contract between them.

**Where it comes from.** The release is cut from a `develop` → `main` PR. The
`prepare-release` workflow takes the last released version from crates.io
(`max_version` of the `rusteal` crate, the one source that cannot drift) and
bumps it by the commits the PR adds: `BREAKING CHANGE:` → major, `feat:` →
minor, anything else → patch. It never reads the version from the repository's
files. If nothing that goes into a package changed since the last tag, it skips
the release.

**Where it is written.** The same bump commit writes it everywhere at once:

| File | Field | 0.2.1 becomes |
|---|---|---|
| `Cargo.toml` | `[workspace.package].version` and the internal `{ version, path }` dependencies | `0.2.1` |
| `ue_plugin/*/*.uplugin` | `VersionName` | `"0.2.1"` |
| `ue_plugin/*/*.uplugin` | `Version` | `2001` |

Tags (`v0.2.1`), crates.io and the manifests only ever use `major.minor.patch`.

**The integer form.** Two places cannot hold `0.2.1`: the `.uplugin` `Version`
field, which UE requires to be an integer, and `RustealApiTable::version`, a
`u32`. Both use the same encoding, `major * 1_000_000 + minor * 1_000 + patch`
(`rusteal_ffi::encode_version`). It is always derived from the version, never
a source of it. Minor and patch must stay below 1000, or the number would be
ambiguous; `encode_version` refuses such a version at compile time, so it cannot
slip through.

**How it is kept in line.**

- `rusteal-cli/tests/plugin_version.rs` fails if either `.uplugin` disagrees with
  the workspace version, so a descriptor edited by hand turns CI red.
- At load, the plugin compares its own version with the library's
  `rusteal_version()` export and refuses a library built against another one;
  `rusteal_runtime::init` checks the table's `version` again.
- Before `build`, `generate` and `setup`, the CLI compares the project's pins
  and plugins with its own version (see [Versions and upgrades](https://viniciusmorgado.github.io/game.rusteal.github.io/getting_started/versions.html)).

`Cargo.lock` is not versioned: every build would rewrite the internal versions
in it. A dependency release that breaks the workspace shows up in CI, where it
gets fixed. `cargo install rusteal --locked` still works, because cargo packs a
lock file of its own into the published crate.

## The pipeline

Four workflows in `.github/workflows/` carry a change from a branch to crates.io:

| Workflow | Runs on | What it does |
|---|---|---|
| `ci.yml` | pushes to `develop`, pull requests to `develop` and `main` | build, tests and Clippy of the workspace, and `cargo audit`, which fails on any dependency with a RustSec vulnerability advisory |
| `prepare-release.yml` | every pull request to `main` | what the release will do after the merge, before it: the version, the tests, Clippy, the plugin snapshot and the packaging of every crate as it will be published |
| `release.yml` | a push to `main` | publishes the crates in dependency order and creates the tag |
| `sync-develop.yml` | the end of a Release run | fast-forwards `develop` to `main`, so the version commit reaches it |

**Prepare release**, in order:

1. **Anything to release?** Nothing that goes into a package changed since the last tag: no release, and the job ends there.
2. **The next version.** The published `max_version` of the `rusteal` crate, bumped by the subjects of the commits since that version's tag (`v<max_version>`; the last `v*` tag when that one is missing, `origin/main` when there is no tag at all). It never goes below the version already in `Cargo.toml`: a release that failed after its version commit does not make the next one lower.
3. **The version commit.** Written into the pull request's branch with the CI skip marker, so it does not start runs that would wait for approval (a commit pushed by the bot to a pull request needs one).
4. **The checks**, on that commit: `cargo test --workspace`, Clippy, `cargo audit`, the plugin snapshot sync, and `cargo +stable package --workspace --exclude xtask`, which builds every crate from its packaged sources. The pinned Rust 1.88 Cargo cannot package a workspace whose internal dependencies are not published yet; the stable one can.
5. **The statuses.** Because the version commit runs no workflow, Prepare release posts the `build` and `bump` statuses on it itself, once the checks passed.

The `main` branch has a ruleset ("main: release checks") that requires `build` and `bump`, with no bypass: a release pull request merges only when Prepare release validated what will be published, so `main` never ends up with a version that does not publish.

**Release** repeats the tests, reads the version the merge brought, syncs the plugin snapshot, publishes the crates (skipping one whose version is already on crates.io, so a re-run finishes a partial release) and pushes the tag. Its own commits carry the CI skip marker too.

Never write a CI skip token in a commit message, a pull request's title or its body, even to talk about one: GitHub reads it anywhere in the head commit's message and runs nothing for that push or pull request. Call it "the CI skip marker" instead.
