# Porting an engine template

`rusteal new --template <name>` starts from one of the engine's own templates and puts its gameplay in Rust. This chapter is how a template gets there, from the engine's C++ or Blueprints to `rusteal-cli/templates/<template>/<variant>/`.

## What a Rusteal template is

A directory under `rusteal-cli/templates/` per template and variant, embedded in the CLI by its `build.rs`:

| File | What it is |
|---|---|
| `template.toml` | The manifest: the engine template it starts from, the paths of it to leave out, the default map, the line printed at the end. |
| `rusteal.toml.tera` | The project's Rusteal configuration: the modules to bind, by feature. |
| `Rust/` | The Cargo workspace and the game crate, with the template's classes. |
| `Content/` | The template's assets that differ from the engine's: the Blueprints reparented to the Rust classes, the levels resaved with them. |
| `Source/` | The game module's build rules, when the template needs other engine modules. |

`rusteal new` copies the engine template with the engine's own recipe (its `Config/TemplateDefs.ini`: ignored folders and files, renames, text replacements, shared content packs), then writes the template's files over it. A `.tera` file is rendered with the project's name and crate name first.

## C++ templates

The engine's C++ templates (`TP_ThirdPerson`, `TP_VehicleAdv`...) have their gameplay in C++ classes and data in Blueprint children of those classes. The port:

1. **Writes the Rust classes**, one per C++ class, with the same Unreal names, properties, functions and events. C++ virtuals become the engine events Rust overrides (`Tick` is `ReceiveTick`, `SetupPlayerInputComponent` is `ReceiveRestarted`).
2. **Redirects the C++ classes to the Rust ones** while the template's assets are resaved: `[CoreRedirects]` with `+ClassRedirects=(OldName="/Script/TP_X.Class",NewName="/Script/Rusteal.Class")`. A resaved Blueprint names its Rust parent from then on.
3. **Leaves the C++ out**: the manifest's `exclude` lists the C++ files and the other variants' content.

The placed actors of a World Partition level are the exception: each actor's descriptor stores its first native class, and a Rust class is not native, so a redirect to it leaves the descriptor invalid. Those levels are resaved with the old class redirected to `/Script/Engine.Actor`; the actor then loads as its Blueprint, whose parent is the Rust class.

## Blueprint-only templates

The engine's Blueprint templates (`TP_VirtualRealityBP`, `TP_HandheldARBP`, `TP_UEIntro_BP`) have no C++ at all. `rusteal new` gives such a project a game module (`Source/<Project>/`, its two targets, the `.uproject`'s `Modules` entry), since the Rusteal plugin and the bindings are built with the project. `Template::load` notices the missing `Source/` and `write_game_module` writes it.

The port moves each Blueprint's logic to a Rust class and keeps the Blueprint as a data-only child of it. Reading the logic comes first, because a Blueprint's graphs are binary.

### Reading a Blueprint

The editor exports any asset as text with `ObjectExporterT3D`, from editor Python:

```python
task = unreal.AssetExportTask()
task.object = unreal.load_asset("/Game/XRFramework/Blueprints/BP_XRPawn")
task.filename = "/tmp/BP_XRPawn.t3d"
task.automated = True
task.exporter = unreal.ObjectExporterT3D()
unreal.Exporter.run_asset_export_task(task)
```

A Blueprint's T3D holds every graph: each node, its function or variable reference, its pins' default values and links. Walking the exec links from each event gives the logic as text, nodes in order with their data inputs inlined. The defaults a graph does not show are elsewhere: function local variables in the function entry node's `LocalVariables`, member defaults in the class default object (read by name with `get_editor_property`), components in the simple construction script (gathered with `SubobjectDataSubsystem.k2_gather_subobject_data_for_blueprint`), timeline curves in the generated class's `<Name>_Template` and its curve objects, also exportable as T3D.

### Moving the logic

For each Blueprint, a Rust class with:

- the Blueprint's **components**, under the same object names (`#[component(name = "VROrigin")]`), set up in `#[class_defaults]` with what is not an asset (transforms, collision, motion sources);
- its **variables**, under the same names and types: a Blueprint `real` is a double (`f64`), unless the variable was made single precision (`f32`); a user enum becomes a `#[uenum]`;
- its **functions and events**, as `#[ufunction]`s and overrides; its **event dispatchers** as `#[udelegate]`s; the Blueprint Interfaces it implements in `implements`, the interface assets staying where they are;
- its **timelines** as state the actor's tick plays, with the curve read from the asset;
- the **assets** its graphs referenced (input actions, mapping contexts, classes to spawn, textures) as properties the Blueprint child sets.

