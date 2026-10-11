# Tracking documents and conventions

## Tracking documents

Engine APIs Rusteal uses that Unreal has deprecated are tracked in
[`docs/ue-deprecations.md`](https://github.com/viniciusmorgado/rusteal/blob/main/docs/ue-deprecations.md): what, since which UE
version, until when and where. A new engine version means checking its
warnings against that list.

What Rusteal still lacks to write Unreal's Third Person template entirely in
Rust, in the order the template needs it, is mapped in
[`docs/third-person-gaps.md`](https://github.com/viniciusmorgado/rusteal/blob/main/docs/third-person-gaps.md).

How the engine templates' variants were ported, where the Rust ports differ
from the C++ and what they still work around is in
[`docs/template-variants.md`](https://github.com/viniciusmorgado/rusteal/blob/main/docs/template-variants.md).

What Rusteal does not do yet for plugins (Slate, details customizations,
importers, Blueprint nodes...), and what each takes, is in
[`docs/plugin-roadmap.md`](https://github.com/viniciusmorgado/rusteal/blob/main/docs/plugin-roadmap.md).

What Unreal Engine 6 changes for Rusteal — Verse, Scene Graph, the end of
Blueprints — and the open questions to check as Epic publishes details are in
the [UE6 radar](https://github.com/viniciusmorgado/rusteal/blob/ue6/docs/ue6-radar.md), kept on the `ue6` branch (see
[Future migration to UE6](https://github.com/viniciusmorgado/rusteal#future-migration-to-ue6)).

## Commits

Commits are small, one per fix. Every subject starts with a [Conventional Commits](https://www.conventionalcommits.org) type, because the release reads them to pick the version (see [Releases and versions](releases.md)): `feat:` or `feat(scope):` for a new capability (minor), `fix:`, `docs:`, `refactor:`, `test:`, `perf:`, `build:`, `ci:` and `chore:` for the rest (patch), and `BREAKING CHANGE:` as the subject for an incompatible change (major). A branch that brings a feature has at least one `feat` commit.
