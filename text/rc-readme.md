% std::rc

This module contains types for single-threaded reference-counting.

Two types defined here, `Rc<T>`, providing shared ownership of a value of `T` on heap, and `Weak`, providing non-owning pointing to a value of `T` on heap; act in corporate with one another.

Let's take a look at the reference counting mechanism's overall design:

First of all, for the actual value on heap, we have:

```ignore
struct RcBox<T: ?Sized> {
    strong: Cell<usize>,
    weak: Cell<usize>,
    value: T,
}
```

`Rc` and `Weak` by themselves are just pointers to such a struct:

```ignore
pub struct Rc<T: ?Sized> {
    ptr: Shared<RcBox<T>>, // `Shared` represents a pointer pointing to a shared value
}

pub struct Weak<T: ?Sized> {
    ptr: Shared<RcBox<T>>,
}
```

Now to construct a new instance of `T` on heap and obtain a `Rc<T>` pointing at it:

```ignore
impl<T> Rc<T> {
    pub fn new(value: T) -> Rc<T> {
        unsafe {
            Rc {
                ptr: Shared::new(Box::into_raw(
                    box RcBox {
                        strong: Cell::new(1),
                        weak: Cell::new(1), // All strong pointers shares an implicit weak pointer
                        value: value,
                    }
                ))
            }
        }
    }
}
```

To obtain a `Weak` pointer of this instance:

```ignore
impl<T: ?Sized> Rc<T> {
    pub fn downgrade(this: &Self) -> Weak<T> {
        this.inc_weak(); // safely increment the pointed `RcBox`'s `weak` count by 1
        Weak { ptr: this.ptr }
    }
}
```

If possible, a `Weak` pointer can also be `upgrade`d to a strong one:

```ignore
impl<T: ?Sized> Weak<T> {
    pub fn upgrade(&self) -> Option<Rc<T>> {
        if self.strong() == 0 { None } // already dropped
        else {
            self.inc_strong();
            Some(Rc { ptr: self.ptr })
        }
    }
}
```

To obtain a new `Rc` pointing to this instance:

```ignore
impl<T: ?Sized> Clone for Rc<T> {
    fn clone(&self) -> Rc<T> {
        self.inc_strong(); // safely increment the pointed `RcBox`'s `strong` count by 1
        Rc { ptr: self.ptr }
    }
}
```

To compare if two `Rc` point to the same underlying value:

```ignore
impl<T: ?Sized> Rc<T> {
    pub fn ptr_eq(this: &Self, other: &Self) -> bool {
        let this_ptr: *const RcBox<T> = *this.ptr;
        let other_ptr: *const RcBox<T> = *other.ptr;
        this_ptr == other_ptr
    }
}
```

To obtain a new `Weak` pointer from another `Weak` pointer:

```ignore
impl<T: ?Sized> Clone for Weak<T> {
    self.inc_weak();
    Weak { ptr: self.ptr }
}
```

When a `Weak` pointer goes out of scope:

```ignore
impl<T: ?Sized> Drop for Weak<T> {
    fn drop(&mut self) {
        unsafe {
            let ptr = *self.ptr;

            self.dec_weak();
            if self.weak() == 0 {
                deallocate(ptr as *mut u8, size_of_val(&*ptr), align_of_val(&*ptr));
            }
        }
    }
}
```

To get the current counting of the instance:

```ignore
impl<T: ?Sized> Rc<T> {
    pub fn weak_count(this: &Self) -> usize {
        this.weak() - 1 // subtract out the implicit weak pointer introduced by strong pointers
    }

    pub fn strong_count(this: &Self) -> usize {
        this.strong()
    }
}
```

When an instance of `Rc` goes out of scope:

```ignore
unsafe impl<T: ?Sized> Drop for Rc<T> {
    fn drop(&mut self) {
        unsafe {
            let ptr = self.ptr.as_muut_ptr();
            self.dec_strong();
            if self.strong() == 0 {
                // if strong count of the underlying type decrements to zero,
                // we drop its content and remove te implicit weak pointer
                ptr::drop_in_place(&mut (*ptr).value);
                self.dec_weak();

                if self.weak() == 0 {
                    // if after the decrement weak count also goes to 0,
                    // we deallocate the `RcBox` on heap
                    deallocate(ptr as *mut u8, size_of_val(&*ptr), align_of_val(&*ptr));
                }
            }
        }
    }
}
```

