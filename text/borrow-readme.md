% std::borrow

> https://doc.rust-lang.org/std/borrow/

In Rust, borrowing generally comes in the flavor of reference: type `T` can be borrowed as `& T` or `&mut T`. However, the concept of borrowing can extend beyond the form of simple reference. Custom types may provide additional kind of borrowing. For example, `Vec<T>` introduces borrowed slices `&[T]` and `&mut [T]`.

To capture the general concept of borrowing, this module defines two traits.

First comes the `Borrow` trait:

```ignore
pub trait Borrow<Borrowed: ?Sized> {
    fn borrow(&self) -> &Borrowed;
}
```

If type `T` implements `Borrow<Borrowed>`, then `& T` can be borrowed immutably as `& Borrowed`. Such design allows a single type to be borrowed as multiple different types. For example, `Vec<T>` implements both `Borrow<Vec<T>>` and `Borrow<[T]>`, thus a `& Vec<T>` can be borrowed both as a `& Vec<T>` or as a `& [T]`.

The required method `fn borrow(&self) -> &Borrowed` performs the actual borrowing.

To demonstrate its usage, check out how the module implements it for generic type `T`:

```ignore
// Of course `& T` can be borrowed as `& T`.
impl <T: ?Sized> Borrow<T> for T {
    fn borrow(&self) -> &T { self }
}

// `&&'a T` can be borrowed as `& T`.
impl<'a, T:?Sized> Borrow<T> for &'a T {
    fn borrow(&self) -> &T { &**self }
}

// `&&'a mut T` can be borrowed as `& T`.
impl<'a, T: ?Sized> Borrow<T> for &'a mut T {
    fn borrow(&self) -> &T { &**self }
}
```

Another example from `Box<T>`:

```ignore
// `& Box<T>` can be borrowed as `& T`
impl<T: ?Sized> Borrow<T> for Box<T> {
    fn borrow(&self) -> &T { &**self }
}
```

Then there is `BorrowMut`:

```ignore
pub trait BorrowMut<Borrowed: ?Sized> : Borrow<Borrowed> {
    fn borrow_mut(&mut self) -> &mut Borrowed;
}
```

, which represents a mutable borrow. That is, suppose we have `T: BorrowMut<U>`, then `&mut T` can be borrowed as `&mut U`. Note that `BorrowMut` inherits from `Borrow`. This matches the semantic behavior of borrowing: we *can* borrow a `& T` out from a `&mut T`, not the opposite.

As we'd expect, the module implements this trait for `T` and `'a mut T`:

```ignore
// `&mut T` can be borrowed as `&mut T`
impl<T: ?Sized> BorrowMut<T> for T {
    fn borrow_mut<&mut self> -> &mut T { self }
}

// `&mut &'a mut T` can be borrowed as `&mut T`
impl<T: ?Sized> BorrowMut<T> for &'a mut T {
    fn borrow_mut<&mut self> -> &mut T { &mut **self }
}
```

Another example from `Box<T>`:

```ignore
// `&mut Box<T>` can be borrowed as `&mut T`
impl<T: ?Sized> BorrowMut<T> for Box<T> {
    fn borrow(&mut self) -> &mut T { &mut **self }
}
```

One thing worth noting is that, if both `Self` and `Borrowed` implement `Hash`, `Eq` and/or `Ord`, they must produce the same result. This should be true for both `Borrow` and `BorrowMut`.

Another thing worth mention is the difference between `Borrow` and `AsRef`. Check out [The Book](https://doc.rust-lang.org/book/borrow-and-asref.html).


There is yet another trait in this module, the `ToOwned` trait:

```ignore
pub trait ToOwned {
    type Owned: Borrow<Self>;
    fn to_owned(&self) -> Self::Owned;
}
```

While `T:Borrow<U>` allows `& T` to be borrowed as `& U`, `U:ToOwned<Owned=T>` allows `&U`, being borrowed from some `&T`, to return a new instance of `T`. It can be perceived as a generalization of `Clone`. Say for `String`, the `Clone` trait it implements allows us to write:

```rust
# fn main() {
let some_string = String::from("Oh Captain");
let cloned_string = some_string.clone();
# assert_eq!(some_string, cloned_string);
# }
```

However, say for some reason we'd like the `cloned_string` to contain only the first word `"Oh"`. Our immediate reflection would be like, ok, let's just make a string slice and see what we can do with that:

```rust
# fn main() {
let some_string = String::from("Oh Captain");
let oh = &some_string[..2];
let cloned_string = oh.clone();
# assert_eq!(cloned_string, "Oh");
# }
```

The code compiles, except that the result is not what we wanted: `cloned_string` now has type `&str` rather than the `String` we'd hope for. This is where the `ToOwned` trait comes to rescue.

As `str` implements `ToOwned<String>`, one can write:

```rust
# fn main() {
let some_string = String::from("Oh Captain");
let oh = &some_string[..2];
let cloned_string = oh.to_owned();
// Now we are happy.
# assert_eq!(cloned_string, "Oh");
# }
```

Now we are happy. To conclude, `ToOwned` trait extends the concept of cloning to the generalized borrowing senario. If `T: ToOwned<Owned=U>`, then a `& T`, being borrowed from some `& U`, can make a clone of what it's borrowing by calling `to_owned` on itself.[^silly]

[^silly]: The example here is pretty silly though, as one can simply reuse `String::from(oh)` instead.

Note that the module implements `ToOwned<Owned=T>` for `T: Clone`. The implementation is trivial, as we'd expect:

```ignore
impl<T> ToOwned for T where T: Clone {
    type Owned = T;
    fn to_owned(&self) -> T {
        self.clone()
    }
}
```

Finally, the module defines `Cow`, providing clone-on-write functionality:

```ignore
pub enum Cow<'a, B: ?Sized + 'a + ToOwned> {
    Borrowed(&'a B),
    Owned(B as ToOwned)::Owned),
}

