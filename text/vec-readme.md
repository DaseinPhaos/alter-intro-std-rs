% std::vec

This module contains definition for the continuous growable array type, `Vec<T>`:

```ignore
pub struct Vec<T> {
    buf: RawVec<T>,
    len: usize,
}

struct RawVec<T> {
    ptr: Unique<T>, // std::ptr::Unique<T>
    cap: usize,
}
```

Note that it internally makes use of another struct `RawVec<T>` to handle the memalloc-related issues.

The most convenient way to create a vector is through the crate level utility macro, `vec!`:

```rust
let mut vec = vec![1, 2, 3];
vec.push(4);

assert_eq!(vec, [1, 2, 3, 4]);
```

```rust
let vec = vec![2; 3];

assert_eq!(vec, [2, 2, 2]);
```

Alternatively, one can construct vectors with

```ignore
impl<T> Vec<T> {
    pub fn new() -> Vec<T> {
        Vec {
            buf: RawVec::new(),
            len: 0,
        }
    }

    pub fn with_capacity(capacity: usize) -> Vec<T> {
        Vec {
            buf: RawVec::with_capacity(capacity),
            len: 0,
        }
    }

    pub unsafe fn from_raw_parts(ptr: *mut T, length: usize, capacity: usize) -> Vec<T> {
        Vec {
            buf: RawVec::from_raw_parts(ptr, capacity),
            len: length,
        }
    }
}

impl<T> RawVec<T> {
    pub fn new() -> RawVec<T> {
        unsafe {
            let cap = if mem::size_of::<T>() == 0 { !0 } else { 0 };
            RawVec {
                ptr: Unique::new(heap::EMPTY as *mut T),
                cap: cap,
            }
        }
    }

    pub fn with_capacity(cap: usize) -> RawVec<T> {
        unsafe {
            let elem_size = mem::size_of::<T>();

            let alloc_size = cap.checked_mul(elem_size).expect("capacity overflow");

            alloc_guard(alloc_size); // guard against allocation overflow

            let ptr = if alloc_size == 0 {
                heap::EMPTY as *mut u8
            } else {
                let align = mem::align_of::<T>();
                let ptr = heap::allocate(alloc_size, align);
                if ptr.is_null() {
                    oom() // out of memory panic procedure
                }
                ptr
            };

            RawVec {
                ptr: Unique::new(ptr as *mut _),
                cap: cap,
            }
        }
    }

    pub unsafe fn from_raw_parts(ptr: *mut T, cap: usize) -> Self {
        RawVec {
            ptr: Unique::new(ptr),
            cap: cap,
        }
    }
}
```

To query the current capacity, i.e. the number of the elements the vector can hold without reallocation:

```ignore
impl<T> Vec<T> {
    pub fn capacity(&self) -> usize {
        self.buf.cap()
    }
}

impl<T> RawVec<T> {
    pub fn cap(&self) -> usize {
        if mem::size_of::<T>() == 0 {
            !0
        } else {
            self.cap
        }
    }
}
```

To query the length of the vector:

```ignore
impl<T> Vec<T> {
    pub fn len(&self) -> usize {
        self.len
    }

    pub fn is_empty(&self) -> bool {
        self.len() == 0
    }
}
```

To reserve capacity for at least `additional` more elements to be inserted:

