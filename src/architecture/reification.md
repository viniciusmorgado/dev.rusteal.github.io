# Reification and the macros

A `#[uclass]` struct becomes a real `UClass` when its library loads: Rusteal **reifies** it. This chapter follows one class from the macro to the engine and back: what the macros generate, where an object's Rust data lives, how the engine calls a Rust function, and how the C++ side builds the class. The order types are registered in, and enums, interfaces and booleans, are in [Types declared in Rust](../internals/rust_types.md).

## The macros (`rusteal-macros`)

Five procedural macros exist, and `rusteal-runtime` re-exports them: `#[uclass]`, `#[uclass_impl]`, `#[ustruct]`, `#[uenum]` and `#[uinterface]`. `#[uproperty]`, `#[component]`, `#[ufunction]`, `#[class_defaults]` and `#[udelegate]` are inert attributes that the outer macro parses and strips.

**Names** (`prop_type.rs`): a Rust name becomes the Unreal one in PascalCase (`move_speed` → `MoveSpeed`), with `b_` turned into UE's bool prefix (`b_can_dash` → `bCanDash`); `name = "..."` overrides it. A type's id is the FNV-1a hash of its module path and identifier (`rusteal_core::type_id(concat!(module_path!(), "::", "Name"))`, computed at compile time), so classes with the same name in two crates or two modules have different ids; `#[uclass]` gives it as `UeClass::RUST_TYPE_ID`, which is 0 for engine classes, and a class's parent and components contribute theirs through it. Two classes with the same name in one library still share an Unreal name in the library's package, which the engine refuses.

### `#[uclass]` (`uclass.rs`)

| Argument | Meaning |
|---|---|
| `parent = Path` | Required: the engine or Rust class it derives from. |
| `implements = "..."` / `[...]` | Interfaces by class path, starting with `/`. |
| `config = "Name"` | The config category its `Config` properties are saved in. |

Other arguments are ignored without an error, and so are attributes written on the struct itself.

Fields are of three kinds:

