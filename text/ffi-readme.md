% std::ffi

> https://doc.rust-lang.org/std/ffi/

This module defines some structs useful for FFI(foreign function interface) bindings. It primarily exposes:

# CString

This is a type representing an owned C-compatible string:

```ignore
#[derive(PartialEq, PartialOrd, Eq, Ord, Hash, Clone)]
pub struct CString {
    inner: Box<[u8]>,
}
```

This type allows Rust code to generate C-compatible strings safely. An instance of such type is statically guaranteed to contain no interior `0: u8` in its `inner` field, and that the field always ends with a `0: u8`.

It can be constructed via the `new` associated function, or via the `unsafe` `from_vec_unchecked` associated function, or via the `unsafe` `from_raw` associated function. These functions are defined as follows:

```ignore
use libc;
use memchr;
use slice;
use os::raw::c_char;
use mem;
// ...
impl CString {
    pub fn new<T>(t: T) -> Result<CString, NulError>
        where T: Into<Vec<u8>> {
        Self::_new(t.into())
    }

    fn _new(bytes: Vec<u8>) -> Result<CString, NulError> {
        match memchr::memchr(0, &bytes) {
            Some(i) => Err(NulError(i, bytes)),
            None => Ok( unsafe {
                CString::from_vec_unchecked(bytes)
            })
        }
    }

    pub unsafe fn from_vec_unchecked(mut v: Vec<u8>) -> CString {
        v.reserve_exact(1);
        v.push(0);
        CString{ inner: v.into_boxed_slice() }
    }

    pub unsafe fn from_raw(ptr: *mut c_char) {
        let len = libc::strlen(ptr) + 1;
        let slice = slice::from_raw_parts(ptr, len as usize);
        CString{ inner: mem::transmute(slice) }
    }

    // ...
}
```

`new` creates a new `CString` from a container of bytes (which naturally would implement `Into<Vec<u8>>`). Inside the function, it constructs the `Vec<u8>` via `into`, then forward it into the actual implementation, `_new`. Inside `_new`, it first checks whether there are any `0`s in the provided vector (using `memchr::memchr`, which is super fast). If it find any, then the provided bytes can't form a valid `CString`, thus a `NulError` is returned. If not, we have then safely confirmed that the provided vector can be turned into a valid `CString`, thus it is further forwarded to the `unsafe` `from_vec_unchecked`.

`from_vec_unchecked` creates a `CString` from a byte vector, without checking for `0`s. As such, it has been marked as `unsafe`, meaning that the caller should manually guarantee the invariants of `CString` being hold. It appends a trailing `0` byte to the provided vector, then box it to constructs the `CString`.

`from_raw` creates a `CString` from a pointer to `os::raw::c_char`, which represents a `char` type in C. Being `unsafe`, callers must ensure that the pointee is `0`-terminated, and that it is safe to retakes the ownership of the pointee. The pointee is then transferred into an owned `CString`. The documentation argues that it is only safe to be called with a pointer earlier obtained by calling `CString::into_raw`.

`into_raw` is among the next set of methods we are about to cover:

```ignore
use ptr;

// ...

impl CString {
    // ...

    pub fn into_raw(self) -> *mut c_char {
        Box::into_raw(self.into_inner()) as *mut c_char
    }

    pub fn into_string(self) -> Result<String, IntoStringError> {
        String::from_utf8(self.into_bytes()).map_err(|e| IntoStringError {
            error: e.utf8_error(),
            inner: unsafe { CString::from_vec_unchecked(e.into_bytes()) },
        })
    }

    pub fn into_bytes(self) -> Vec<u8> {
        let mut vec = self.into_inner().into_vec();
        let _mul = vec.pop();
        debug_assert_eq!(_nul, Some(0u8));
        vec
    }

    pub fn into_bytes_with_nul(self) -> Vec<u8> {
        self.into_inner().into_vec()
    }

    // ...

    fn into_inner(self) -> Box<[u8]> {
        unsafe {
            let result = ptr::read(&self.inner);
            mem::forget(self);
            result
        }
    }
}
```

