% std::clone

> https://doc.rust-lang.org/std/clone/

This module defines the `Clone` trait:

```ignore
pub trait Clone: Sized {
    fn clone(&self) -> Self;

    #[inline(always)]
    fn clone_from(&mut self, source: &Self) {
        *self = source.clone();
    }
}
```

The trait provides the ability to explicitly duplicate an object.

While the `Copy` trait allows types to be implicitly copied when assigning or passing by value; the trait only suits those types which are "cheap"(not large, don't require heap allocation) and "safe"(do not contain owned pointers or implement `Drop`) to copy.

For other types, copies must be made explicitly, by convention implementing the `std::clone::Clone` trait and calling its `clone` method.

The trait can be easily implemented by using `#[derive(Clone)]`, when all fields of the type are `Clone`. The default implementation of `clone` calls `clone` on each field.

Also note that, types that are `Copy` should have a trivial implementation of `Clone`: if `T: Copy`, `x: T` and `y: &T`, then `let x = y.clone();` should be equivalent to `let x = *y;`. Manual implementations should be careful to uphold this invariant; however, like most invariants that should be manually uphold by Safe Rust, when writing `unsafe` code, it shouldn't be relied on.

