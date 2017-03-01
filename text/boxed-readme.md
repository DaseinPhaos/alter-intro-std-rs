% std::boxed

> https://doc.rust-lang.org/std/boxed/

This module introduces a pointer type for heap allocation, `Box<T>`. It owns the allocated `T` instance, and drop that content when itself goes out of scope.

The definition is as follows:

```ignore
pub struct Box<T: ?Sized>(Unique<T>);

// ...

impl<T> Box<T> {
    pub fn new(x: T) -> Box<T> {
        // ...
    }
}

impl<T: ?Sized> Box<T> {
    pub unsafe fn from_raw(raw: *mut T) -> Self {
        mem::transmute(raw)
    }

    pub fn into_raw(b: Box<T>) -> *mut T {
        unsafe { mem::transmute(b) }
    }
}
```

The `new` method allocates memory on the heap and places the `x` into it, then return the boxed `Box<T>`.

Then `from_raw` method constructs a box from raw pointer `raw`. After calling, the pointee is owned by the resulting `Box`. This is considered as using raw pointer, thus the method is marked as `unsafe`. The only valid pointer to pass to this function is the one take from another `Box` via `Box::into_raw` function.

The `into_raw` method, on the other hand, consumes the calling `Box`, and returns a raw pointer pointing to the calling data. The caller should be responsible for that memory after invoking this function. Note that this is an associated function. As such, the calling syntax should be `Box::into_raw(b)` instead of `b.into_raw()`. This is so that there is no conflict with methods of the inner type.

Additionally, for `T: Any + 'static` (` + Send`), the struct `Box<T>` also implements `downcast`.

As for the traits, basically if `T` implements any `Hash`, `Debug`, `Display`, `Default`, `PartialEq`, `Ord`, `Eq`, `PartialOrd`, `Iterator`, `FnOnce`, `Clone`, `Seek`, `BufRead`; then the corresponding `Box<T>` would also implement those traits, with trivial behaviors as we'd expect.

Additionally, `Box<T>` implements `Borrow<T>`, `BorrowMut<T>`, `Deref<Target=T>`, `Pointer`, `Boxed`, `AsMut<T>`, `From<T>`.

TODO:

- [x] introduce trait implementations.
- [ ] introduce unstable features in this module.