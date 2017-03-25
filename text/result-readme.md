% std::result

This module contains definition for the error-type `Result`:

```ignore
#[derive(Clone, Copy, PartialEq, PartialOrd, Eq, Ord, Debug, Hash)]
#[must_use]
pub enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

This type is used for returning and propagating errors. Variant `Ok` represents success, and `Err` represents an error. Both successes and errors can have values associated, the types of which are defined by generic `T` and `E`, respectively. We highlight the `must_use` attribute here: this attribute ensures that whenever the compiler encounters an unused `Result`, it will issue a warning: results indicates a possibly error, they shouldn't be implicitly ignored.

To make error handling more simple and elegant, several utility methods are defined.

To check whether the value is an `Err` or `Ok`:

```ignore
impl<T, E> Result<T, E> {
    pub fn is_ok(&self) -> bool {
        match *self {
            Ok(_) => true,
            Err(_) => false,
        }
    }

    pub fn is_err(&self) -> bool {
        !self.is_ok()
    }
}
```

To consume the result and converts the underlying variant values into their respective `Option`s:

```ignore
impl<T, E> Result<T, E> {
    pub fn ok(self) -> Option<T> {
        match self {
            Ok(v) => Some(v),
            Err(_) => None,
        }
    }

    pub fn err(self) -> Option<E> {
        match self {
            Ok(_) => None,
            Err(e) => Some(e),
        }
    }
}
```

To borrow the content and return an `Result` over the references:

```ignore
impl<T, E> Result<T, E> {
    pub fn as_ref(&self) -> Result<&T, &E> {
        match *self {
            Ok(ref v) => Ok(v),
            Err(ref e) => Err(e),
        }
    }

    pub fn as_mut(&mut self) -> Result<&mut T, &mut E> {
        match *self {
            Ok(ref mut v) => Ok(v),
            Err(ref mut e) => Err(e),
        }
    }
}
```

To map a result of some type into another type, either by applying a mapping function on `Ok` variant or `Err` variant:

```ignore
impl<T, E> Result<T, E> {
    pub fn map<U, F>(self, op: F) -> Result<U, E>
        where F: FnOnce(T) -> U,
    {
        match self {
            Ok(v) => Ok(op(v)),
            Err(e) => Err(e),
        }
    }

    pub fn map_err<U, F>(self, op: F) -> Result<T, U>
        where F: FnOnce(E) -> U,
    {
        match self {
            Ok(v) => Ok(v),
            Err(e) => Err(op(e)),
        }
    }
}
```

To convert the `Ok` variant into an iterator:

```ignore
impl<T, E> Result<T, E> {
    pub fn iter(&self) -> Iter<T> { /* ... */ }

    pub fn iter_mut(&mut self) -> IterMut<T> { /* ... */ }
}

impl<T, E> IntoIterator for Result<T, E> {
    type Item = T,
    // ...
}

impl<'a, T, E> IntoIterator for &'a Result<T, E> {
    type Item = &'a T,
    // ...
}

impl<'a, T, E> IntoIterator for &'a mut Result<T, E> {
    type Item = &'a mut T,
    // ...
}
```

To chain one result with another, such that if the current result is `Ok`, then return the next result. Otherwise, propagate the current `Err`:

```ignore
impl<T, E> Result<T, E> {
    pub fn and<U>(self, res: Result<U, E>) -> Result<U, E> {
        match self {
            Ok(_) => res,
            Err(e) => Err(e),
        }
    }

    pub fn and_then<U, F>(self, op: F) -> Result<U, E>
        where F: FnOnce(T) -> Result<U, E>,
    {
        match self {
            Ok(v) => op(v),
            Err(e) => Err(e),
        }
    }
}
```

To chain one result with another, such that if the current result is `Ok`, return it. Otherwise, return the other result:

```ignore
impl<T, E> Result<T, E> {
    pub fn or<U>(self, res: Result<T, U>) -> Result<T, U> {
        match self {
            Ok(v) => Ok(v),
            Err(_) => res,
        }
    }

    pub fn or_else<U, F>(self, op: F) -> Result<T, U>
        where F: FnOnce(E) -> Result<T, U>,
    {
        match self {
            Ok(v) => Ok(v),
            Err(e) => op(e),
        }
    }
}
```

To consume the result and return its underlying `Ok` value, panic if `Err`; or the other way around:

```ignore
impl<T, E> Result<T, E> where E: Debug {
    pub fn unwrap(self) -> T {
        match self {
            Ok(v) => v,
            Err(e) => // ...
        }
    }

    pub fn expect(self, msg: &str) -> T {
        match self {
            Ok(v) => v,
            Err(e) => // ...
        }
    }
}

impl<T, E> Result<T, E> where T: Debug {
    pub fn unwrap_err(self) -> E {
        match self {
            Ok(v) => /*...*/,
            Err(e) => e,
        }
    }

    pub fn expect_err(self, msg: &str) -> E {
        match self {
            Ok(v) => /*...*/,
            Err(e) => e,
        }
    }
}
```

Note that the above four methods requires either `E: Debug` or `T: Debug`, as they would get printed by the panic message.

To consume the result, returning the underlying `Ok` value, falling back to default values when `Err`:

```ignore
impl<T, E> Result<T, E> {
    pub fn unwrap_or(self, optb: T) -> T {
        match self {
            Ok(v) => v,
            Err(_) => optb,
        }
    }

    pub unwrap_or_else<F>(self, op: F) -> T
        where F: FnOnce(E) -> T,
    {
        match self {
            Ok(v) => v,
            Err(e) => op(e),
        }
    }
}

impl<T, E> Result<T, E> where T: Default {
    pub unwrap_or_default(self) -> T {
        match self {
            Ok(v) => v,
            Err(_) => Default::default(),
        }
    }
}
```
