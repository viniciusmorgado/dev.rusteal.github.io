# Types declared in Rust

A game's Rust library declares classes (`#[uclass]`), interfaces (`#[uinterface]`), structs (`#[ustruct]`), enums (`#[uenum]`) and delegates (`#[udelegate]`), and the engine sees them as its own: the editor lists them, Blueprints use them, assets save them. None of them exist when the engine starts. The Rusteal plugin creates them when it loads the library, from what the macros registered.

## Registration

Each macro submits a registration with the `inventory` crate, which collects them across the library at link time:

| Macro | Registration | What it creates |
|---|---|---|
| `#[uenum]` | `EnumRegistration { create }` | A `UEnum` |
| `#[uinterface]` | `InterfaceRegistration { create, register, finalize }` | A `UClass` with `CLASS_Interface` and its `UFunction`s |
| `#[ustruct]` | `StructRegistration { create, register, finalize }` | A `UScriptStruct` and its properties |
| `#[uclass]` | `ClassRegistration { create, register, finalize, after_defaults }` | A `URustealReifiedClass` and its properties and components |
| `#[uclass_impl]` | `ClassFunctionRegistration { register_functions, class_defaults }` | The class's `UFunction`s, delegates and class defaults; on a trait implementation, the interface too |

`reify_registry::register_all_from_inventory` runs them in dependency order:

1. **Enums**, first, since properties and function parameters of every other kind can hold them.
2. **Interfaces are created**, empty, so classes can implement them.
3. **Classes are created**, empty, parents before children (a pass repeats until no class is left waiting for its parent).
4. **Structs** are created, then given their properties, then finalized (linked, sized).
5. **Interface functions** are added and the interfaces finalized: their parameters can be structs, enums and classes, all created by now.
6. **Class properties, components and interfaces** are added.
7. **Functions** of every class.
8. **Classes are finalized**, then their `#[class_defaults]` run on the class default object. Each registration lists the type ids of its parent and of its components' classes (`depends_on`), and this step runs in an order where those come first: a class's default object creates its components from their classes' default objects, which must be finalized and set up by then.

Every step calls into the plugin through the reify table of the API table (`RustealReifyApi`): `create_enum`, `create_struct`, `create_class`, `add_property`, `add_function`... The objects go in the library's package: `/Script/Rusteal` for the game, `/Script/<Plugin>` for a plugin's library.

## Enums

`#[uenum]` turns a Rust enum with unit variants into a `UEnum`:

- the macro keeps the enum as Rust code, `#[repr(u8)]`, and implements `UeEnum` on it (`static_enum`, `to_i64`, `from_i64`), the trait the bindings' engine enums implement too;
- `create_enum` receives the name, the variant names joined by newlines and their values, and builds the `UEnum` with `SetEnums` in the `EnumClass` form (`EGrabType::Snap`), marked `BlueprintType`; Unreal adds the `_MAX` entry, which is why values stop at 254.

Properties of the enum's type are `FEnumProperty`s over a byte, which is also how the macros pass enums to and from `#[ufunction]`s: the parameter is read as the enum's representation and turned back into the Rust enum with `from_i64`. A compile-time check refuses an enum stored on more than one byte.

## Interfaces

`#[uinterface]` keeps the trait as Rust code and builds an interface from its functions:

- `create_interface` makes a plain `UClass` (not a `URustealReifiedClass`: an interface has no instances, so no Rust data) whose parent is `UInterface`, with `CLASS_Interface | CLASS_Abstract` and the metadata `IsBlueprintBase` and `BlueprintType`, which is what the editor's interface picker (`FKismetEditorUtilities::IsClassABlueprintImplementableInterface`) and variable types look for;
- each function is an `add_function` with no callback and `FUNC_Event | FUNC_BlueprintEvent | FUNC_BlueprintCallable`, a C++ `BlueprintImplementableEvent`, with its parameters; `finalize_interface` binds and links them, then the class, and makes its default object.

The macro also implements `UeInterface` on `dyn Trait`, which is how Rust names the interface (`implements::<dyn Interactable>()`): Rust has no type of the trait's name to put it on, and the trait object type is the crate's own. An inherent `impl dyn Trait` carries the hidden pieces the other macros read: the interface handle, and for each function its Unreal name and parameter names (`__rusteal_signature_<fn>`).

A Rust class implements the interface with `#[uclass_impl]` on `impl Trait for Class`. That registration adds the interface to the class (`add_interface_class`, an `FImplementedInterface` with no pointer offset and `bImplementedByK2`, as for a Blueprint) and a native function per method, named and with parameters named as the trait's (read from `__rusteal_signature_<fn>`, since an implementation may name its parameters differently), with `FUNC_Native | FUNC_Event | FUNC_BlueprintEvent`: a C++ `BlueprintNativeEvent`, which a Blueprint child can override. The functions are found by name (`FindFunction`, which also looks in the class's interfaces), so a call from a Blueprint message, `ProcessEvent` or a `DynamicCall` reaches the class's function, or the interface's empty one when the class has none; that is why such a call returns the default value instead of failing.

Rust calls the interface through `ScriptInterface<dyn Trait>`, which the macro makes implement the trait: each method is a `DynamicCall` by name on the object, with the trait's parameter names. `core_is_a` answers `ImplementsInterface` when its class is an interface, so `UObjectRef::interface` and `implements` are the casts.

## Booleans

A `bool` property or parameter is an `FBoolProperty` made a native bool (`SetBoolSize(sizeof(bool), true)`), the kind UHT makes for a C++ `bool`: it takes a whole byte, written as 0 or 1. `FBoolProperty`'s constructor makes a one-bit field instead, whose writes change only that bit; Rust reads the byte, so the bits left in a parameter buffer made a `false` from Python or a Blueprint arrive as `true`.

## Hot reload

A hot reload loads a new library and registers everything again. Classes are replaced; structs and enums are not: `create_struct` and `create_enum` find the existing object by name and return it, and warn when its fields or values changed, which takes an editor restart.

Interfaces are rebuilt in place instead. The Blueprints that implement an interface hold its class (`FBPInterfaceDescription::Interface`), and every class implementing it lists it in `Interfaces`; replacing the class would leave all of them pointing at the old one. An interface has no instance data, only functions, so `create_interface` keeps the class when the macro's shape (the functions' signatures) changed and retires its functions: they leave the class's `Children` and function map, and are renamed out of the way and marked transient, for the bytecode still referring to them until its Blueprint compiles again. The new functions are added and linked, and the function caches of the classes implementing the interface (`AllFunctionsCache`, which `FindFunctionByName` fills with interface functions) are cleared. An interface whose shape did not change is kept as it is: `add_function` and `add_function_param` do not touch an interface that is not being built.

After the reload, the editor module's interface refresher (`RustealSetInterfaceRefresher`) conforms and recompiles the loaded Blueprints that implement or call a changed interface, as the Blueprint editor does when a Blueprint Interface changes, and refreshes the interface's Blueprint actions. It runs after the classes are reinstanced, since a Rust class implementing a changed interface changed too (its trait implementation is part of its shape) and is replaced, with its Blueprint children recompiled. Without the editor module (`-game`), a changed interface is kept as it was, with a warning.
