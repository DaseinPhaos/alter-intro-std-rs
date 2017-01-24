% std::any

> ref: https://doc.rust-lang.org/std/any/

TODO:

- [ ] Fix backward reference: any reference to `std` components not mentioned beforehead should be marked and explained.


This module defines the `Any` trait and the `TypeId` struct. Together they form minimal support for runtime reflection.

#TypeId

A `TypeId` represents a globally unique identifier for a type. It's an opacue object, but does allow basic operations such as cloning, comparison, and formatting.

currently it is only available for `'static` types.

The definition looks like (with some attributes and comments removed):

```ignore
#[derive(Clone, PartialEq, Eq, Debug, Hash)]
pub struct TypeId {
    t: u64,
}

impl TypeId {
    pub fn of<T: ?Sized + 'static>() -> TypeId {
        TypeId {
            t: unsafe { std::intrinsics::type_id::<T>() },
        }
    }
}
```

Note the static method `of`. It should be used like this:

```rust
use std::any::TypeId;

fn is_string<T: ?Sized + 'static>(_a: &T) -> bool {
    TypeId::of::<String>() == TypeId::of::<T>()
}

fn main() {
    assert_eq!(is_string(&0), false);
    assert!(is_string(&"foo".to_owned()));
}
```

#Any

The `Any` trait, however, being implemented for any `'static` types as follows(comments and attributes omitted):

```ignore
pub trait Any: 'static {
    fn get_type_id(&self) -> TypeId;
}

impl<T: 'static + ?Sized> Any for T {
    fn get_type_id(&self) -> TypeId {
        TypeId::of::<T>()
    }
}
```

is more powerful. When used statically it's nothing more than a object-oriented wrapper for `TypeId`'s `of` method, not to mention that the method is currently marked as `unstable`([#27745](https://github.com/rust-lang/rust/issues/27745)).


Its true strength comes when used as a trait object. It has an `is<T>` method whcich would return true if the underlying object is of type `T`, and two `downcast`ing methods returning corresponding `Some`.

The general use case can be demonstrated as follow:

```rust
use std::any::Any;

fn is_string(a: &Any) -> bool {
    a.is::<String>()   
}

fn format_if_str_literal(a: &Any) -> String {
    if let Some(s) = a.downcast_ref::<&'static str>() {
        format!("It is a string literal with a length of ({}): {}", s.len(), s)
    } else {
        format!("Not a string literal...")
    }
}

fn main() {
    assert_eq!(format_if_str_literal(&1.1), "Not a string literal...");
    assert_eq!(format_if_str_literal(&"cookie monster"), "It is a string literal with a length of (14): cookie monster");

    assert!(is_string(&"Foo".to_string()));
    assert!(!is_string(&"Foo"));
}
```

The actual implementation looks like the following:

```ignore
impl Any {
    pub fn is <T: Any>(&self) -> bool {
        let t = TypeId::of::<T>();

        let boxed = self.get_type_id();

        t == boxed
    }

    pub fn downcast_ref<T: AnyM(&self) -> Option<&T> {
        if self.is::<T>() {
            unsafe {
                Some(&*(self as *const Any as *const T))
            }
        } else {
            None
        }
    }

    pub fn downcast_mut<T: Any>(&mut self) -> Option<&mut T> {
        if self.is::<T>() {
            unsafe {
                Some(&mut *(self as *mut Any as *mut T))
            }
        } else {
            None
        }
    }
}

impl Any+Send {
    // Same as above..
}
```
 Note the direct use of `impl` on trait objects. This is valid code, but The Book has failed to mention it(not in my memory).

Also note that for convenience, these methods are also implemented for trait object `Any+Send` (with the implementation simply forwarding to `Any`).

Another thing to notice is that `std::fmt::Debug` is implemented for `Any`, and of course for `Any+Send`.