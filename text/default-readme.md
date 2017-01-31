% std::default

> https://doc.rust-lang.org/std/default/

This module defines the `Default` trait:

```ignore
pub trait Default: Sized {
    fn default() -> Self;
}
```

The trait is used to give the implementing type a default value to fallback to.

Sometimes, like when initializing some `Vec<T>`, all we really care is that the vector being created, not the actual values the elements would get initialized to. Some other cases we want just want the type we defined, say it represents some configurations, to have a nice default value. Under these cases, we can make use of this trait:

```rust
#[derive(Default)]
struct SomeConfigs {
    foo: i32,
    bar: f32,
}

fn main() {
    let configs: SomeConfigs = Default::default();
}
```

Note that this trait can be used with #[derive] if all of the type's fields implement Default. When derived, it will use the default value for each field's type.