To access the underlying instance, we can either `deref` it to get a reference, or `try_unwrap` it to get the value:

```ignore
impl <T: ?Sized> Deref for Rc<T> {
    type Target = T;

    fn deref(&self) -> &T {
        &self.inner().value
    }
}

impl<T> Rc<T> {
    pub fn try_unwrap(this: Self) -> Result<T, Self> {
        if Rc::strong_count(&this) == 1 
        unsafe {
            let val = ptr::read(&*this); // copy the value
            this.dec_strong(); // remove the last strong count
            let _weak = Weak { ptr: this.ptr }; // create a fake weak pointer, let its dtor handle the removal of the implicit weak count
            forget(this); // disable this dtor from working, as we've already handled what's need to be done
            Ok(val)
        }
    } else {
        Err(this) // can't unwrap it because other strong pointers exist. Return this `Rc` to the caller
    }
}
```

Notice that `Rc` only have `Deref`. It doesn't implement `DerefMut`. To mutate the content of the underlying instance, one must introduce interior mutability. However, when a mutable reference of an `Rc` is obtainable, one actually *can* try to get a mutable reference of the inner value. This is the case because, if we can obtain a mutable reference of the handle, we statically ensures that we are the only one that can modify the underlying value. No new pointers to that value can get spawned as long as we hold that mutable reference:

```ignore
impl<T: ?Sized> Rc<T> {
    pub fn get_mut(this: &mut Rc<T>) -> Option<&mut T> {
        if Rc::weak_count(this) == 0 && Rc::strong_count(this) == 1 {
            // `this` is the only pointer to the underlying value
            let inner = unsafe { &mut *this.ptr.as_mut_ptr() };
            Some(&mut inner.value)
        } else {
            None
        }
    }
}
```

Alternatively, if we want to obtain a mutable reference anyway, no matter whether the if there is only one pointer pointing to the underlying value or not, you can call

```ignore
impl<T: Clone> Rc<T> {
    pub fn make_mt(this: &mut Rc<T>) -> &mut T {
        if Rc::strong_count(this) != 1 {
            // other strong pointers exists,
            // we have to make a new instance of `T` on heap
            *this = Rc::new(**this).clone());
        } else if Rc::weak_count(this) != 0 {
            // this is the only strong pointer,
            // but there exist other weak pointers
            // we avoid the clone, and make these weak yatsu pointing to void
            unsafe {
                let mut swap = Rc::new(ptr::read(&(**this.ptr).value));
                mem::swap(this, &mut swap);
                swap.dec_strong();
                swap.dec_weak();
                forget(swap);
            }
        }
        
        let inner = unsafe { &mut *this.ptr.as_mut_ptr() };
        &mut inner.value
    }
}
```

To convert into and from a raw pointer:

```ignore
impl<T> Rc<T> {
    pub fn into_raw(this: Self) -> *const T {
        let ptr = unsafe { &mut (*this.ptr.as_mut_ptr()).value as *const _ };
        mem::forget(this);
        ptr
    }

    pub unsafe fn from_raw(ptr: *const T) -> Self {
        Rc { ptr: Shared::new((ptr as *const u8).offset(-offset_of!(RcBox<T>, value)) as *const _) }
    }
}
```

Alternatively, one can construct a weak pointer pointing at nothing by

```ignore
impl<T> Weak<T> {
    pub fn new() -> Weak<T> {
        unsafe {
            Weak {
                ptr: Shared::new(Box::into_raw(
                    box RcBox {
                        strong: Cell::new(0),
                        weak: Cell::new(1),
                        value: uninitialized(),
                    }
                ))
            }
        }
    }
}
```


Notice that almost all methods operating on `Rc` are not "normal" methods, just associate functions. This is the case so that they won't shadow method names of the underlying `T`. However, some trait-impls do shadow its underly type's implementation. These notably include `Borrow`, `Clone`, `AsRef`, `Deref` for `Rc`, and `Clone` for `Weak`. Also notice that `From<T>`, `Default`, `Hash`, `Ord` and some other commonly used traits are implemented for `Rc` (some of which only exists if the underlying type implemented them as well). `Default` is also implemented for `Weak`, creating a pointer pointing at nowhere.

Finally, note that marker `!Send` is implemented for `Rc<T>` and `Weak<T>`. This is because the underlying `RcBox` uses non-atomic types for reference-counting, which is efficient for single-thread usage, but not safe to be accessed across threads.