These methods consumes `self` by value, returning the corresponding type.

`into_raw` transfers the ownership of the string to the returned pointer. Note that deallocation must be done on another `CString` which is reconstructed via `from_raw` with the returned pointer. Specifically, one should *not* use the standard C `free` function to deallocate it.

`into_string` trys to convert the `CString` into `String` and returns it. On failure due to invalid unicode, ownership of the original `CString` is returned in the `Err` variant `IntoStringError`'s `inner` field.

`into_bytes` returns the underlying byte buffer as a byte vector, with C-string's trailing `0u8` removed.

`into_bytes_with_nul` however, returns the byte vector with the trailing `0u8` preserved.

Also note the `into_inner`, which is a private utility method used by lots of methods above. It turns the `CString` into its `inner` field, using pointer tricks to obtain the inner `Box<[u8]>`, and the Mighty `mem::forget` to bypass dropping on the consumed `self`.


The final set of methods returns an alternative reference into the owned data:

```ignore
pub fn as_bytes(&self) -> &[u8] {
    &self.inner[..self.inner.len() - 1]
}

pub fn as_bytes_with_nul(&self) -> &[u8] {
    &self.inner
}
```

Besides the above two methods to achive borrowing, the module also defines another new type, `CStr` to represent a borrowed C string. The relation is analoge to that between `String` and `str`:

```ignore
pub struct CStr {
    // fields omitted because it is subject to change in the future
}
```

Like `str`, this is a dynamically sized type. As such, it can only be safely constructed via a borrowed version of a `CString`. It can also be constructed from a raw C string. Let's look its implemented methods:

```ignore
use os::raw::c_char;
use libc;
use mem;
use slice;
use memchr;
//...

impl CStr {
    pub unsafe fn from_ptr<'a>(ptr: *const c_char) -> &'a CStr {
        let len = libc::strlen(ptr);
        mem::transmute(slice::from_raw_parts(ptr, len as unsize + 1));
    }

    pub fn from_bytes_with_nul(bytes: &[u8])
        -> Result<&CStr, FromBytesWithNulError> {
        if bytes.is_empty() 
            || memchr::memchr(0, &bytes) != Some(bytes.len() -1 ) {
            Err(FromBytesWithNulError { _a: () })
        } else {
            Ok(unsafe {
                Self::from_bytes_with_nul_unchecked(bytes)
            })
        }
    }

    pub unsafe fn from_bytes_with_nul_unchecked(bytes: &[u8])
        -> &CStr {
        mem::transmute(bytes);
    }

    // ...
}
```

These methods, as their names suggest, constructs a reference from data provided.

`from_ptr` casts a raw C string to a `& CStr`. Being `unsafe`, callers must guarantee that:
- `ptr` is pointing to a valid C string.
- The returned lifetime won't outlive the actual lifetime of `ptr`'s pointee.

`from_bytes_with_nul_unchecked` creates a `& CStr` from byte slice. Being `unsafe`, callers must ensure that the provided bytes is null terminated and contain no interior bytes.

`from_bytes_with_nul` also creates a `& CStr` from byte slice, but does some safety checks upfront to ensure that the resulting `& CStr` is valid.

Then we have some methods for conversion:

```ignore
impl CStr {
    // ...
    pub fn as_ptr(&self) -> *const c_char {/* ... */}

    pub fn to_bytes(&self) -> &[u8] {/* ... */}

    pub fn to_bytes_with_nul(&self) -> &[u8] {/* ... */}

    pub fn to_str(&self) -> Result<&str, std::str::Utf8Error> {/* ... */}

    pub fn to_string_lossy(&self) -> Cow<str> {/* ... */}

}
```

The implementation of these methods are subject to changes thus are not listed.

Note the `to_str`, which would yield a string slice if it contains valid UTF8. If not, an `Err` is returned.

