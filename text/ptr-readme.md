% Raw Pointers

Rust provides two raw pointers, `*const T` and `*mut T`.

The most common way to create raw pointers is to coerce from a reference, `&T` or `&mut T`.

To check whether a pointer is `null`:

```ignore
impl<T: ?Sized> *const T {
    pub fn is_null(self) -> bool where T: Sized {/*...*/}
}

impl<T: ?Sized> *mut T {
    pub fn is_null(self) -> bool where T: Sized {/*...*/}
}
```

To convert a raw pointer into a reference:

```ignore
impl<T: ?Sized> *const T {
    pub unsafe fn as_ref<'a>(self) -> Option<&'a T> where T: Sized {
        if self.is_null() { None }
        else { Some(&*self) }
    }
}

impl<T: ?Sized> *mut T {
    pub unsafe fn as_ref<'a>(self) -> Option<&'a T> where T: Sized {
        if self.is_null() { None }
        else { Some(&*self) }
    }
}
```

This method is marked as `unsafe`, because the caller must ensure that during `'a` the value pointed by the original pointer is actually valid.

To offset a pointer in `count` elements:

```ignore
impl<T: ?Sized> *const T {
    pub unsafe fn offset(self, count: isize) -> *const T where T: Sized {/*...*/}
}

impl<T: ?Sized> *mut T {
    pub unsafe fn offset(self, count: isize) -> *mut T where T: Sized {/*...*/}
}
```

Note that `count` is in the unit of `T`. That is, the pointer would be offset in `count * size_of::<T>()` bytes.

The method is `unsafe`， because the caller mst be in-bounds of the object, or one-byte-pass-the-end. Otherwise, it would always cause UB, not matter whether the pointer returned get used or not.

Another set of methods, `wrapping_offset` don't have this UB, but we should always use `offset` when possible, as the compiler optimize it better.


Additionally, we can obtain a mutable reference from `*mut T`:

```ignore
impl<T: ?Sized> *mut T {
    pub unsafe fn as_mut<'a>(self) -> Option<&'a mut T> where T: Sized {
        if self.is_null() { None }
        else { Some(&mut *self) }
    }
}
```

As with `as_ref`, this is `unsafe` because it cannot ensure that the lifetime `'a` returned is indeed a valid lifetime for the contained data. Additionally, the caller must ensure that the pointed data would *only* be referenced by the returned `&'a mut T` during `'a`.

# std::ptr

This module provides additional functionalities to manipulate pointers.

To convert equality of two raw pointers:

```ignore
pub fn eq<T: ?Sized>(a: *const T, b: *const T) -> bool {
    a == b
}
```

To create a null pointer:

```ignore
pub const fn null<T>() -> *const T { 0 as *const T }

pub const fn null_mut<T>() -> *mut T { 0 as *mut T }
```

To copy bytes:

```ignore
pub unsafe fn copy<T>(src: *const T, dst: *mut T, count: usize);

pub unsafe fn copy_nonoverlapping<T>(src: *const T, dst: *mut T, count: usize);
```

These method semantically moves `count * sie_of<T>()` bytes from `src` to `dst`. They are marked as unsafe, because it is up to the caller to ensure that data at `src` is not being modified during the invocation of the caller, the caller is the only one modifying `dst` during the invocation. Additionally, for the case of `copy`, it must also ensure that the contents being copied don't overlap.

Whenever possible, one should prefer using `copy`, as it might be more efficient.

To read:

```ignore
pub unsafe fn read<T>(src: *const T) -> T;

pub unsafe fn read_unaligned<T>(src: *const T) -> T;

pub unsafe fn read_volatile<T>(src: *const T) -> T;
```

These methods would construct a `T` from the value at `src`. Apart from the usual safety requirements regarding a raw pointer, it is up to the caller to make sure that further usage of `src` would be semantically correct. For the case of `read` `src` must also be an aligned pointer. `read_volatile`s are guaranteed to not be elided or reordered by the compiler across other volatile operations.

To write:

```ignore
pub unsafe fn write<T>(dst: *mut T, src: T);

pub unsafe fn write_unaligned<T>(dst: *mut T, src: T);

pub unsafe fn write_volatile<T>(dst: *mut T, src: T);
```

These methods would overwrite memory at `dst` with the given value `src` without reading or dropping the old value at that position. Apart from the usual safety requirements regarding a mutable pointer, `write` only accepts aligned pointer. `write_unaligned` don't have this requirement. `write_volatile`s are guaranteed to not be elided or reordered by the compiler across other volatile operations.

To write raw bytes:

```ignore
pub unsafe fn write_bytes<T>(dst: *mut T, val: u8, count: usize);
```

This would set `count * size_of<T>()` bytes of memory starting at `dst` to `val`.

To replace the pointed value, returning the old value without dropping either:

```ignore
pub unsafe fn replace<T>(dest: *mut T, src: T) -> T {
    mem::swap(&mut *dest, &mut src);
    src
}
```

To swap values at two mutable locations, without deinitializing either:

```ignore
pub unsafe fn swap<T>(x: *mut T, y: *mut T) {
    let mut tmp: T = mesm::uninitialized();
    copy_nonoverlapping(x, &mut tmp, 1);
    copy(y, x, 1);
    copy_nonoverlapping(&tmp, y, 1);

    mem::forget(tmp);
}
```

To drop a value in place:

```ignore
pub unsafe fn drop_in_place<T>(to_drop: *mut T);
```

This function execute the drop (if any) of the pointed-to value. This is a compliment for `mem::drop`, because `?Sized` types like trait objects can only be dropped this way.