- **`#[uproperty(...)]`**: a UE property. Flags: `BlueprintReadWrite`, `BlueprintReadOnly`, `EditAnywhere`, `EditDefaultsOnly`, `VisibleAnywhere`, `SaveGame`, `Config`, `Transient`, `BindWidget`, `BindWidgetOptional`, `BindWidgetAnim` and `BindWidgetAnimOptional` (both `Transient`); values: `default = expr` (scalars and enums only), `name`, `category`. Anything else is a compile error.
- **`#[component(...)]`**: a default subobject: `root`, `attach = "field"`, `socket = "..."` (needs `attach`), `name = "..."` (the subobject's name, PascalCase of the field otherwise).
- **anything else**: a Rust-private field, never seen by the engine.

The flags map to `CPF_*` as UHT's do:

| Argument | Flags |
|---|---|
| `BlueprintReadWrite`, `BlueprintReadOnly` | `CPF_BlueprintVisible` |
| `BlueprintReadOnly` | `CPF_BlueprintReadOnly`, and `CPF_Edit` + `CPF_EditConst` unless an edit flag is given |
| `EditAnywhere`, `BlueprintReadWrite` | `CPF_Edit` |
| `EditDefaultsOnly` | `CPF_Edit`, `CPF_DisableEditOnInstance` |
| `VisibleAnywhere` | `CPF_Edit`, `CPF_EditConst` |
| `SaveGame` / `Config` / `Transient` | `CPF_SaveGame` / `CPF_Config` / `CPF_Transient` |

`category` and the `BindWidget*` flags become property metadata, which only exists in editor builds (`WITH_EDITORONLY_DATA`).

A property's type is `bool`, any integer, `f32`, `f64`, `String`, `FName`, `Text`, `UObjectRef<T>`, `SubclassOf<T>`, `SoftObjectRef<T>`, `SoftClassRef<T>`, `TWeakObjectPtr<T>`, `ScriptInterface<dyn I>`, `OwnedStruct<T>`, an engine or Rust enum (any capitalized path without generics), or a `UeArray<T>`, `UeMap<K, V>` or `UeSet<T>` of those (`prop_type::map_type`; keys and set elements are the hashable kinds). `RustealReifyPropExtra` carries what the type needs: a class, a struct, an enum, an inner element's type, and for a map its value's. A `const _` block asserts at compile time that the parent and every component type are Unreal classes.

**The expansion**, in order:

1. The struct is rewritten to `{ __obj: UObjectHandle, __rust_data: *mut __<Name>RustData }`: a handle to the engine object and a pointer to the Rust-private fields, which move to `__<Name>RustData`. That type's `Default` calls `Default::default()` on each field, so private fields must implement `Default`.
2. `UeClass` (`static_class()`, which panics if the class was never registered), `HasParent`, `Inherits<U>` through the parent, and `Deref<Target = UObjectRef<Parent>>` (a pointer cast, relying on `UObjectRef`'s `#[repr(transparent)]` layout): a Rust class has its parent's methods.
3. For each property, a getter and a `set_` setter that look up the `FPropertyHandle` once (a `static OnceLock` per accessor) and then call `property_get_*`/`property_set_*`; an array gets a getter returning a `UeArray` view. For each private field, `x()` (a clone), `set_x()` and `x_mut()`. For each component, a getter returning it from `reify_find_default_subobject`.
4. `as_ref()`, `from_obj(obj)` (validity, `IsA`, then the instance data; null data means the library was reloaded or unloaded), `get_default()` (the default object).
5. Hidden registration functions: `__rusteal_register_<name>` (find the parent, register the `RustTypeInfo` whose construct is `Box::into_raw(Box::new(Default::default()))`, call `reify_create_class`), `__rusteal_members_<name>` (properties, interfaces, components), `__rusteal_finalize_<name>` (`reify_finalize_class`, then the `default =` values on the default object), `__rusteal_after_defaults_<name>` (`reify_set_class_config` for `config`).
6. `inventory::submit!(ClassRegistration { krate, name, type_id, shape, depends_on, create, register, finalize, after_defaults })`, `krate` being the crate that declared it (`env!("CARGO_CRATE_NAME")`) and `depends_on` the parent's and the components' `RUST_TYPE_ID`s.

### `#[uclass_impl]` (`uclass_impl.rs`)

It takes no arguments and relies on the names `#[uclass]` generated, so it sits in the same module as the struct.

`#[ufunction(...)]` accepts `BlueprintCallable` (the default), `BlueprintPure`, `Override`, `BlueprintImplementableEvent`, `BlueprintNativeEvent` (at most one of the last four), `Exec`, `CallInEditor`, `name`, `category` and `meta(...)`; anything else is an error. The doc comment becomes the `ToolTip`, and the metadata goes through `reify_set_function_metadata`. A method with no receiver is static (`FUNC_Static`), and its thunk calls `Type::method(args)`:

| Kind | Flags |
|---|---|
| Callable | `FUNC_Native \| FUNC_BlueprintCallable \| FUNC_Public` |
| Pure | the same, `\| FUNC_BlueprintPure` |
| Override | `FUNC_Native \| FUNC_BlueprintEvent \| FUNC_Public` |
| ImplementableEvent | `FUNC_Event \| FUNC_BlueprintEvent \| FUNC_BlueprintCallable \| FUNC_Public`, no `FUNC_Native` |
| NativeEvent | `FUNC_Native \| FUNC_Event \| FUNC_BlueprintEvent \| FUNC_BlueprintCallable \| FUNC_Public` |
| `Exec` | adds `FUNC_Exec` |

Parameters and returns are what `prop_type::map_fn_type` maps: a property's types, with `Vec<T>`, `HashMap<K, V>` and `HashSet<T>` instead of the engine containers, `Option<ScriptInterface<dyn I>>`, and `UStructRef<T>` for a struct the engine lends; a `&mut T` parameter is an out parameter (`CPF_OutParm`). Scalars, objects, classes and enums are read from and written to the parameter buffer at their offsets; the other kinds cross as properties of the buffer, through `DynamicValue` and the property handles the thunk finds once (`crossed_as_property`). An override's parameters come from the parent's function, duplicated with `FField::Duplicate`, not from Rust. A `NativeEvent`'s body is renamed `<name>_implementation`, registered as the native function, and a dispatcher with the method's name calls the event through `ProcessEvent`. A return type of `RustealResult<T>` (or `Result<T, RustealError>`) makes the function fallible: Unreal sees `T`, and an `Err` is raised as a script error (see [Errors](errors.md)). An `ImplementableEvent` has an empty body, which the macro replaces with a `DynamicCall` to the Blueprint's implementation.

`#[class_defaults]` (one per block, `&mut self`) runs on the class default object after `finalize`; a `RustealResult<()>` it returns is logged on error. `#[udelegate]` (an empty `&self` method; `name`, and `single` or `sparse`, optional) declares a multicast delegate, a single-cast one or a sparse one: its body becomes a broadcast (an execute for a single-cast one), and the macro adds `<name>_add(closure)` and `<name>_add_ufunction(target, "Function")` (`_bind` and `_bind_ufunction` for a single-cast one). Its parameters take what a function's do, and cross the same way.

The expansion registers each function with a closure (`register_function`, whose returned index is the callback id), then `reify_add_function` and its parameters with `CPF_Parm` (and a `ReturnValue` with `CPF_ReturnParm`); events get callback id 0. It submits a `ClassFunctionRegistration { krate, type_id, shape, register_functions, class_defaults }`.

### `#[ustruct]` and `#[uenum]`

`#[ustruct]` requires every field to be a `#[uproperty]` (no `default`; a nested struct is `OwnedStruct<T>`). The Rust struct becomes a unit marker type; the fields are reached through a `<Name>Ext` trait implemented for `UStructRef<Name>` (memory Unreal owns) and `OwnedStruct<Name>` (a buffer Rust owns). `#[uenum]` takes unit variants with discriminants in 0..=254 and makes the enum `#[repr(u8)]`; see [Types declared in Rust](../internals/rust_types.md).

### The shape (`shape.rs`)

Each macro hashes its declarations: the sorted list of the attribute tokens with `ident: Type` for fields, or the signature for methods, joined and FNV-1a hashed. Bodies, doc comments, private fields and order do not count. `reify_registry::class_shape` combines a class's shape with its impl blocks', and the plugin compares it on [hot reload](hot_reload.md) to keep or replace the class.

## The registry (`rusteal-core/src/reify_registry.rs`)

| Store | What it holds |
|---|---|
| `TYPE_REGISTRY` | `type_id` → the module that registered it and its `RustTypeInfo` (construct and drop functions). A type id already registered by another module is refused with an error. |
| `FUNC_REGISTRY` | A `Vec` of `FunctionEntry { callback, type_id, module }` slots; the callback id is the index, and a cleared module's slots are emptied, never reused while another module holds callbacks. |
| `INSTANCE_DATA` | The object's address → one `InstanceEntry { data, type_id }` per reified class in its chain (a Blueprint child of a Rust class of a Rust class has two). |

- `register_module(module)` runs the registration of the types whose crate the module owns (every crate, in a dynamic library); `manifest(owned)` lists them as `kind:crate:name` entries (`impl:crate:<type id>` for impl blocks), which a static build checks (see [Static linking](boundary.md#static-linking)).
- `construct_instance(obj, type_id)` builds the data and stores it, replacing an existing entry of that type; `drop_instance(obj)` drops the entries of the address whose types belong to the current module, which is what C++ calls once per library of the chain; `clear_module` drops a module's types, functions and data when it shuts down.
- **A call from the engine** reaches `invoke_function(id, obj, params)`: the entry is cloned, the instance data found by type, and the generated closure runs. It caches the parameters' offsets (reflection, once), reads each with `native_mem_read` (or as a `UStructRef`/`OutRef` into the buffer), zero-initializes the return slot, builds a temporary `Self { __obj, __rust_data }`, calls the method, raises an `Err`, and writes the return value back (`reflection_copy_struct` for a struct).

## The C++ side

### `URustealReifiedClass`

A `UBlueprintGeneratedClass` subclass, the class behind every `#[uclass]`: `RustTypeId`, its `Library`, its `Shape`, its first native ancestor (`NativeSuperClass`) and its component definitions. Being a Blueprint-generated class is what lets the engine create and destroy it at runtime, and Blueprints derive from it; it is also why editor Python sees Rust classes by their Unreal names.

**`RustealClassConstructor`**, its `ClassConstructor`:

1. finds every reified class in the object's class chain, root first (`ReifiedChain`), which is how a Blueprint child of a Rust class is constructed;
2. calls the native parent's constructor;
3. creates each component definition of the chain with `CreateDefaultSubobject`, sets its property, and makes it the root or attaches it (to a component of this pass, or an inherited one found by name);
4. calls `construct_rust_instance(obj, RustTypeId, bIsCDO)` for each class of the chain, each through its own library.

### `RustealReifyApiImpl.cpp`

- **`CreateClassImpl`**: in the library's package (`/Script/Rusteal` or `/Script/<Plugin>`, created rooted with `PKG_CompiledIn`), a `NewObject<URustealReifiedClass>(RF_Public | RF_Standalone)` with the parent as super struct, its `ClassWithin`, the inherited class flags without the config ones plus `CLASS_CompiledFromBlueprint`, and `RustealClassConstructor`. In the editor, a stub `UBlueprint` named `RS_<Name>` is its `ClassGeneratedBy`, up to date and rooted, so the editor's Blueprint machinery accepts the class. The class is rooted. An existing class is kept or retired by its shape (see [Hot reload](hot_reload.md)).
- **`AddPropertyImpl`**: an existing property of the same name is reused (hot reload); otherwise one is created by type and added with `AddCppProperty`. An enum property is an `FEnumProperty` over an `FByteProperty`.
- **`AddFunctionImpl`**: an existing `URustealReifiedFunction` keeps its object and gets the new callback id and library; a new one is linked into the class's `Children`. A native function gets `execCallRustFunction` as its native thunk; an override copies its parameters from the parent's function of the same name (`CopyParamsFromParentFunction`).
- **`FinalizeClassImpl`**: binds and links the functions and the class, assembles the reference token stream, notifies the registration, and creates the default object, which runs the class constructor and therefore the default object's Rust data. An actor class with its own `ReceiveTick` gets `bCanEverTick` and `bStartWithTickEnabled`.
- **Components** (`AddDefaultSubobjectImpl`) are stored as `FRustealComponentDef`s with an object property (`EditConst`, `BlueprintReadOnly`, `InstancedReference`...). **Interfaces** (`AddInterfaceImpl`) are loaded by path, after engine init when needed, and added as `FImplementedInterface(I, 0, true)`, as for a Blueprint. **Config** (`SetClassConfigImpl`) sets the config name and flags and loads the default object's config.

### `URustealReifiedFunction::execCallRustFunction`

1. Finds the reified function: `Stack.CurrentNativeFunction`, `Stack.Node`, the super function chain, or by name in the object's reified classes.
2. Gets the parameters: from `ProcessEvent`, they are `Stack.Locals`; from the Blueprint VM, they are stepped into a zeroed `FMemory_Alloca` buffer, with the out parameters' addresses recorded.
3. Calls `invoke_rust_function(CallbackId, Self, Params)` with the library current.
4. A pending script error becomes a Blueprint runtime error, on the VM path.
5. Copies out parameters and the return value back, and destroys the VM path's parameters.

### Lifetime

The engine owns the object; Rust owns its data. `FRustealDeleteListener::NotifyUObjectDeleted` calls `drop_rust_instance` once per library of the object's reified chain when the garbage collector deletes it. Private fields live on the Rust heap and are not reflected, so the garbage collector does not see objects they reference: a private `UObjectRef` does not keep its object alive (a `#[uproperty]` or a `Pinned` does).

## Structs, delegates and subsystems

- **Structs.** `CreateStructImpl` makes a `UScriptStruct` (`BlueprintType`), reused on reload; `FinalizeStructImpl` finalizes nested structs first, then binds, links and prepares the struct ops. `UStructRef<T>` borrows memory Unreal owns (a parameter buffer); `OwnedStruct<T>` owns a buffer, initialized and destroyed through reflection.
- **Delegates.** `#[udelegate]` adds a signature function `<Name>__DelegateSignature` and an `FMulticastInlineDelegateProperty` (`BlueprintAssignable`); `single` makes it an `FDelegateProperty`, and `sparse` an `FMulticastSparseDelegateProperty` with a `USparseDelegateFunction` signature, whose offset `FinalizeClassImpl` registers with `FSparseDelegateStorage`. A broadcast is a `DynamicCall::for_delegate`. Binding a Rust closure stores it in `delegate_registry` under an id and creates a `URustealDelegateProxy` (outer: the owner) whose `ProcessEvent` forwards the broadcast to `invoke_delegate_callback`. A `DelegateBinding` unbinds when dropped, unless `detach()`ed; a closure re-entered while it runs is skipped.
- **Subsystems.** The engine only creates subsystems of native classes, so `RustealSubsystems.h` declares abstract native parents that turn the virtuals into events Rust overrides: `URustealGameInstanceSubsystem`, `URustealWorldSubsystem` (a tickable world subsystem with `ReceivePostInitialize`, `ReceiveWorldBeginPlay`, `ReceiveTick`, `bWantsTick`, `bCreateInEditorWorlds`), `URustealEngineSubsystem`, `URustealLocalPlayerSubsystem` (`ReceivePlayerControllerChanged`), and, in the editor module, `URustealEditorSubsystem`. Each forwards `ShouldCreateSubsystem`, `Initialize` and `Deinitialize` to `ReceiveShouldCreateSubsystem`, `ReceiveInitialize` and `ReceiveDeinitialize`.

## Object references

| Type | What it is | Contract |
|---|---|---|
| `UObjectRef<T>` | `#[repr(transparent)]` over `UObjectHandle` | `Copy`, no ownership, no GC root; `is_valid()` is `::IsValid`; `checked()` before use; `cast()` checks `IsA`, `upcast` is checked at compile time. The `Deref` chain casts pointers, so the layout must stay transparent. |
| `TWeakObjectPtr<T>` | An `FWeakObjectPtr`'s bytes | `get()` is `None` once the object is gone. |
| `SoftObjectRef<T>` | A path | `get()` finds a loaded object; `load_synchronous()` loads it. |
| `SubclassOf<T>` | `#[repr(transparent)]` over `UClassHandle` | `base()` is `T::static_class()`. |
| `Pinned<T>` | A GC root (`AddToRoot`) | Not reference counted: dropping one of two `Pinned` of the same object unroots it. Pins are released when their library unloads. |
