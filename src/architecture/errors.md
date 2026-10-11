# Errors

An error in Rusteal says what failed, on which object, why Unreal refused, and where in the user's Rust code, and it reaches the places a game developer looks: the return value, the Output Log, the Message Log and, under a Blueprint call, a Blueprint runtime error. How it is used is in the game developer's handbook, [Errors](https://viniciusmorgado.github.io/game.rusteal.github.io/engine_api/errors.html); this chapter is how it is built.

## `RustealError` (`rusteal-core/src/error.rs`)

Modelled on `std::io::Error`: a struct, matched by `kind()`, not an enum of variants. It is one `Box<Inner>`, so `RustealResult<()>` is a pointer wide (a test checks it), and it is `Send + Sync` and implements `std::error::Error`, so it works with `?`, `anyhow` and user error types.

`Inner` holds:

| Field | What it is |
|---|---|
| `kind` | `ErrorKind`, `#[non_exhaustive]`: `ObjectDestroyed`, `InvalidCast`, `PropertyNotFound`, `FunctionNotFound`, `TypeMismatch`, `NullArgument`, `IndexOutOfRange`, `InvalidOperation`, `BufferTooSmall`, `Internal`, `Other`. |
| `message` | The name involved, or the user's message. |
| `engine` | The engine's reason, from `take_error_message`. |
| `subject` | The object's and class's names, read when the error is made. |
| `object` | A weak handle to the object, for the Message Log link. |
| `location` | The caller's `Location`: every fallible core function, macro accessor and generated binding is `#[track_caller]`. |
| `frames` | The contexts, each with its own location. |
| `source` | The cause, a `Box<dyn Error + Send + Sync>`. |

- **Constructors** (all `#[track_caller]`): `new(kind)`, `msg(..)` (`Other`), `other(error)` (a `RustealError` stays itself), and one per kind (`object_destroyed()`, `invalid_cast_to(obj, target)`, `property_not_found(name)`...).
- **Builders**: `context(..)`, `on(&object)`/`on_handle(h)`/`on_class(..)` (the first subject wins), `with_source`.
- **Context** on results: `ErrorContext` for `Result<T, E: Error>` and `Option<T>` (`.context(..)`, `.with_context(..)`); `bail!` and `ensure!`; `From` for the common std errors.
- **Formatting**: `{e}` is one line (`context: kind: message (subject): Unreal: reason`); `{e:#}` adds the causes and the location; `{e:?}` is the full report, with `at file:line:column` and a `while ...` line per context.

## The engine's reason

The plugin's implementations fail with `RustealFail(code, message)` (`Private/RustealErrors.cpp`), which keeps the message in a thread-local with the frame number; `RustealFailType` writes the type mismatches ("Health (FloatProperty of BP_Enemy_C) is not a BoolProperty"). On the Rust side, `check_ffi(code)` makes the error and, for every code but `ObjectDestroyed`, takes the message with `core.take_error_message`, which only answers for the same code in the same frame, so a stale reason never attaches to a later error. `check_ffi_ctx(code, ctx)` uses the context as the message for the "not found" and invalid operation codes, as a context frame otherwise. `ffi_infallible` is for calls that cannot fail: a `debug_assert` only.

## Reporting

`RustealError::log(level)`, the macros' generated code (`__raise`, `__log_in`) and the panic guard send a `RustealErrorReport` through `logging.report`: level, whether it is a script error, the one-line summary, the full report, the weak object, the absolute source file (relative locations are joined to the crate's directory, recorded by `entry!()` with `set_source_root(env!("CARGO_MANIFEST_DIR"))`), line and column.

`RustealReport` on the C++ side:

1. appends the Blueprint script call stack to the details;
2. writes `[Rusteal] ...` to `LogRusteal` at the matching verbosity;
3. for a script error, keeps the summary as the thread's pending script error;
4. on the game thread only, adds an entry to the **Rusteal** Message Log: the summary, a link to the object (an object of Play In Editor falls back to the level's actor of the same path once Play stops, through `UWorld::RemovePIEPrefix`), and a `file:line` link that calls the editor module's source opener; and for an error, an editor notification, at most every five seconds.

## Fallible functions and Blueprint runtime errors

A `#[ufunction]` returning `RustealResult<T>` is a function returning `T` to Unreal. When its body returns `Err`, the generated closure calls `__raise("Struct::Function")`: the error is reported as a script error with that prefix, and the function returns the default value. `execCallRustFunction` then takes the pending script error and, when the call came from the Blueprint VM (not from `ProcessEvent`), raises it with `FBlueprintCoreDelegates::ThrowScriptException` as a `UserRaisedError`: in Play In Editor, a "Blueprint Runtime Error" in the PIE Message Log with a link to the node. A `NonFatalError` would not show there, which is why the error type is `UserRaisedError`.

## Panics

`ffi_guard::install_panic_hook` chains a hook that records the panic's location in a thread-local, before the previous hook runs. `ffi_boundary` catches the panic and reports it as a script error, "Rust panic: message", at that location, then returns the callback's default. A panic under a Blueprint call therefore shows like any fallible function's error.