// ...

impl<'a, B: ?Sized> Cow<'a, B> where B: ToOwned {
    pub fn to_mut(&mut self) -> &mut <B as ToOwned>::Owned {
        match *self {
            Borrowed(borrowed) => {
                *self = Owned(borrowed.to_owned());
                match *self {
                    Borrow(..) => unreachable!(),
                    Owned(ref mut owned) => owned,
                }
            }
            Owned(ref mut owned) => owned,
        }
    }

    pub fn into_owned(self) -> <B as ToOwned>::Owned {
        match self {
            Borrowed(borrowed) => borrowed.to_owned(),
            Owned(owned) => owned,
        }
    }
}

```

A `Cow<'a, B>` might initially just hold some reference `&'a B` in its `Borrowed` variant like this:

```ignore
use std::borrow::Cow;
let numbers = vec![1, 2, 3, 4, 5];
let cow = Cow::Borrowed(&numbers[1..4]);
println!("{}", cow);
```

. As such, it's just pointing into something owned by others. However, when mutability is required, it would by then lazily perform `B: ToOwned`'s `to_owned` method, changing its content to the `Owned(..)` variant.

When `to_mut` method is invoked, the lazy clone(actually `to_owned`) would be performed(if not already), and a mutable reference to the newly cloned data would be returned. When `into_owned` is invoked, the invoking `Cow` would be consumed, returning the newly cloned(if not already) data by value. Continueing the example above, say if we want to change the content of the newly created `cow`:

```rust
# fn main() {
#    use std::borrow::Cow;
#    let numbers = vec![1, 2, 3, 4, 5];
#    let cow = Cow::Borrows(&numbers[1..4]);
let mut cow = cow.into_owned();
cow[1] = 6;
assert_eq!(vec![2, 6, 4], cow);
# }
```

Several `std`-traits are implemented for `Cow<'a, B>`, such as `Hash`, `Debug`, `Display`, `Clone`, `Eq`, `Ord` and so on. Among them, the `Deref` trait is worth mentioning:

```ignore
impl<'a, B: ?Sized> Deref for Cow<'a, B> where B: ToOwned {
    type Target = B;

    fn deref(&self) -> &B {
        match *self {
            Borrowed(borrowed) => borrowed,
            Owned(ref owned) => owned.borrow(),
        }
    }
}
```

 This allows `Cow<'a, B>` to be deref-coerced as `&B`.

The `std::convert::From` trait is also worth mentioning. The trait allows the following code:

```rust
# fn main() {
    use std::borrow::Cow;
    let cow = Cow::from("Foo");
#   println!("{}", cow);
# }
```

The actual implementation is something like the following:

```ignore
impl<'a> From<&'a str> for Cow<'a, str> {
    fn from(s: &'a str) -> Cow<'a, str> {
        // ...
    }
}

impl<'a> From<String> for Cow<'a, str> {
    fn from(s: String) -> Cow<'a, str> {
        // ...
    }
}

impl<'a, T> From<&'a [T]> for Cow<'a, [T]> where T: Clone {
    fn from(s: &'a [T]) -> Cow<'a, [T]> {
        // ...
    }
}

impl<'a, T> From<Vec<T>> for Cow<'a, [T]> where T: Clone {
    fn from(s: Vec[T]) -> Cow<'a, [T]> {
        // ...
    }
}
```