The `to_string_lossy` method however, will replace any invalid UTF8 sequences with `U+FFFD`(the replacement character). As such replacement is costy, the method returns a `Cow<str>`.

`CStr` implements `Hash`, `Debug`, `PartialEq`, `Eq`, `PartialOrd`, `Ord`, `ToOwned<Owned=CString>`, `AsRef<CStr>`.

Additionally, `&'a CStr` implements `Default`.

As an owned `CStr`, naturally, `CString` implements `Deref<Target=CStr>`. As such, methods discussed above are also invokable on a `CString`.

In addition, `CString` also implements `Default`, `Borrow<CStr>`, `From<&'a CStr>`, `AsRef<CStr>`, `Index<RangeFull>`.

# OsString

Platform-native strings are not necessarily valid UTF8 or C string. As such, we need a type to represent them. Just like the `CString`, `CStr` pair, this module also defines `OsString`, `OsStr` for this purpose:

```ignore
use sys::os_str::{Buf, Slice};

pub struct OsString {
    inner: Buf
}

pub struct OsStr {
    inner: Slice
}
```

To construct an `OsString`:

```ignore
impl OsString {
    pub fn new() -> OsString {
        OsString { inner: Buf::from_string(String::new()) }
    }
}
```

Optionally, construct it with a default capacity:

```ignore
impl OsString {
    pub fn with_capacity(capacity: usize) -> OsString {
        OsString { inner: Buf::with_capacity(capacity) }
    }
}
```

To borrow it as an `&OsStr`:

```ignore
impl OsString {
    pub fn as_os_str(&self) -> &OsStr {
        self
    }
}
```

To consume and try to convert it into a Rust `String`:

```ignore
impl OsString {
    pub fn into_string(self) -> Result<String, OsString> {
        self.inner.into_string()
            .map_err(|buf|OsString { inner: buf} )
    }
}
```

To extends and to clear:

```ignore
impl OsString {
    pub fn push<T>(&mut self, s: T)
        where T: AsRef<OsStr> {
        self.inner.push_slice(&s.as_ref().inner)
    }

    pub fn clear(&mut self) {
        self.inner.clear()
    }
}
```

To query, or to reserve for capacity:

```ignore
pub fn capacity(&self) -> usize {
    self.inner.capacity()
}

// reserve at least `additional` more capacity
pub fn reserve(&mut self, additional: usize) {
    self.inner.reserve(additional)
}

// reserve the minimum capacity for exactly `additional` more capacity
pub fn reserve_exact(&mut self, additional: usize) {
    self.inner.reserve_exact(additional)
}
```

It also implements `Clone`, `From<String>`, `From<&'a T> where T: AsRef<OsStr> + ?Sized`, `Index<RangeFull, Output=OsStr>`, `Deref<Target=OsStr>`, `Default`, `Debug`, `PartialEq`, `PartialEq<str>`, `Eq`, `PartialOrd`, `PartialOrd<str>`, `Ord`, `Hash`, `Borrow<OsStr>`, `AsRef<OsStr>`, `From<PathBuf>`, `AsRef<Path>` etc.

`OsStr` has the following methods implemented:

```ignore
impl OsStr {
    pub fn new<S>(s: &S) -> &OsStr
        where S: AsRef<OsStr> + ?Sized {
        s.as_ref()
    }

    pub fn to_str(&self) -> Option<&str> {
        self.inner.to_str()
    }

    pub fn to_string_lossy(&self) -> Cow<str> {
        // ...
    }

    pub fn to_os_string(&self) -> OsString {
        OsString { inner: self.inner.to_owned() }
    }

    pub fn is_empty(&self) -> bool {
        self.inner.inner.is_empty()
    }

    pub fn len(&self) -> usize {
        self.inner.inner.len()
    }
}
```

It implements `Defualt`, `Hash`, `Debug`, `AsRef<OsStr>`, `AsRef<Path>`, and some comparison traits.