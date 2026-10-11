# Template directories

How a template is laid out in the repository, and what `rusteal new` does with it. How an engine template becomes one is in [Porting an engine template](porting.md), and features in [Features](features.md).

A template is a directory under `rusteal-cli/templates/`, one directory per
variant inside it, `base` being the template without `--variant`; they are
embedded in the binary when it is built, and nothing else registers them. A
variant directory mirrors the root of the project it creates:

- `template.toml` names the engine template the project starts from
  (`engine_template`), the paths of it to leave out (`exclude`: the C++
  gameplay the template replaces, the other variants), the level to open
  when it is not the engine template's own (`default_map`), a one-line
  `description` and the `next_step` printed at the end;
- every other file is written into the project over the engine template's:
  `*.tera` files are rendered with [Tera](https://keats.github.io/tera/) and
  lose the extension, the rest (Blueprints, meshes) is copied as is;
- `{{ variable }}` works in folder and file names too:
  `Rust/{{crate_name}}/src/lib.rs.tera`.

The context is the same for every template: `project`, `crate_name`,
`version`, `glam_version` and, with `--runtime-path`, `runtime_path`.

Each variant is complete on its own. A new one starts as a copy of the
closest existing one (a variant from its template's `base`) and is changed
from there, never layered on top of it. Its Blueprints come from a project
where they were made and played, saved with the engine version Rusteal
targets. A variant with something to see has a screenshot in
`assets/templates/`, shown on the template's page in the [game developer's handbook](https://viniciusmorgado.github.io/game.rusteal.github.io/getting_started/templates.html).

A plugin template is a directory under `rusteal-cli/plugin_templates/`,
written into `Plugins/<Name>/` by `rusteal plugin new --template <name>`:
its `template.toml` has the `description` and the `next_step`, and its files
are rendered the same way, with `plugin`, `crate_name`, `version`,
`glam_version` and `runtime_path` as context. The plugin's module, its
`Build.cs` and `<Name>Module.cpp`, come with the template: copy them from
`blank` (a runtime module) or `editor` (an editor module).

A plugin template with a `[feature]` table in its `template.toml` is a
feature: `plugin` is the plugin's name in the project (its assets live under
`/<plugin>/`, so a feature with `Content/` keeps that name), and
`shared_packs` lists the engine template packs it needs
(`High/Characters` is `Templates/TemplateResources/High/Characters`), copied
into the project's `Content/` without replacing what is there. A template's
`template.toml` bundles features with `features = [...]`.
