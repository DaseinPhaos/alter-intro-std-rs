% std::cmp

> https://doc.rust-lang.org/std/cmp/

This module provides functionality for ordering and comparison.

Rust have 6 binary operators (`==`, `!=`, `<`, `>`, `<=`, `>=`) for comparison. They are just syntactic sugar for calls to trait methods defined in this module.

Namely, the `PartialEq` trait defines methods for `==`(`eq`) and `!=`(`ne`):

```ignore
pub trait PartialEq<Rhs: ?Sized = Self> {
    // for `==`
    fn eq(&self, rhs: &Rhs) -> bool;
    
    // for `!=`
    #[inline]
    fn ne(&self, rhs: &Rhs) -> bool {
        !self.eq(rhs)
    }
}
```

Note that, as the trait name suggests, the relationship defined is a [partial equivalence relation](https://en.wikipedia.org/wiki/Partial_equivalence_relation), meaning that, in addition to `a!=b` meaning `!(a==b)`, the equality relation should be both *symmetric*(if `a==b` then `b==a`) and `transitive`(if `a==b` and `b==c` then `a==c`).


The `PartialOrd` trait defines the rest operators. Note that it derives from `PartialEq<Rhs>`.

```ignore
pub trait PartialOrd<Rhs: ?Sized = Self> : PartialEq<Rhs> {
    fn partial_cmp(&self, rhs: &Rhs) -> Option<Ordering>;

    // for `<`
    #[inline]
    fn lt(&self, rhs: &Rhs) -> bool {
        match self.partial_cmp(rhs) {
            Some(Less) => true,
            _ => false,
        }
    }

    // for `<=`
    #[inline]
    fn le(&self, rhs: &Rhs) -> bool {
        match self.partial_cmp(rhs) {
            Some(Less) | Some(Equal) => true,
            _ => false,
        }
    }

    // for `>`
    #[inline]
    fn gt(&self, rhs: &Rhs) -> bool {
        match self.partial_cmp(rhs) {
            Some(Greater) => true,
            _ => false,
        }
    }

    // for `>=`
    #[inline]
    fn ge(&self, rhs: &Rhs) -> bool {
        match self.partial_cmp(rhs) {
            Some(Greater) | Some(Equal)  => true,
            _ => false,
        }
    }
}
```

Note that only the `partial_cmp` method needs to be implemented, the actual operator-methods is default generalized using this method.

The method returns an `Option<Ordering>`, where `Ordering` is defined as:

```ignore
pub enum Ordering {
    Less = -1,
    Equal = 0,
    Greater = 1,
}
```

Being optional means that, values of the implementing type do not necessarily need to be total-ordered. Take floating point numbers as an example, when comparing `NaN` against any other floating point values, the method would always return `None`.

The defined relation should be both *antisymmetric*(if `a<b` then `!(a>b)`; if `a>b` then `!(a<b)`) and *transitive*(if `a<b` and `b<c` then `a<c`; the same goes for `>`).


Additionally, the module provides `Eq`:

```ignore
pub trait Eq: PartialEq<Self> { }
```

for [equivalence relation](https://en.wikipedia.org/wiki/Equivalence_relation). In addition to `a==b` and `a!=b` being strict inverses, the equality relation should be symmetric, transitive as well as *reflexive*(a==a).

It also provides `Ord`:

```ignore
pub trait Ord: Eq + PartialOrd<Self> {
    fn cmp(&self, rhs: &Self) -> Ordering;
}
```

for [total order](https://en.wikipedia.org/wiki/Total_order). This means that, for all values in the implementing type: exactly one of `a<b`, `a==b` or `a>b` can be true; and that `a<b` and `b<c` implies `a<c`, the same goes for both `==` and `>`.

This property is also reflected in the required method `cmp`: it returns an `Ordering`, instead of an `Option<Ordering` as in the case of `PartialOrd<Self>`.

A common trick can be used when implementing `Ord`: as the trait implies a total order, when implementing `PartialOrd<Self>` for supplementary, the `partial_cmp` method can directly calls into `cmp` to avoid duplicated typing. For example,

```rust
#[derive(Eq)]
struct Student {
    id: u32,
    grade: u32,
    name: String,
}

impl Ord for Student {
    fn cmp(&self, other: &Student) -> Ordering {
        self.grade.cmp(other.grade)
    }
}

impl PartialOrd for Student {
    fn partial_cmp(&self, other: &Student) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

impl PartialEq for Student {
    fn eq(&self, other: &Student) -> bool {
        self.grade == other.grade
    }
}
```

Finally, note that all these traits can be used with the `#[derive]` attribute if all of the type's fields also implement the corresponding trait.

Also note that the `Ordering` enum has some useful methods defined:

```ignore
impl Ordering {
    #[inline]
    pub fn reverse(self) -> Ordering {
        match self {
            Less => Greater,
            Equal => Equal,
            Greater => Less,
        }
    }
}
```

> TODO : introduce unstable methods.

It also implements `Eq`, `PartialEq`, `PartialOrd`, `Ord`, `Copy`, `Clone`, `Debug`, `Hash`, as we'd expect.

Additionally, the module defines two utility methods for comparing and returning values of total-ordered types:

```ignore
#[inline]
pub fn min<T: Ord>(v1: T, v2: T) -> T {
    if v1 <= v2 { v1 } else { v2 }
}

#[inline]
pub fn max<T: Ord>(v1: T, v2: T) -> T {
    if v2 >= v1 { v2 } else { v1 }
}
```