```ignore
impl<T> Vec<T> {
    pub fn reserve(&mut self, additional: usize) {
        self.buf.reserve(self.len, additional);
    }

    pub fn reserve_exact(&mut self, additional: usize) {
        self.buf.reserve_exact(self.len, additional);
    }
}

impl<T> RawVec<T> {
    pub fn reserve_exact(&mut self, used_cap: usize, extra_cap: usize) {
        unsafe {
            let elem_size = mem::size_of::<T>();
            let align = mem::align_of::<T>();

            if self.cap().wrapping_sub(used_cap) >= extra_cap {
                return;
            }

            let new_cap = used_cap.checked_add(extra_cap).expect("capacity overflow");
            let new_alloc_size = new_cap.checked_mul(elem_size).expect("capacity overflow");

            alloc_guard(new_alloc_size);
            let ptr = if self.cap == 0 {
                heap::allocate(new_alloc_size, align)
            } else {
                heap::reallocate(self.ptr() as *mut _, self.cap * elem_size, new_alloc_size, align)
            };

            if ptr.is_null() {
                oom()
            }

            self.ptr = Unique::new(ptr as *mut _);
            self.cap = new_cap;
        }
    }

    pub fn reserve(&mut self, used_cap: usize, extra_cap: usize) {
        unsafe {
            let elem_size = mem::size_of::<T>();
            let align = mem::align_of::<T>();

            if self.cap().wrapping_sub(used_cap) >= needed_extra_cap {
                return;
            }

            // `amortized_new_size` guarantees expotential growth of the actual capacity
            let (new_cap, new_alloc_size) = self.amortized_new_size(used_cap, extra_cap);

            alloc_guard(new_alloc_size);

            let ptr = if self.cap == 0 {
                heap::allocate(new_alloc_size, align)
            } else {
                heap::reallocate(self.ptr() as *mut _, self.cap * elem_size, new_alloc_size, align)
            };

            if ptr.is_null() {
                oom()
            }

            self.ptr = Unique::new(ptr as *mut _);
            self.cap = new_cap;
        }
    }
}
```

To shrink the capacity of the vector as much as possible:

```ignore
impl<T> Vec<T> {
    pub fn shrink_to_fit(&mut self) {
        self.buf.shrink_to_fit(self.len);
    }
}

impl<T> RawVec<T> {
    pub fn shrink_to_fit(&mut self, amount: usize) {
        let elem_size = mem::size_of::<T>();
        let align = mem::align_of::<T>();

        if elem_size == 0 {
            self.cap = amount;
            return;
        }

        assert!(self.cap >= amount, "Tried to shrink to a larger capacity");

        if amout == 0 {
            mem::replace(self, RawVec::new());
        } else if self.cap != amount {
            unsafe {
                let ptr = heap::reallocate(self.ptr() as *mut _, self.cap * elem_size, amount * elem_size, align);

                if ptr.is_null() {
                    oom()
                }
                self.ptr = Unique::new(ptr as *mut _);
            }
            self.cap = amount;
        }
    }
}
```

To convert the vector into a boxed slice:

```ignore
impl<T> Vec<T> {
    pub fn into_boxed_slice(mut self) -> Box<[T]> {
        unsafe {
            self.shrink_to_fit();
            let buf = ptr::read(&self.buf);
            mem::forget(self);
            buf.into_box()
        }
    }
}

impl<T> Into<Box<[T]>> for Vec<T> {
    fn into(self) -> Box<[T]> {
        self.into_boxed_slice()
    }
}
```

To truncate the vector at `len`, dropping succeed values (if any):

```ignore
impl<T> Vec<T> {
    pub fn truncate(&mut self, len: usize) {
        unsafe {
            while len < self.len {
                self.len -= 1;
                let last = self.len;
                ptr::drop_in_place(self.get_unchecked_mut(last));
            }
        }
    }
}
```

To clear the vector, dropping all elements:

```ignore
pub fn clear(&mut self) {
    self.truncate(0)
}
```

To get a sliced view of the vector:

```ignore
impl<T> ops::Deref for Vec<T> {
    type Target = [T];

    fn deref(&self) -> &[T] {
        unsafe {
            let p = self.buf.ptr();
            assume(!p.is_null()); // intrinsic that informs the compiler that the passed condition is always true
            slice::from_raw_parts(p, self.len)
        }
    }
}

impl<T> ops::DerefMut for Vec<T> {
    fn deref_mut(&mut self) -> &mut [T] {
        unsafe {
            let p = self.buf.ptr();
            assume(!p.is_null());
            slice::from_raw_parts_mut(p, self.len)
        }
    }
}

impl<T> Vec<T> {
    pub fn as_slice(&self) -> &[T] {
        self
    }

    pub fn as_mut_slice(&mut self) -> &mut [T] {
        self
    }
}
```

