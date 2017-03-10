% std::panic

This module provides some functions to deal with unwinding panics.

Firstly, there is

```ignore
pub fn set_hook(hook: Box<Fn(&PanicInfo) + 'static + Sync + Send>)
```

After the panic yet before the panic runtime gets invoked, a hook function would get called. The default hook that we've all been familiar with prints a message to `stderr` and generates a backtrace (if requested). This function gives us the chance to customize the behavior of the hook. It registers a customized panic `hook`, replacing any existing hook.

`hook` should be a function object taking a `&PanicInfo` as input. `PanicInfo` contains information about a panic. To retrieve the info one can:

- call the associated method `payload(&self) -> &Any + Send` to get the payload associated with the panic. The underlying type of the payload is typically a `&'static str` (as is the case of `panic!("foo")`) or a `String` ( when the string needs to be formatted by runtime info), but it can be objects of other types.
- call the associated method `location(&self) -> Option<&Location>` to get the location where the panic originated, if available.

The counterpart of this function is

```ignore
pub fn take_hook() -> Box<Fn(&PanicInfo) + 'static + Sync + Send>
```

which unregisters and returns the current panic hook.

---

Then there is

```ignore
pub fn catch_unwind<F, R>(f: F) -> Result<R, Box<Any+Send+'static>>
    where F: UnwindSafe + FnOnce() -> R
```

The function takes a function object and invokes it. During its invocation, if the invocation panics, an `Err(cause)` would be returned with the `cause` being the object with which the panic was originally invoked. Otherwise, an `Ok` would be returned which contains the result of this call.

The function is primarily used when some Rust code needs to be called from another language. As it is currently UB to unwind into a foreign code, turning a possibly panic into a `Result` would allow a graceful handling of the error to become possible. It is **not** recommended to use this function as a "try/catch" mechanism within Rust, as panicking is only meant to be used by *non-recoverable* errors. In Rust, such a mechanism would be better off implemented via directly returning `Result`.

Note that the provided function object is required to adhere to the `UnwindSafe` marker trait. The trait represents a type with "unwinding safety", meaning that the implementing type cannot easily allow witnessing of a broken invariant through the `catch_unwind` boundary.[^1] Just like `Send` and `Sync`, this trait is automatically implemented by the compiler when it find appropriate. Basically speaking, types with logical mutability (such as `&mut T` and `&RefCell<T>`) would be marked as `!UnwindSafe`.

[^1]: The actual definition is pretty involved, see the [related RFC](https://github.com/rust-lang/rfcs/blob/master/text/1236-stabilize-catch-panic.md).

The counterpart of `catch_unwind` is

```ignore
pub fn resume_unwind(payload: Box<Any + Send>) -> !
```

which would trigger a panic with the provided payload without invoking the panic hook.

