% std::marker

> https://doc.rust-lang.org/std/marker/

This module contains some "marker" traits that represent some basic properties of types.

Primarily, the module defines:

```ignore
pub trait Copy: Clone { }
```

This trait represents a type whose values can be duplicated by simply "copying bits". Types implementing this trait changes the default "moving" semantic of variable binding into by-value copying.

Note that this trait derives from `Clone`, which totally makes sense: a type that can be trivially duplicated of course can also be trivially cloned. It also means that for a `T: Copy` and some `x: T`, `let y = x;` is equivalent to `let y = x.clone();`.

Like `Clone`, this trait can be derived if all of its components implement `Copy`.

---

```ignore
pub unsafe trait Send { }
```

This trait represents a type that can be safely transferred across thread boundaries.

For most of the times, you don't have to deal with implementing thi trait explicitly, (in fact, the trait is marked as `unsafe`, thus it is essentially `unsafe` for a user to implement this trait by hand) the compiler would automatically generates these traits for your types when it determines appropriate.

The rule the compiler uses is quite simple: automatcial deriving. That is, if a type is composed entirely of `Send` types, then it is `Send`.

Most types are `Send`, exceptions include:

- raw pointers
- `Rc`

For further explanations of why this is the case, refer to [the Rustonomicon](https://doc.rust-lang.org/nomicon/send-and-sync.html).

---

```ignore
pub unsafe trait Sync { }
```

This trait represents a type whose immutable references can be safely shared between threads. In other words, if `T: Sync` then `&T: Send`.

From this definition, a somewhat surprising consequence is that, if `T: Sync` then `&mut T: Sync`.

Like `Send`, this trait is automatically derived by the compiler. Like `Send`, it is also `unsafe` to implement by hand.

Essentially, types with interior mutability are determined to be not `Sync` by the compiler, primitives including:

- raw pointers,
- `UnsafeCell`, (thus `Cell` and `RefCell`),
- `Rc`

For further explanations, refer to [the Rustonomicon](https://doc.rust-lang.org/nomicon/send-and-sync.html).

---

```ignore
pub trait Sized { }
```

This trait refers to a type with a constant size known at compile time. We should be familiar with it by now, as all type parameters have an implicit bound of `Sized`, which can only be removed by a special bound `?Sized`. Note one exception of this implicit bound, the `Self` type of a trait. This prevents the a trait from directly being used to form a trait object.

---

```ignore
pub struct PhantomData<T> where T: ?Sized;
```

This type is a zero-sized type marker, being used to relate one type *logically* with another.

Typically usages include:

- associating lifetime with raw-pointer-wrappers:

```rust
# use std::marker::PhantomData;

struct Slice<'a, T> {
    start: *const T,
    end: *const T,
    _phantom: PhantomData<&'a T>,
}
```

A `Slice` references some data that should only be valid for `'a`. Yet internally the `Slice` uses raw pointers for efficiency, which are not related to any lifetimes. To actually relate the lifetime to `Slice`, a `PhantomData` is used.

- associating types:

```rust
extern crate foreign_lib;
use std::marker::PhantomData;
use std::mem;

struct ExternalResource<R> {
    handle: &mut (),
    pahntom_type: PhantomData<R>,
}

impl<R: ResType> ExternalResource<R> {
    fn new() -> ExternalResource<R> {
        let res_type_size = mem::size_of::<R>();
        ExternalResource {
            handle: foreign_lib::acquire_resource(),
            phantom_type: PhantoData,
        }
    }
}
```

- associating ownership and drop checking:
 Adding a field of type `PhantomData<T>` indicates that your type *owns* some data of type `T`. This in turn means that, when your type is dropped, it may also drop some instance of type `T`. This in turn means that, the owning type should soundly implement drop. If the struct doesn't in fact *own* any `T`s, it is better off to use a reference type (or raw pointer types when no lifetime applies).

 