# The workspace

| Crate | Role |
|---|---|
| `rusteal-ue-flags` | UE reflection flag constants |
| `rusteal-ffi` | `#[repr(C)]` types and the API table: the Rust ↔ C++ contract |
| `rusteal-core` | the safe runtime: object references, lifecycle, containers, math |
| `rusteal-macros` | `#[uclass]`, `#[uclass_impl]` and their attributes |
| `rusteal-runtime` | what a game depends on; re-exports the above |
| `rusteal-codegen` | reflection JSON → the `bindings` crate and the C++ wrappers; owns `manual/` |
| `rusteal-cli` | the `rusteal` binary; owns `templates/` and `plugin_templates/`, and embeds `ue_plugin/` |

Changing the C++ plugin means running `cargo run -p rusteal -- sync-plugin`
before publishing, which refreshes the snapshot the binary embeds.
