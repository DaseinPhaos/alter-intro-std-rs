% std::mem

> https://doc.rust-lang.org/std/mem/

This module contains functions dealing with memories.

```ignore
pub fn size_of<T>() -> usize
```

returns the size of a type, in bytes.

---

```ignore
pub fn size_of_val<T: ?Sized>(val: &T) -> usize
```

returns the size of the type of the pointed value, in bytes.

Unlike `size_of`, this function can be used to get the dynamically-known size of a DST.

---

```ignore
pub fn align_of<T>() -> usize
```

returns the ABI-required minimum alignment of a type.

---

```ignore
pub fn align_of_val<T: ?Sized>(val: &T) -> usize
```

returns the ABI-required minimum alignment of the type of value that reference `val` points to.

---

```ignore
pub fn drop<T>(_x: T) { }
```

Disposes a value.

---

```ignore
pub fn forget<T>(t: T)
```

Unlike `drop`, `forget` takes ownership of the passed value, but *forgets* it: **no destructor invoked!**

Common use cases:

- an uninitialized value produced by `mem::uninitialized` should either be initialized or forgotten before it goes out of the scope. Otherwise, running dtors on it would cause UB.

- a bitwise-duplication of a droppable item was made. One of the value has to be forgotten before going out of scope, or double-dropping would introduce UB.

- transferring ownership across FFI.

---

```ignore
pub fn swap<T>(x: &mut T, y: &mut T)
```

Swaps the values at two mutable locations, without deinitializing either one.

---

```ignore
pub fn replace<T>(dest: &mut T,mut src: T) -> T {
    swap(dest, &mut src);
    src
}
```

replaces the value at `dest` with `src`, returning the old value at `dest`, without deinitializing either one.

---

```ignore
pub unsafe extern "rust-intrinic" fn transmute<T, U>(e: T) -> U
```

reinterprets the bits of a value of one type as another. The compiler only makes sure that `size_of<T>() == size_of<U>()`, nothing more. It is semantically equivalent to a bitwise move of a value of one type into a value of another type, After the transmutation the original type would be forgotten thus **no dropping** occurs.

This is marked as `unsafe`, because the caller must ensure that Rust's safety model won't be violated. Some common hints:

- The returned value must be in a valid state.
- Transmuting between non-repr(C) types is UB.
- Transmuting an immutable reference to a mutable one is UB.

---

```ignore
pub unsafe fn transmute_copy<T, U>(src: &T) -> U
```

reinterprets the bits of a value of a pointed type as another. The compiler made **no** assumption about the relation on size of the two types. It is semantically equivalent to a bitwise **copy**, starting from the location pointed by `src`, with a size of `size_of<U>()`, and returns the copied result as `U`.

Like `transmute`, this function is marked as `unsafe`. Callers must ensure that Rust's safety model won't be violated. Common hints include:

- Making sure that the returned value of type `U` is in a valid state.
- Transmuting from a `src: &T` to `U` where `size_of_val(src) < size_of<U>()` *always* introduces UB.

---

```ignore
pub unsafe fn uninitialized<T>() -> T
```

bypasses Rust's normal memory initialization checks, by pretending to produce a valid value of type `T`.

This is useful when dealing with `FFI` or initializing some complex arrays.

But the caller must ensure that any subsequent usage of the returned type *actually* respects that the value was uninitialized:

- reading always causes UB.
- writing to such a value of type `T` where `T: Drop` or recursively contains any droppable fields also causes UB, because a value at invalid state would have been dropped before the new value being written in. The only way to safely initialize such a value is with `std::ptr::write`, `std::ptr::copy` or `std::ptr::copy_nonoverlapping`.
- before it goes out of scope, it must be either forgotten or be given a valid state. Otherwise, subsequent dropping may introduce UB.

The caller must make sure that such a value *is* properly initialized into a valid state before it can be used outside the `unsafe` code blocks.

---

```ignore
pub unsafe fn zeroed<T>() -> T
```

Creates a value whose bits are all `0`.

Being `unsafe`, this gives the same responsibility to the caller as `uninitialized` does.

