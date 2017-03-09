% std::ops

This module contains traits that allows the user to overload certain operators.

The first set of traits deals with numerical operators, including `+`, `-`, `*`, `/`, `%`, and bitwise operators `&`, `|`, `^`, `<<` and `>>`:

```ignore
pub trait Add<RHS = Self> {
    type Output;
    fn add(self, rhs: RHS) -> Self::Output;
}

pub trait Sub<RHS = Self> {
    type Output;
    fn sub(self, rhs: RHS) -> Self::Output;
}

pub trait Mul<RHS = Self> {
    type Output;
    fn mul(self, rhs: RHS) -> Self::Output;
}

pub trait Div<RHS = Self> {
    type Output;
    fn div(self, rhs: RHS) -> Self::Output;
}

pub trait Rem<RHS = Self> {
    type Output = Self;
    fn rem(self, rhs: RHS) -> Self::Output;
}

pub trait BitAnd<RHS = Self> {
    type Output;
    fn bitand(self, rhs: RHS) -> Self::Output;
}

pub trait BitOr<RHS = Self> {
    type Output;
    fn bitor(self, rhs: RHS) -> Self::Output;
}

pub trait BitXor<RHS = Self> {
    type Output;
    fn bitxor(self, rhs: RHS) -> Self::Output;
}

pub trait Shl<RHS> {
    type Output;
    fn shl(self, rhs: RHS) -> Self::Output;
}

pub trait Shr<RHS> {
    type Output;
    fn shr(self, rhs: RHS) -> Self::Output;
}
```

Note that these traits consumes both parameters by value.

Next comes a set of traits dealing with assigning-operators, including `+=`, `-=`, `*=`, `/=`, `%=`, `&=`, `|=` and `^=`:

```ignore
pub trait AddAssign<Rhs = Self> {
    fn add_assign(&mut self, Rhs);
}

pub trait SubAssign<Rhs = Self> {
    fn sub_assign(&mut self, Rhs);
}

pub trait MulAssign<Rhs = Self> {
    fn mul_assign(&mut self, Rhs);
}

pub trait DivAssign<Rhs = Self> {
    fn div_assign(&mut self, Rhs);
}

pub trait RemAssign<Rhs = Self> {
    fn rem_assign(&mut self, Rhs);
}

pub trait BitAndAssign<Rhs = Self> {
    fn bitand_assign(&mut self, Rhs);
}

pub trait BitOrAssign<Rhs = Self> {
    fn bitor_assign(&mut self, Rhs);
}

pub trait BitXorAssign<Rhs = Self> {
    fn bitxor_assign(&mut self, Rhs);
}

pub trait ShlAssign<Rhs> {
    fn shl_assign(&mut self, Rhs);
}

pub trait ShrAssign<Rhs> {
    fn shr_assign(&mut self, Rhs);
}
```

Additionally, some unary-operators can also be overloaded, including: `-`, `!`, `*`:

```ignore
pub trait Neg {
    type Output;
    fn neg(self) -> Self::Output;
}

pub trait Not {
    type Output;
    fn not(self) -> Self::Output;
}
```

The dereference operator `*` is kind of special. According to the context, a `*v` can be treated either as a lvalue( e.g. `let x = *y;`) or a rvalue (e.g. `*x = y;`). Thus, two different traits are provided for this operator.

Under the lvalue context, we have

```ignore
pub trait Deref {
    type Target: ?Sized;
    fn deref(&self) -> &Self::Target;
}
```

Thus, say we have

```rust
use std::ops::Deref;

struct Foo {
    u: i32,
    v: i32,
}

impl Deref for Foo {
    type Target = i32;

    fn deref(&self) -> &i32 {
        &self.u
    }
}
```

Then

```rust
# use std::ops::Deref;

# struct Foo {
#    u: i32,
#    v: i32,
# }

# impl Deref for Foo {
#     type Target = i32;
# 
#     fn deref(&self) -> &i32 {
#         &self.u
#     }
# }

let foo = Foo { u: 2, v: 3 };
let x = *foo; // let x: i32 = foo.u
assert_eq!(x, 2);
```

Under the rvalue context, we have:

```ignore
pub trait DerefMut: Deref {
    fn deref_mut(&mut self) -> &mut Self::Target;
}
```

Note `DerefMut` was built on top of `Deref`. This ensures that the underlying type returned by different context stays consistent.

Continuing with our example above, we implement `DerefMut` for `Foo` as well:

```rust
use std::ops::DerefMut;
# use std::ops::Deref;

# struct Foo {
#    u: i32,
#    v: i32,
# }

# impl Deref for Foo {
#     type Target = i32;
# 
#     fn deref(&self) -> &i32 {
#         &self.u
#     }
# }

impl DerefMut for Foo {
    fn deref_mut(&mut self) -> &mut i32 {
        &mut self.v
    }
}

let mut foo = Foo { u: 0, v: 0 };
*foo = 2; // foo.v = 2
assert_eq!(0, *foo); // foo.u == 0
assert_eq!(2, foo.v); // foo.v == 2
```

In our example, we implement the two traits such that they actually return references to *different* places. This is merely a demonstration of the ability that we *can*, but in practice I don't see any reason that we *should*, as this may cause confusion to the user.

Additionally, the module contains two traits for indexing operator `[]`. Like `*`, depending on the context, one of the two trait methods would be chosen.

```ignore
pub trait Index<Idx: ?Sized> {
    type Output: ?Sized;
    fn index(&self, index: Idx) -> &Self::Output;
}

pub trait IndexMut<Idx: ?Sized>: Index<Idx> {
    fn index_mut(&mut self, index: Idx) -> &mut Self::Output;
}
```

Additionally, the module contains traits for operator`()`: `FnOnce`, `FnMut` and `Fn`. As the time of writing these three traits act more like a marker. Their usage and implementation are feature-gated, thus we might not discuss them here.

Finally, the module contains the special trait `Drop`. This trait doesn't have any operator associated. Its associated method would get invoked when the value goes out of scope.

```ignore
pub trait Drop {
    fn drop(&mut self);
}
```