### Rebasing the Blueprint

Then, in a headless editor, each Blueprint is turned into a child of its Rust class:

1. Record the values of the Blueprint's own components (from their T3D, compared with their class's defaults, read back by Unreal name).
2. Remove its graphs (`BlueprintEditorLibrary.list_graph_names` and `remove_graph`), its event dispatchers (`remove_event_dispatcher`), its now unused variables (`remove_unused_variables`) and its components (`SubobjectDataSubsystem.delete_subobjects`).
3. Reparent it (`reparent_blueprint`) and compile.
4. Set the recorded values on the inherited components of the same names, and the assets on the class defaults, by their Unreal names: editor Python wraps a Rust class as its nearest engine class, so `get_editor_property("MoveAction")` works where `move_action` does not.
5. Set again the values a layout gives the widgets placed in it (`find_object` of `<WidgetBlueprint>:WidgetTree.<Name>`): they were stored under the Blueprint variables the reparent removed. Python wraps a `#[uenum]` without its `E` (`EIconType` is `unreal.IconType`), and `unreal.IconType.cast(2)` makes a value from an index.
6. Compile and save; resave the levels that place the Blueprints.

When the template's levels place the Blueprints, their components stay in the Blueprints. A placed actor saves its components' own values (a text render's text, a spline's points, the actor's place in its root) under the components' names; once the Rust parent creates a component of the same name, loading the level gives the saved values to the Rust component, marks it as created by the construction script, and the construction script that follows destroys it. The Intro To Unreal level places 102 actors this way, so its Blueprints keep their components (`no_components` in the rebase), the Rust classes find them by name (`components::named(actor, "TextRender")`), and a component the Blueprint inherited from a Blueprint parent the reparent removes is added back to it under the same name (`adopt_inherited`, the door's `Box`). A Blueprint that keeps its own `DefaultSceneRoot` cannot have a Rust parent with a component of that name either: the compiler refuses the second `DefaultSceneRoot` property.

A data table whose row structure is a Blueprint struct moves to the `#[ustruct]` that replaces it: export its rows with `DataTableFunctionLibrary.export_data_table_to_json_string` before the port, then `fill_data_table_from_json_string(table, json, rust_struct)`, which also makes the Rust struct the table's row structure. The JSON names the fields as the Blueprint struct did, so the Rust fields take those names (`name = "BGColor"`).

A variable the graphs left still use (an Anim Blueprint's AnimGraph reads the hand's pose alphas) clashes with the Rust property of the same name when the Blueprint is reparented, and the compiler renames it (`PoseAlphaGrasp_0`). In a second editor session, `replace_variable_references` moves its references to the inherited property and `remove_unused_variables` deletes it. It has to be a new session: until then Python does not see the properties the reparented class gained.

### Validating

The levels are played headless (`UnrealEditor <project> -nullrhi -ExecCmds="py test.py"`), the test driving Play In Editor from a Slate tick callback: load the map, `editor_request_begin_play`, then call the Rust classes' functions and read their properties by Unreal name (`pawn.call_method("SnapTurn", (True,))`). A template is done when a project `rusteal new` creates from it passes the same test.

## Rusteal changes the ports brought

| Port | Change |
|---|---|
| Vehicle | Structs reached from exported properties are exported even when they are not Blueprint types (the Chaos vehicle setups). |
| Vehicle | `FName` writes keep the display index the editor shows. |
| Virtual Reality | Blueprint-only engine templates. |
| Virtual Reality | `#[uenum]`; Unreal enums as `#[ufunction]` parameters and return values; struct return values for overrides (`ModifyRaw`). |
| Virtual Reality | `#[uproperty(BindWidget)]`. |
| Virtual Reality | Enum values from third-party headers (OpenXR's blend modes), getters for properties with only a `BlueprintSetter` (`MotionSource`), and no clippy on the generated bindings. |
| Handheld AR | Classes reached from exported functions and properties are exported, without functions, even when their module exports no API (`UARBasicLightEstimate`, which `GetCurrentLightEstimate` returns). |
| Handheld AR | Arrays of structs as properties (`UeArray<OwnedStruct<T>>`), the struct in the array finalized before the one holding it; `#[uenum]`s as `#[udelegate]` parameters. |
| Handheld AR | `#[uproperty(BindWidgetAnim)]` and `Transient`. |
| Handheld AR | Rust `bool` properties and parameters are native bools: they were one-bit fields, and a `false` written by Python or a Blueprint left the byte's other bits, which Rust read as `true`. |
| Intro To Unreal | Features: plugin templates with a `[feature]` table, `rusteal new --feature`, templates bundling features, plugin templates carrying content. |