To set the length of a vector, without actually modifying the buffer (thus `unsafe`, the caller must ensure the validity of the new vector):

```ignore
impl<T> Vec<T> {
    pub unsafe fn set_len(&mut self, len: usize) {
        self.len = len;
    }
}
```



To insert at `index`, O(n):

```ignore
impl<T> Vec<T> {
    pub fn insert(&mut self, index: usize, element: T) {
        let len = self.len();
        assert!(index <= len);

        if len == self.buf.cap() {
            self.buffer.double(); // reallocate the capacity to double
        }

        unsafe {
            {
                let p = self.as_mut_ptr().offset(index as isize);

                ptr::copy(p, p.offset(1), len - index);

                ptr::write(p, element);
            }
            self.set_len(len + 1);
        }
    }
}
```

To remove at `index` and return the removed value, O(n):

```ignore
impl<T> Vec<T> {
    pub fn remove(&mut self, index: usize) -> T {
        let len = self.len();
        assert!(index < len);
        unsafe {
            let ret;
            {
                let p = self.as_mut_ptr().offset(index as isize);

                ret = ptr::read(p);

                ptr::copy(p.offset(1), p, len - index - 1);
            }

            self.set_len(len - 1);
            ret
        }
    }
}
```

To push an element into the back of the vector, amortized O(1):

```ignore
impl<T> Vec<T> {
    pub fn push(&mut self, value: T) {
        if self.len == self.buf.cap() {
            self.buf.double();
        }

        unsafe {
            let end = self.as_mut_ptr().offset(self.len as isize);
            ptr::write(end, value);
            self.len += 1;
        }
    }
}
```

To move all the elements of `other` into the vector, leaving `other` empty:

```ignore
impl<T> Vec<T> {
    pub fn append(&mut self, other: &mut Self) {
        self.reserve(other.len());
        let len = self.len();
        unsafe {
            ptr::copy_non_overlapping(other.as_ptr(), self.get_unchecked_mut(len), other.len());
            self.len += other.len();
            other.set_len(0);
        }
    }
}
```

To remove and return the last element (if any), O(1):

```ignore
impl<T> Vec<T> {
    pub fn pop(&mut self) -> Option<T> {
        if self.len == 0 {
            None
        } else {
            unsafe {
                self.len -= 1;
                Some(ptr::read(self.get_unchecked(self.len)))
            }
        }
    }
}
```

To remove at `index` and return the removed value, swapping the content at `index` with the last element, O(1):

```ignore
impl<T> Vec<T> {
    pub fn swap_remove(&mut self, index: usize) -> T {
        let len = self.len();
        self.swap(index, len - 1);
        self.pop().unwrap()
    }
}
```

