% std::convert

> https://doc.rust-lang.org/std/convert/

This module defines traits that provide a general way to talk about type conversions.

* `As`* traits are for conversions between references.
* `Into` trait is for by-value conversions.
* `From` trait is the most flexible one, useful for both value and ref conversions.
* The currently [unstable](https://github.com/rust-lang/rust/issues/33417) `Try*` traits behave like their non-prefixed siblings, but allow for the conversion fail.

# As*

These traits are for ref-to-ref conversions.

First there is `AsRef`:

```ignore
pub trait AsRef<T: ?Sized> {
    fn as_ref(&self) -> &T;
}
```

This is intended to be a cheap ref-to-ref conversion that **must not fail**.

Note that, its generic implementations includes:

```ignore
impl<'a, T, U> AsRef<U> for &'a T
    where T: AsRef<U> + ?Sized,
          U: ?Sized {
    fn as_ref(&self) -> &U {
        <T as AsRef<U>>::as_ref(*self)
    }
}

impl<'a, T, U> AsRef<U> for &'a mut T
    where T: AsRef<U> + >Sized,
          U: ?Sized {
    fn as_ref(&self) -> &U {
        <T as AsRef<U>>::as_ref(*self)
    }
}
```

This means that, `AsRef` would auto-dereference its inner type. For example, if `Foo: AsRef<Bar>`, then `foo.as_ref()` will work the same if `foo: & Foo` or `foo: &&mut Foo` etc.

With the knowledge above, also note the concrete implementations included:

```ignore
impl<T> AsRef<[T]> for [T] {
    fn as_ref(&self) -> &[T] {
        self
    }
}

impl<T> AsRef<str> for str {
    #[inline]
    fn as_ref(&self) -> &str {
        self
    }
}
```

This in effect guarantees that for any `foo: &[T]` (or `foo: &&mut [T}`), `foo.as_ref()` would produce an `&[T]` as we'd expect. The same goes for `str`.

Some other examples of its implementation include:

```ignore
impl<T: ?Sized> AsRef<T> for Box<T> {
    fn as_ref(&self) -> &T {
        &**self
    }
}
```

```ignore
impl AsRef<str> for String {
    #[inline]
    fn as_ref(&self) -> &str {
        self
    }
}

impl AsRef<[u8]> for String {
    #[inline]
    fn as_ref(&self) -> &[u8] {
        self.as_bytes()
    }
}
```

Next there is `AsMut`, for cheap `& mut`-to-`& mut` conversion, which also must not fail:

```ignore
pub trait AsMut<T: ?Sized> {
    fn as_mut(&mut self) -> &mut T;
}
```

Similar to `AsRef`, it also performs auto dereferencing:

```ignore
impl<'a, T, U> AsMut<U> for 'a T
where T: AsMut<U> + ?Sized,
      U: ?Sized {
    fn as_mut(&mut self) -> &mut U {
        (*self).as_mut()
    }
}
```

and also is implemented for `[T]`:

```ignore
impl<T: ?Sized> AsMut<[T]> for [T] {
    fn as_mut(&mut self) -> &mut [T] {
        self
    }
}
```

# From and Into

Then there are traits for by-value conversions:

```ignore
pub trait From<T>: Sized {
    fn from(T) -> Self;
}
```

The `From<T>` trait constructs a `Self` from `T`.

```ignore
pub trait Into<T>: Sized {
    fn into(self) -> T;
}
```

The `Into<T>` trait constructs a `T` from a `self: Self`.

Note that the module also defines the following generic implementations:

```ignore
impl<T, U> Into<U> for T
    where U: From<T> {
    fn into(self) -> U {
        U::from(self)
    }
}

impl<T> From<T> for T {
    fn from(t: T) -> T { t }
}
```

That is:
* `T: From<U>` implies `U: Into<T>`,
* `From` is reflexive on `T`, thus `Into` is also reflexive.

As such, the general advice is that, library authors should not directly implement any `U: Into<T>`, but implement `T: From<U>` instead, as the later offers greater flexibility, and provides the equivalent `Into` implementation anyway.

These two traits are used broadly within the standard library.