% std::any

> ref: https://doc.rust-lang.org/std/any/

This module defines the `Any` trait and the `TypeId` struct. Together they form minimal support for runtime reflection.

#TypeId

A `TypeId` represents a globally unique identifier for a type. It's an opaque object, but does allow basic operations such as cloning, comparison, and formatting.

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

The `Any` trait,

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

is more powerful.

Its true strength is revealed when used as a trait object. It has an `is<T>` method which would return true if the underlying object is of type `T`, and two `downcast`ing methods returning corresponding `Some`:

```ignore
impl Any {
    pub fn is <T: Any>(&self) -> bool {
        let t = TypeId::of::<T>();

        let boxed = self.get_type_id();

        t == boxed
    }

    pub fn downcast_ref<T: Any>(&self) -> Option<&T> {
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

The general use case is demonstrated as follow:

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

Another thing to notice is that `std::fmt::Debug` is implemented for `Any`, and of course for `Any+Send`.