To retain the elements that meet the predicate (note that the retained elements' order is preserved), O(n):

```ignore
impl<T> Vec<T> {
    pub fn retain<F>(&mut self, f: F)
        where F: FnMut(&T) -> bool
    {
        let len = self.len();
        let mut del = 0;
        {
            let v = &mut **self; // v : &mut [T]
            for i in 0..len {
                if !f(&v[i]) {
                    del += 1;
                } else if del > 0 {
                    v.swap(i - del, i)
                }
            }
        }
        if del > 0 {
            self.truncate(len - del);
        }
    }
}
```

To remove consecutive elements according to some sort of criteria:

```ignore
impl<T> Vec<T> {
    pub fn dedup_by<F>(&mut self, same_bucket: F)
        where F: FnMut(&mut T, &mut T) -> bool
    {
        unsafe {
            let len = self.len();
            if len <= 1 {
                return;
            }

            let p = self.as_mut_ptr();
            let mut r = 1usize; // next index to read against
            let mut w = 1usize; // next index to swap the next unique element into (if needed)

            while r < len {
                let p_r = p.offset(r as isize);
                let p_bw = p.offset((w - 1) as isize); // pointing to the last guaranteed unique element

                if !same_bucket(&mut *p_r, &mut p_bw) {
                    // New unique element encountered at `p_r`.
                    // We need to make sure that it get swapped to `p_w`.

                    if r != w {
                        let p_w = p_bw.offset(1);
                        ptr::swap(p_r, p_w);
                    }

                    // Swapping complete, advance the writing index
                    w += 1;
                }

                // One round of checking complete, advance the reading index
                r += 1;
            }

            // Checking process complete, elements beginning at index `w` are
            // all duplicated elements. Truncating them out.
            self.truncate(w);
        }
    }

    pub fn dedup_by_key<F, K>(&mut self, key: F)
        where F: FnMut(&mut T) -> K,
              K: PartialEq<K>,
    {
        self.dedup_by(|a, b| key(a) == key(b))
    }
}

impl<T: Clone> Vec<T> {
    pub fn dedup(&mut self) {
        self.dedup_by(|a, b| a == b)
    }
}
```

To split the vector into two at the given index, after the invocation of which `self` contains elements of indice `0..at` and the returned value contains `at..len`:

```ignore
impl<T> Vec<T> {
    pub fn split_off(&mut self, at: usize) -> Vec<T> {
        assert!(at <= self.len(), "`at` out of bounds");

        let other_len = self.len - at;
        let mut other = Vec::with_capacity(other_len);

        unsafe {
            self.set_len(at);
            other.set_len(other_len);

            ptr::copy_nonoverlapping(self.ast_ptr().offset(at as isize), other.as_mut_ptr(), other.len());
        }
        other
    }
}
```

For `T: Clone`, additional methods are provided:

```ignore
impl<T: Clone> Vec<T> {
    pub fn resize(&mut self, new_len: usize, value: T) {
        let len = self.len();
        if new_len > len {
            self.extend_with_element(new_len - len, value); // helper method that extends the vector with `n` clones of `value`.
        } else {
            self.truncate(new_len);
        }
    }

    /// Extends the vector with clones of all elements in `other`.
    pub fn extend_from_slice(&mut self, other: &[T]) {
        /*...*/
    }
}

impl<T: Clone> Clone for Vec<T> {/*...*/}
```

Finally, when a vector goes out of scope:

```ignore
unsafe impl<#[may_dangle] T> Drop for Vec<T> {
    fn drop(&mut self) {
        unsafe {
            // use drop for [T]
            ptr::drop_in_place(&mut self[..])
        }
        // `RawVec<T>` handles memory deallocation
    }
}

unsafe impl<#[may_dangle] T> Drop for RawVec<T> {
    fn drop(&mut self) {
        let elem_size = mem::size_of::<T>();
        if elem_size != 0 && self.cap != 0 {
            let align = mem::align_of::<T>();
            let num_bytes = elem_size * self.cap;
            unsafe {
                heap::deallcate(*self.ptr as *mut _, num_bytes, align);
            }
        }
    }
}
```

Also notice that `Borrow<[T]>`, `BorrowMut<[T]>`, `AsRef<Vec<T>>`, `AsRef<[T]>`, `AsMut<Vec<T>>`, `AsMut<[T]>`, `Default`, `From<BinaryHeap<T>>`, `From<VecDeque<T>>`, `From<&'a [T]>`, `From<Cow<'a, [T]>>`, `From<Box<[T]>>`, `FromIterator<T>` are implemented for `Vec<T>`.

Also notice that `Index<usize>`, `Index<Range<usize>>`, `Index<RangeTo<usize>>`, `Index<RangeFrom<usize>>`, `Index<RangeFull>` and their `IndexMut` siblings are all implemented for `Vec<T>`.

Also notice that `Debug`, `Clone`, `Hash`, `Eq`and`Ord`-related traits are implemented for `Vec<T>` if `T` implements those traits respectively.

Also notice that `From<CString>`, `From<String>` and `From<&'a str>` are implemented for `Vec<u8>`.
