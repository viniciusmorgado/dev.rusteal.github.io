# Features

A feature is gameplay that is not a template of its own but that any template can take: the Intro To Unreal template's stations (a mannequin copying the player, checkpoints, triggers and doors) are worth having in a third person game too. A feature is packaged as a Rusteal plugin, so its Rust library, its assets and its configuration stay apart from the game's and come along with `rusteal new`.

## A plugin template with a `[feature]` table

Features live with the plugin templates, in `rusteal-cli/plugin_templates/<feature>/`. What makes one a feature is a table in its `template.toml`:

```toml
description = "Mannequins that copy the player's movement, as the Intro To Unreal template's, in a Mannequin plugin."
next_step = "..."

[feature]
plugin = "Mannequin"
shared_packs = ["High/Characters"]
```

- `plugin` is the plugin's name in the project. A feature's assets are mounted at `/<plugin>/`, and Blueprints and levels refer to them by that path, so a feature with a `Content/` directory is always created under this name: `rusteal plugin new` refuses another one for it.
- `shared_packs` are engine template packs, `Templates/TemplateResources/<Level>/<Pack>`, the feature's classes or assets use. They are copied into the project's `Content/<Pack>/` file by file, skipping what the project already has, so a template that ships a newer copy of a pack keeps it.

`templates::features()` lists the plugin templates that have the table, `select_feature` resolves a name as loosely as the templates (`Mannequin`, `mannequin`), and `listing()`, printed when `rusteal new` gets an unknown template, ends with them.

## How `rusteal new` adds them

`NewOptions::features` holds the `--feature` names. `run_new` resolves them, after the ones the template bundles (`features = [...]` in a template's `template.toml`), before writing anything, so a wrong name stops it early; a feature named twice is added once. Once the project files and the Rusteal plugins are written, each feature goes through `plugin_cmd::create_plugin`, the part of `rusteal plugin new` that writes a plugin without building it:

1. check the name (letters and digits, not Rusteal's modules nor the project's);
2. render the plugin template into `Plugins/<plugin>/` with the plugin context (`plugin`, `crate_name`, `version`, `glam_version`, `runtime_path` from the project's pins);
3. write the bindings placeholder and the C++ stubs the plugin's module compiles against before `rusteal build` generates the real ones;
4. enable the plugin in the `.uproject`;
5. copy the feature's shared packs.

The build that follows is the project's one: `rusteal build` finds every Rusteal plugin under `Plugins/` and builds and deploys each library after the game's. `run_new` prints each feature's `next_step` after its own.

## Validating a feature

A feature is validated in a project of a template it does not come from: `rusteal new RsFeat --template third-person --feature mannequin` with `--runtime-path`, then a headless Play In Editor test that loads the plugin's classes by their `/Script/<Plugin>.` path, places what it needs in the editor world before Play (Python cannot spawn into the PIE world), and drives the player (`add_movement_input` every tick: a test step that waits a tick between inputs lets the character brake to a stop).

## Features made from a template's Blueprints

The Intro To Unreal features come from that template's Blueprints, which the `intro` template keeps where the engine has them (`/Game/DemoTemplate/...`), reparented to the features' Rust classes so the template's level works unchanged. A feature used in another template needs the same Blueprints and what they use (materials, meshes, widget layouts, Niagara systems, MetaSounds) inside its plugin, mounted at `/<Plugin>/`. They are produced from the rebased `intro` project, one feature at a time:

1. Copy the rebased project, without its levels and `__ExternalActors__` (nothing there is needed, and moving assets would rewrite every placed actor that uses them).
2. In a headless editor, collect the feature's Blueprints and, through the asset registry, every package under `/Game/DemoTemplate/` they depend on.
3. Move them all at once with `AssetTools.rename_assets` to the same relative paths under `/<Plugin>/`, which rewrites their references to each other, then save the plugin's content.
4. Check that no moved asset still depends on `/Game/DemoTemplate/`, and copy `Plugins/<Plugin>/Content` into the feature's plugin template.

The move runs in the `intro` project itself, on its rebased content with the levels and external actors deleted, and the content is put back afterwards: a copy of the project at another path loads the Rusteal module twice and stops. The editor refuses renames while the asset registry is still scanning, so the script waits for it, and a plugin without a `Content` folder is not mounted, so an empty one is created first.

Each feature starts again from the rebased project, so an asset two features use (a material, a widget) is moved into both and neither plugin depends on the other. The shared engine packs (`Characters`, `LevelPrototyping`) stay where the feature's `shared_packs` put them, under `/Game/`.
