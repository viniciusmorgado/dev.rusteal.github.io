# Types declared in Rust

A game's Rust library declares classes (`#[uclass]`), structs (`#[ustruct]`), enums (`#[uenum]`) and delegates (`#[udelegate]`), and the engine sees them as its own: the editor lists them, Blueprints use them, assets save them. None of them exist when the engine starts. The Rusteal plugin creates them when it loads the library, from what the macros registered.

## Registration

Each macro submits a registration with the `inventory` crate, which collects them across the library at link time:

| Macro | Registration | What it creates |
|---|---|---|
| `#[uenum]` | `EnumRegistration { create }` | A `UEnum` |
| `#[ustruct]` | `StructRegistration { create, register, finalize }` | A `UScriptStruct` and its properties |
| `#[uclass]` | `ClassRegistration { create, register, finalize, after_defaults }` | A `URustealReifiedClass` and its properties and components |
| `#[uclass_impl]` | `ClassFunctionRegistration { register_functions, class_defaults }` | The class's `UFunction`s, delegates and class defaults |

`reify_registry::register_all_from_inventory` runs them in dependency order:

1. **Enums**, first, since properties and function parameters of every other kind can hold them.
2. **Classes are created**, empty, parents before children (a pass repeats until no class is left waiting for its parent).
3. **Structs** are created, then given their properties, then finalized (linked, sized).
4. **Class properties and components** are added.
5. **Functions** of every class.
6. **Classes are finalized**, then their `#[class_defaults]` run on the class default object. Each registration lists the type ids of its parent and of its components' classes (`depends_on`), and this step runs in an order where those come first: a class's default object creates its components from their classes' default objects, which must be finalized and set up by then.

Every step calls into the plugin through the reify table of the API table (`RustealReifyApi`): `create_enum`, `create_struct`, `create_class`, `add_property`, `add_function`... The objects go in the library's package: `/Script/Rusteal` for the game, `/Script/<Plugin>` for a plugin's library.

## Enums

`#[uenum]` turns a Rust enum with unit variants into a `UEnum`:

- the macro keeps the enum as Rust code, `#[repr(u8)]`, and implements `UeEnum` on it (`static_enum`, `to_i64`, `from_i64`), the trait the bindings' engine enums implement too;
- `create_enum` receives the name, the variant names joined by newlines and their values, and builds the `UEnum` with `SetEnums` in the `EnumClass` form (`EGrabType::Snap`), marked `BlueprintType`; Unreal adds the `_MAX` entry, which is why values stop at 254.

Properties of the enum's type are `FEnumProperty`s over a byte, which is also how the macros pass enums to and from `#[ufunction]`s: the parameter is read as the enum's representation and turned back into the Rust enum with `from_i64`. A compile-time check refuses an enum stored on more than one byte.

## Hot reload

A hot reload loads a new library and registers everything again. Classes are replaced; structs and enums are not: `create_struct` and `create_enum` find the existing object by name and return it, and warn when its fields or values changed, which takes an editor restart.
