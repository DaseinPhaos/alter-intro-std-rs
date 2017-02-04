% std::cell

> https://doc.rust-lang.org/std/cell/

This module introduces "interior mutability" into the language. There is [a section](https://doc.rust-lang.org/book/mutability.html) in The Book which is dedicated to explaining this matter.


# UnsafeCell

To really understand what's going on under the hood, we should take a look at `UnsafeCell<T>` first:

```ignore
#[lang = "unsafe_cell"]
pub struct UnsafeCell<T: ?Sized> {
    value: T,
}

impl<T: ?Sized> !Sync for UnsafeCell<T> {}

impl<T> UnsafeCell<T> {
    #[inline]
    pub const fn new(value: T) -> UnsafeCell<T> {
        UnsafeCell { value: value }
    }

    #[inline]
    pub unsafe fn into_inner(self) -> T {
        self.value
    }
}

impl<T: ?Sized> UnsafeCell<T> {
    #[inline]
    pub fn get(&self) -> *mut T {
        &self.value as *const T as *mut T
    }
}
```

Generally speaking, when the compiler is doing its job, it'd make optimizations to the code based on the knowledge that a `&T` is not mutably aliased or mutated, and that a `&mut T` is the unique mutating reference to its referent. As such, transmuting an `&T` into an `&mut T` is considered undefined behavior, because it violates the compiler's assumption about data mutability.

However, interior mutability by its nature *is* such a violation. To inform the compiler to turn off related optimizations(thus remove the undefinedness), we have no way but to perform some magic. The `#[lang = "unsafe_cell"]` attribute is exactly that magic, making `UnsafeCell<T>` the only legal building block for interior mutability in the language. When an `UnsafeCell<T>` is immutably aliased, it's still safe to obtain mutable reference to its interior `value: T` and/or to mutate it.

However, it is now up to the user to ensure that, no two mutable references obtained this way are active at the same time, and that there are no active mutable reference whe an immutable reference is obtained from the cell. This is why the `get` method returns a raw pointer, and the `into_inner` is being marked as `unsafe`.

Another thing to notice is that, what this magical cell provides is only the ability to mutating or mutably aliasing the contents of an `&UnsafeCell<T>`. Nothing else. Any other kind of violation to Rust's mutability system, say, having multiple `&mut UnsafeCell<T>` referencing the same referent simultaneously, is still undefined behavior.

Additionally, note that `UnsafeCell<T>` implements `From<T>`. It also implements `Default` and/or `Debug` when `T` implements those traits, respectively.

One thing to keep in mind is that, `!Sync` is also implemented for `UnsafeCell<T>`, meaning that by default this is only intended for single-threaded scenarios. The reason should be obvious if you're familiar with multithread programming and Rust's safely promise. Same thing goes for `Cell` and `RefCell`, which we are going to cover next.

# Cell

On top of `UnsafeCell`, lies the "safe" `Cell` for safe code to use:

```ignore
pub struct Cell<T> {
    value: UnsafeCell<T>,
}

impl<T: Copy> Cell<T> {
    #[inline]
    pub const fn new(value: T) -> Cell<T> {
        Cell { value: UnsafeCell::new(value) }
    }

    #[inline]
    pub fn get(&self) -> T {
        unsafe{ *self.value.get() }
    }

    #[inline]
    pub fn set(&self, value: T) {
        unsafe { *self.value.get() = value; }
    }

    #[inline]
    pub fn as_ptr(&self) -> *mut T {
        self.value.get()
    }

    #[inline]
    pub fn get_mut(&mut self) -> &mut T {
        unsafe { &mut *self.value.get() }
    }
}
```

`Cell<T>` is designed for `T: Copy` to use. It's `set` method allows the celled value to be mutated through an `& self`. 

Note that, instead of returning an `&T`, which, under this case of interior mutability can't guarantee that its referent is actually immutable; the `get` method returns a copy of the celled value. As such, while the returned value is indeed an equivalent of the celled value, Safe Rust's assumption about mutability has also been maintained. 

Note that the `get_mut` method requires a `&mut self`. This behavior is in consistent with our statement above that, interior mutability only introduces mutation through immutable reference, *not* mutation through multiple mutable reference. That is to say, the following code will still fail to compile, as we'd normally expect:

```ignore
fn main() {
    use std::cell::Cell;
    let mut mutable = Cell::new(1);
    println!("{:?}", mutable);
    let mutable_ref = mutable.get_mut();
    *mutable_ref = 2;
    println!("{:?}", mutable_ref);
    let mutable_ref_2 = mutable.get_mut();
    *mutable_ref = 3;
    println!("{:?}", mutable_ref_2);
}
```

Compiling the code fragment above (under rustc 1.14.0) would fail with something like:

```text
error[E0499]: cannot borrow `mutable` as mutable more than once at a time
   --> src\main.rs:8:25
    |
  4 |     let mutable_ref = mutable.get_mut();
    |                       ------- first mutable borrow occurs here
...
  8 |     let mutable_ref_2 = mutable.get_mut();
    |                         ^^^^^^^ second mutable borrow occurs here
...
 11 | }
    | - first borrow ends here
```

Also note that `Cell<T>` implements `Clone`, `From<T>` and `!Sync`. Additionally, it implements `Eq`, `PartialOrd<Cell<T>>`, `PartialEq<Cell<T>>`, `Send`, `Ord`, `Default`, `Debug` when `T` implements those triats, respectively.

# RefCell

Remember that, `UnsafeCell<T>` requires its `unsafe` user to guarantee that Safe Rust's mutability model wouldn't be violated. Now the thing is, with interior mutability kicking in, Safe Rust's conventional way of referencing things immutably, `& T`, actually cannot guarantee the immutability of its referent. As such, `& T` can no longer be directly used by Safe Rust. For `T: Copy`, `Cell<T>` gets away by returning a copy; for uncopyable types however, we need something more than that.

Standard library introduces `RefCell<T>`(with a bunch of related types) for this. It uses Rust's lifetime to implement 'dynamic borrowing', where one can claim temporary, exclusive, mutable access to the inner value. Such borrowing are tracked at runtime, so that Rust's mutability model won't be actually violated. As such, interior mutability is safely introduced into a wider scope.

Its implementation is also a good demonstration for `Cell`'s usage, so let's take a look:

```ignore
type BorrowFlag = usize;
const UNUSED: BorrowFlag = 0;
const WRITING: BorrowFlag = !0;

pub struct RefCell<T: ?Sized> {
    borrow: Cell<BorrowFlag>,
    value: UnsafeCell<T>,
}
```

As we can see inside `RefCell<T>`, in addition to the `value: UnsafeCell<T>` field where the actual value is stored, the `borrow: Cell<BorrowFlag>` field represents a flag indicating how the value is currently being dynamically borrowed. As such, logically speaking, the `borrow` flag shouldn't get involved in the (static) mutability consideration of `RefCell<T>`, thus it is wrapped in a `Cell`.

Now let's checkout `RefCell`'s methods:

```ignore
impl<T> RefCell<T> {
    pub const fn new(value: T) -> RefCell<T> { /* ... */}

    pub fn into_inner(self) -> T { /* ... */}
}

impl<T: ?Sized> RefCell<T> {
    pub fn try_borrow(&self) -> Result<Ref<T>, BorrowError> { /* ... */}

    pub fn borrow(&self) -> Ref<T> { /* ... */}

    pub fn try_borrow_mut(&self) -> Result<RefMut<T>, BorrowMutError> { /* ... */}

    pub fn borrow_mut(&self) -> RefMut<T> { /* ... */}

    pub fn as_ptr(&self) -> *mut T { /* ... */}

    pub fn get_mut(&mut self) -> &mut T { /* ... */}
}
```

As usual, `new` constructs a new `RefCell`, `into_inner` returns the celled value.

`try_borrow` will try to immutably borrow the wrapped value. If the value is currently mutably borrowed, it would return an `Err`. If not, the value of the `borrow` flag would increment by 1, indicating that another immutable borrow has been made, then a custom reference type `Ref<T>` would be returned (inside an `Ok` of course). It has a `panic`ing variant, `borrow`.

`try_borrow_mut` will try to mutably borrow the wrapped value. Note that it only requires a statically immutable `&self`, such "interior mutable" semantics is, after all, what we've been after. If the value is currently being borrowed (either mutably or not), it would return an `Err`. If not, then the value of the `borrow` flag would be set to `WRITING`, then a custom reference type `RefMut<T>` would be returned. It also has a `panic`ing variant `borrow_mut`.[^borrow_state]

[^borrow_state]: Actually `RefCell` has yet another method `borrow_state`. We didn't cover it in the text because it is unstable currently, and it won't be stable in any foreseeable future, as discussed in its related [issue](https://github.com/rust-lang/rust/issues/27733). The discussion itself, however, is worth mentioning, because it explained some general philosophy behind the interface designing of the standard library. Anyone interested should take a look.

Now `BorrowError` and `BorrowMutError` are just some random error types with their formatting traits defined:

```ignore
pub struct BorrowError {
    _private: (),
}

impl Debug for BorrowError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        f.debug_struct("BorrowError").finish()
    }
}

impl Display for BorrowError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        Display::fmt("already mutably borrowed", f)
    }
}

pub struct BorrowMutError {
    _private: (),
}

impl Debug for BorrowMutError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        f.debug_struct("BorrowMutError").finish()
    }
}

impl Display for BorrowMutError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        Display::fmt("already borrowed", f)
    }
}
```

What really interests us should be the reference type `Ref` and `RefMut`.

Let's take a look at type `Ref` first.  As mentioned above, mutability should be checked dynamically. Thus, when any `Ref` goes out of scope, the referent's `borrow` field should be set back to the state before borrowing. That is, its value should decrement by `1`.

The action is done when `Ref` goes out of scope. In Rust that is to say, `Ref` should implements the `Drop` trait and the action should happen in the `drop` method. Also, being a reference means that the `Deref` trait should be implemented. Now let's check out what `Ref` really do:

```ignore
pub struct Ref<'b, T: ?Sized + 'b> {
    value: &'b T,
    borrow: BorrowRef<'b>,
}

impl<'b, T: ?Sized> Deref for Ref<'b, T> {
    type Target = T;

    #[inline]
    fn deref(&self) -> &T {
        self.value
    }
}
```

Obviously, the `value: &'b T` field should be the reference pointing to the referenced `RefCell<T>`'s celled value. Also notice that it indeed implements the `Deref` trait (with the implementation exactly as we'd expect). However, the `Drop` trait is nowhere to be seen. In fact, it makes use of another type, `BorrowRef<'b>`, internally, which in turn implements `Drop`. Let's take a look at that type:

```ignore
struct BorrowRef<'b> {
    borrow: &'b Cell<BorrowFlag>,
}

impl<'b> BorrowRef<'b> {
    #[inline]
    fn new(borrow: &'b Cell<BorrowFlag>) -> Option<BorrowRef<'b>> {
        match borrow.get() {
            WRITING => None,
            b => {
                borrow.set(b + 1);
                Some(BorrowRef{ borrow: borrow })
            },
        }
    }
}

impl<'b> Drop for BorrowRef<'b> {
    #[inline]
    fn drop(&mut self) {
        let borrow = self.borrow.get();
        debug_assert!(borrow != WRITING && borrow != UNUSED);
        self.borrow.set(borrow - 1);
    }
}

impl<'b> Clone for BorrowRef<'b> {
    #[inline]
    fn clone(&self) -> BorrowRef<'b> {
        let borrow = self.borrow.get();
        debug_assert!(borrow != UNUSED);
        assert!(borrow != WRITING);
        self.borrow.set(borrow + 1);
        BorrowRef{ borrow: self.borrow }
    }
}
```

It turns out that this type is simply just a wrapper around a reference to a celled flag.

The associated `new` method is somewhat unusual as it returns an `Option` (instead of `Self` as what we'd normally expect). internally, it determines if the flag presented is `WRITING`(indicating an immutable borrow is active). If it is, the method returns `None` indicating borrowing failure; it not, it returns a `Some` with the referenced flag count increnmented (indicating another immutable borrow has been activated).

The `Drop` trait's `drop` method, on the contrary, decrements the referenced flag count, indicating that current immutable borrow has gone out of scope.

The `Clone` implementation by now should appears to be trivial.

This addtional layer of abstraction allows the following method to be implemented:

```ignore
impl<'b, T: ?Sized> Ref<'b, T> {
    pub fn map<U: ?Sized, F>(orig: Ref<'b, T>, f: F) -> Ref<'b, U>
        where F: FnOnce(&T) -> &U {
        Ref {
            value: f(orig.value),
            borrow: orig.borrow,
        }
    }
}
```

The `map` function takes an `Ref<'b, T>`, a function `FnOnce<&T) -> &U` and returns a `Ref<'b, U>`. This function can be useful when we are only interested in some subfields of the originally referenced type. Take this example:

```rust
use std::cell::{RefCell, Ref};
let c = RefCell::new((5, 'b'));
let b1 = c.borrow(); // b1: Ref<(i32, char)>
let b2 = Ref::map(b1, |t| &t.0); // b2: Ref<i32>
assert_eq!(*b2, 5);
```

Separating the management of borrow checking out (into a `BorrowRef<'b>`) from the reference type `Ref<'b, T>` makes the implementation of such function trivial.

Notice that this is an associated function rather than a method, meaning that it can only be used as `Ref::map`. This is it so that when `Ref<'b, T>` gets automatically dereferenced, the function's (commonly seen) name `map` wouldn't interfere with methods of the same name on the dereferenced `T` type.

Similarly, for mutable referencing there are `RefMut<'b, T>` and `BorrowMutRef<'b>`:

```ignore
struct BorrowRefMut<'b> {
    borrow: &'b Cell<BorrowFlag>,
}

impl<'b> Drop for BorrowRefMut<'b> {
    #[inline]
    fn drop(&mut self) {
        let borrow = self.borrow.get();
        debug_assert!(borrow == WRITING);
        self.borrow.set(UNUSED);
    }
}

impl<'b> BorrowRefMut<'b> {
    #[inline]
    fn new(borrow: &'b Cell<BorrowFlag>) -> Option<BorrowRefMut<'b> {
        match borrow.get() {
            UNUSED => {
                borrow.set(WRITING);
                Some(BorrowRefMut{ borrow: borrow]})
            },
            _ => None,
        }
    }
}

pub struct RefMut<'b, T: ?Sized + 'b> {
    value: &'b mut T,
    borrow: BorrowRefMut<'b>,
}

impl<'b, T: ?Sized> Deref for RefMut<'b, T> {
    type Target = T;

    #[inline]
    fn deref(&self) -> &T {
        self.value
    }
}

impl<'b, T: ?Sized> DerefMut for RefMut<'b, T> {
    #[inline]
    fn deref_mut(&mut self) -> &mut T {
        self.value
    }
}

impl<'b, T: ?Sized> RefMut<'b, T> {
    pub fn map<U: ?Sized, F>(orig: RefMut<'b, T>, f: F)
        where F: FnOnce(&mut T) -> &mut U {
        RefMut {
            value: f(prig.value),
            borrow: orig.borrow,
        }
    }
}
```

With these structures introduced, now we can finally check out how the methods of `RefCell<T>` are implemented:

```ignore
impl<T> RefCell<T> {
    #[inline]
    pub const fn new(value: T) -> RefCell<T> {
        RefCell {
            value: UnsafeCell::new(value),
            borrow: Cell::new(UNUSED),
        }
    }

    #[inline]
    pub fn into_inner(self) -> T {
        debug_assert!(self.borrow.get() == UNUSED);
        unsafe { self.value.into_inner() }
    }
}

impl<T: ?Sized> RefCell<T> {
    #[inline]
    pub fn try_borrow(&self) -> Result<Ref<T>, BorrowError> {
        match BorrowRef::new(&self.borrow) {
            Some(b) => Ok(Ref {
                value: unsafe { &*self.value.get() },
                borrow: b,
            }),
            None => Err(BorrowError {_private: () }),
        }
    }

    #[inline]
    pub fn borrow(&self) -> Ref<T> {
        self.try_borrow().expect("already mutably borrowed")
    }

    #[inline]
    pub fn try_borrow_mut(&self) -> Result<RefMut<T>, BorrowMutError> {
        match BorrowRefMut::new(&self.borrow) {
            Some(b) => Ok(RefMut {
                value: unsafe { &mut *self.value.get() },
                borrow: b,
            }),
            None => Err(BorrowMutError { _private: () } ),
        }
    }

    #[inline]
    pub fn borrow_mut(&self) -> RefMut<T> {
        self.try_borrow_mut().expect("already borrowed")
    }

    #[inline]
    pub fn as_ptr(&self) -> *mut T {
        self.value.get()
    }

    #[inline]
    pub fn get_mut(&mut self) -> &mut T {
        unsafe { &mut *self.value.get() }
    }
}
```

Simple and clear.

Like `UnsafeCell<T>`, `RefCell<T>` implements `From<T>` and `!Sync`. It also implements `Send`, `Clone`, `Default`, `Debug`, `Eq`, `Ord`, `PartialEq<RefCell<T>>`, `PartialOrd<RefCell<T>>` when `T` implements those traits, respectively.[^final-note]

[^final-note]: As [RFC 1651](https://github.com/rust-lang/rfcs/pull/1651) points out, the API design of `Cell` can be altered to support non-`Copy` types, check it out for more details. The actual implementation is nowhere under its way into the standard library yet, thus we wouldn't cover it here.