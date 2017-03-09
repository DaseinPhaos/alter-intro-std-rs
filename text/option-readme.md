% std::option

This module contains definition for `Option` and some relevant structs.

```ignore
pub enum Option<T> {
    None,
    Some(T),
}
```

An `Option<T>` represents a value that can optionally hold a value of type `T`. Being an `enum`, whether it actually holds a value (`Some(T)`) or not (`None`) is always determined. It was *very* very commonly used in the Rust world.

It is commonly paired with pattern matching. As such, some utility methods are defined.

---

To determine whether the underlying value is `Some` or `None`:

```ignore
impl<T> Option<T> {
    pub fn is_some(&self) -> bool {
        match *self {
            Some(_) => true,
            None => false,
        }
    }

    pub fn is_none(&self) -> bool {
        !self.is_some()
    }
}
```

---

To borrow the underlying value (if any):

```ignore
impl<T> Option<T> {
    pub fn as_ref(&self) -> Option<&T> {
        match *self {
            Some(ref x) -> Some(x),
            None => None,
        }
    }

    pub fn as_mut(&mut self) -> Option<&mut T> {
        match *self {
            Some(ref mut x) => Some(x),
            None => None,
        }
    }
}
```

---

To consume the option and return its underlying value, `panic`king if it is `None`:

```ignore
impl<T> Option<T> {
    pub fn unwrap(self) -> T {
        match self {
            Some(val) => val,
            None => panic!("called `Option::unwrap()` on a `None` value"),
        }
    }
    
    pub fn expect(self, msg: &str) -> T {
        match self {
            Som(val) => val,
            None => expect_failed(msg), // panic with the given message
        }
    }
}
```

---

To consume the option and return its underlying value, returning a `def`ault value if it is `None`:

```ignore
impl<T> Option<T> {
    pub fn unwrap_or(self, def: T) -> T {
        match self {
            Some(x) => x,
            None => def,
        }
    }

    pub fn unwrap_or_else<F: FnOnce() -> T>(self, f: F) -> T {
        match self {
            Some(x) => x,
            None => f(),
        }
    }
}
```

Additionally, for `T: Default`, we have:

```ignore
impl<T: Default> Option<T> {
    pub fn unwrap_or_default(self) -> T {
        match self {
            Some(x) => x,
            None => Default::default(),
        }
    }
}
```

---

To apply a mapping function on the underlying value and returns the result, (`map` if any, `map_or` with a default value, `map_or_else` with a default generation function):

```ignore
impl<T> Option<T> {
    pub fn map<U, F: FnOnce(T) -> U>(self, f: F) -> Option<U> {
        match self {
            Some(x) => Some(f(x)),
            None => None,
        }
    }

    pub fn map_or<U, F: FnOnce(T) -> U>(self, def: U, f: F) -> U {
        match self {
            Some(x) => f(x),
            None => def,
        }
    }

    pub fn map_or_else<U, D, F>(self, def: D, f: F) -> U
        where D: FnOnce() -> U,
              F: FnOnce(T) -> U,
    {
        match self {
            Some(x) => f(x),
            None => def(),
        }
    }
}
```

---

To transform an `Option` into a `Result`, with `Some` mapping to `Ok`:

```ignore
impl<T> Option<T> {
    pub fn ok_or<E>(self, err: E) -> Result<T, E> {
        match self {
            Some(v) => Ok(v),
            None => Err(err),
        }
    }

    pub fn ok_or_else<E, F: FnOnce() -> E>(self, err: F) -> Result<T, E> {
        match self {
            Some(v) => Ok(v),
            None => Err(err()),
        }
    }
}
```

---

To transform an `Option` into a iterator over the reference of its underlying value (if any):

```ignore
impl<T> Option<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { inner: Item { opt: self.as_ref() } }
    }

    pub fn iter_mut(&mut self) -> IterMut<T> {
        IterMut { inner: Item { opt: self.as_mut() } }
    }
}
```

Note that `Option<T>` also implements `IntoIterator<Item=T>`, `&Option<T>` implements `IntoIterator<Item=&T>`, `&mut Option<T>` implements `IntoIterator<Item=&mut T>`.

---

To chain one option with possibly another:

```ignore
impl<T> Option<T> {
    pub fn and<U>(self, optb: Option<U>) -> Option<U> {
        match self {
            Some(_) => optb,
            None => None,
        }
    }

    pub fn and_then<U, F: FnOnce(T) -> Option<U>>(self, f: F) -> Option<U> {
        match self {
            Some(x) => f(x),
            None => None,
        }
    }
}
```

---

To chain options of the same type:

```ignore
impl<T> Option<T> {
    pub fn or(self, optb: Option<T>) -> Option<T> {
        match self {
            Some(_) => self,
            None => optb,
        }
    }

    pub fn or_else<F: FnOnce() -> Option<T>>(self, f: F) -> Option<T> {
        match self {
            Some(_) => self,
            None => f(),
        }
    }
}
```

To take the underlying out (if any) and return it, leaving a `None` in place:

```ignore
impl<T> Option<T> {
    pub fn take(&mut self) -> Option<T> {
        std::mem::replace(self, None)
    }
}
```

---

For `T: Clone` and an `Option<&'a T>`, we have:

```ignore
impl<'a, T: Clone> Option<&'a T> {
    pub fn cloned(self) -> Option<T> {
        self.map(|t| t.clone())
    }
}
```

Also note that `Option<T>` implements `From<T>`.

Also note that `Option<T>` implements `Copy`, `Debug`, `Ord`, `PartialOrd`, `PartialEq`, `Eq`, `Hash`, `Clone` if `T` has corresponding traits implemented.

Also note that `Option<T>` implements `FromIterator<Option<A>>` if `T: FromIterator<A>`.

Finally, note that `Option<T>` implements `Default`, with `default()` always returning a `None`.