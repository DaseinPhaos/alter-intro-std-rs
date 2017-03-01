% std::iter

> https://doc.rust-lang.org/std/iter/

This module introduces iteration into Rust.

# Introduction

The heart and soul of this module is the `Iterator` trait:

```ignore
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

The required associated type, `Item`, represents the type of elements that this iterator is to iterate over.

The required `next` method defines the actual mechanism of iteration. Every invocation of it means another round of iteration. It returns an `Option<Item>`. If the returned value is a `Some`, that element is the result of this round of iteration. Returning a `None`, on the other hand, means that the iterator has finished this round of iteration. Due to the actual implementation, the iterator is permitted to resume iteration after such `None` is returned. Subsequent calls to `next` *can* start returning `Some` again at some point. How to evaluate the returned value is up to the implementation itself. However, the general philosophy of iterator implementation is to apply some form of "lazy evaluation", that is, we don't perform any evaluation unless we have to.

The module also defines some other more specific kinds of iterator-trait on top of this trait. This trait itself also has a bunch of default methods for utility usage. Lots of these methods deal with iterator composition, the module also contains corresponding generic struct definitions for composited iterators. We will come to these definitions later in this post.

For now let's just check out how to make a simple iterator:

```rust
struct EvenNums {
    i: u32,
}

impl EvenNums {
    fn new() -> EvenNums {
        Counter { i: 0 }
    }
}

impl Iterator for EvenNums {
    type Item = u32;
    fn next(&mut self) -> Option<usize> {
        let ret = 2 * self.i;
        self.i += 1;
        Some(ret)
    }
}

let mut evenNums = EvenNums::new();
assert_eq!(evenNums.next().unwrap(), 0);
assert_eq!(evenNums.next().unwrap(), 2);
assert_eq!(evenNums.next().unwrap(), 4);
```

## For Loops and IntoIterator

Another core trait is

```ignore
pub trait IntoIterator {
    type Item;
    type IntoIter: Iterator<Item=Self::Item>;

    fn into_iter(self) -> Self::IntoIter;
}
```

The trait represents a value type that can be consumed into a iterator. The type of the iterator it converts into is captured by the associated type `IntoIter`. The type of item the converted iterator iterates over is captured by type `Item`. The `into_iter` method consumes the caller and returns the converted `IntoIter`.

Any struct implementing this trait can be consumed by a `for` statement. `Vec<T>` implements this trait, that is the reason why we can write:

```rust
let values = vec![1, 2, 3];

for x in values {
    println!("{}", x);
}
```
The `for` statement will be desugared to something equivalent as

```rust
# let values = vec![1, 2, 3];

let iter = values.into_iter();
while let Some(x) = iter.next() {
    println!("{}", x);
}
```

# Iterator

Now let's dig deeper into the provided methods of `Iterator`.

---

```ignore
pub trait Iterator {
    #[inline]
    fn size_hint(&self) -> (usize, Option<usize>) { (0, None) }
}
```

This method returns the bounds on the remaining items' size. The first element is the lower bound, the second is the upper bound, with `None` meaning either the bound is unknown or beyond the range of `usize`. The default implementation simply returns `(0, None)`, which is of course valid but of not much use. Explicit implementation can choose to return a more correct estimation.

The result acquired from this method, as its name suggests, should be used as a hint for optimizations such as reserving space ahead. It must not be trusted though.

---

```ignore
pub trait Iterator {
    #[inline]
    fn count(self) -> usize where Self: Sized {
        self.fold(0, |count, _| count + 1)
    }
}
```

This method consumes the iterator and tries to count the number of iterations it provides. Internally, it makes use of another trait method `fold`, which is more general.

---

```ignore
pub trait Iterator {
    fn fold<B, F>(self, init: B, mut f: F) -> B
        where Self: Sized,
              F: FnMut(B, Self::Item) -> B, {
        let mut accum = init;
        for x in self { // This is valid because `IntoIterator` is implemented for `T: Iterator`,
                        // with its `into_iter` simply returning `self`
            accum = f(accum, x);
        }
        accum
    }
}
```

`fold` consumes the iterator, and takes two additional arguments, an `init`ial value, and a closure `f`. The closure should represents an "accumulator", that is, a function which takes the a value of `init`ial's type, and another argument of the iterator's `Item` type, returning another value of type of `init`.

Internally, the method applies the closure over and over again until the iterator has been consumed, then returns the final accumulated result.

A quick demo:

```rust
let values = vec![1, 2, 3];
let sum = values.into_iter().fold(0, |sum, next| sum + next);
assert_eq!(sum, 6);
```

---

```ignore
pub trait Iterator {
    fn last(self) -> Option<Self::Item> where Self: Sized {
        let mut last = None;
        for x in self { last = Some(x); }
        last
    }
}
```

The method tries to consume the iterator and returns its last element.

---

```ignore
pub trait Iterator {
    fn nth(&mut self, mut n: usize) -> Option<Self::Item> {
        for x in self {
            if n == 0 { return Some(x) }
            n -= 1;
        }
        None
    }
}
```

The method tries to return the `n`th element of the iterator.

Note that `for` loop, which operates on a `self: &mut Self`. This is valid because the module also contains a generic implementation:

```ignore
impl<'a, I: Iterator + ?Sized> Iterator for &'a mut I {
    type Item = I::Item;
    fn next(&mut self) -> Option<I::Item> { (**self).next() }

    // preserves valid size hint
    fn size_hint(&self) -> (usize, Option<usize>) { (**self).size_hint() }

    // preserve redundant round of dereferencing
    fn nth(&mut self, n: usize) -> Option<Self::Item> {
        (**self).nth(n)
    }
}
```

Together with the generic implementation we mentioned above:

```ignore
impl<I> IntoInterator for I where I: Iterator {
    type Item = I::Item;
    type IntoIter = I;
    fn into_iter(self) -> I { self }
}
```

The `for` loop here is just identical as a call of `for` on `self: Self`.

---

```ignore
pub trait Iterator {
    #[inline]
    fn chain<U>(self, other: U) -> Chain<Self, U::IntoIter>
        where Self: Sized, U: IntoIterator<Item=Self::Item> {
        Chain{ a: self, b: other.into_iter(), state: ChainState::Both }
    }
}
```

This method consumes the iterator, and chains it with another iterator-can-be.

The returned `Chain` is one of the iterator adaptor defined by the module:

```ignore
pub struct Chain<A, B> {
    a: A,
    b: B,
    state: ChainState,
}

enum ChainState {
    Both, // both iterator are remaining
    Front, // only the first iterator remains
    Back, // only the later iterator remains
}

impl<A, B> Iterator for Chain<A, B>
    where A: Iterator,
          B: Iterator<Item=A::Item> {
    type Item = A::Item;

    #[inline]
    fn next(&mut self) -> Option<A::Item> {
        match self.state {
            ChainState::Both => match self.a.next() {
                Some(item) => Some(item),
                None => {
                    self.state = ChainState::Back;
                    self.b.next()
                }
            },
            ChainState::Front => self.a.next(),
            ChainState::Back => self.b.next(),
        }
    }
}

// ...
```

From the implementation it's not hard to infer how this "chain" works.

Also notice that the sturcture also implements `DoubleEndedIterator` if its subcomponents implement it. The same goes for `Clone` and `Debug`.

---

```ignore
pub trait Iterator {
    fn zip<U>(self, other: U) -> Zip<Self, U::IntoIter>
        where Self: Sized, U: IntoIterator {
        Zip::new(self, other.into_iter())
    }
}
```

This method consumes the calling iterator and "zips" it up with another.

The returned `Zip` type is defined as follows:

```ignore
pub struct Zip<A, B> {
    a: A,
    b: B,
    index: usize,
    len: usize,
}

impl<A, B> Iterator for Zip<A, B>
    where A: Iterator, B: Iterator {
    type Item = (A::Item, B::Item);

    #[inline]
    fn next(&mut self) -> Option<Self::Item> {
        ZupImpl::next(self)
    }

    #[inline]
    fn size_hint(&self) -> (usize, Option<usize>) {
        ZipImpl::size_hint(self)
    }
}

impl<A, B> DoubleEndedIterator for Zip<A, B> where
    A: DoubleEndedIterator + ExactSizeIterator,
    B: DoubleEndedIterator + ExactSizeIterator,
{
    #[inline]
    fn next_back(&mut self) -> Option<(A::Item, B::Item)> {
        ZipImpl::next_back(self)
    }
}
```

Note that the actual implementation is forwarded into another trait, `ZipImpl`. The trait makes use of a currently unstable feature, [impl-specialization](https://github.com/rust-lang/rfcs/blob/master/text/1210-impl-specialization.md), to improve the performance of `Zip` on certain types. We omit that implementation here, interested readers can look into it in the source code, or find more info at the [related PR](https://github.com/rust-lang/rust/pull/33090).

---

```ignore
pub trait Iterator {
    fn map<B, F>(self, f: F) -> Map<Self, F>
        where Self: Sized, F: FnMut(Self::Item) -> B {
        Map { iter: self, f: F }
    }
}
```

This method consumes the iterator, then creates a new iterator calling the closure on each of the original iterator's elements.

The returned `Map` is defined as follows:

```ignore
pub struct Map<I, F> {
    iter: I,
    f: F,
}

impl<B, I: Iterator, F> Iterator for Map<I, F>
    where F: FnMut(I::Item) -> B
{
    type Item = B;

    #[inline]
    fn next(&mut self) -> Option<B> {
        self.iter.next().map(&mut self.f)
    }

    #[inline]
    fn size_hint(&self) -> (usize, Option(usize)) {
        self.iter.size_hint()
    }

    #[inline]
    fn fold<Acc, G>(self, init: Acc, mut g: G) -> Acc
        where G: FnMut(Acc, Self::Item) -> Acc
    {
        let muf f = self.f;
        self.iter.fold(init, move |acc, elt| g(acc, f(elt))) // move `g` inside the closure
    }
}
```

Note that for `I: DoubleEndedIterator` or `ExactSizeIterator` or `FusedIterator`, and some other utility traits like `Debug`, `Clone`, etc.. ; `Map` and `Zip` would also have these corresponding traits implemented. The same goes for a lot of other adapters. These details would be omitted in this intro.

---

```ignore
pub trait Iterator {
    fn filter<P>(self, predicate: P) -> Filter<Self, P>
        where Self: Sized, P: FnMut(&Self::Item) -> bool
    {
        Filter { iter: self, predicate: predicate }
    }
}
```

The returned `Filter` consumes the original iterator, and would yield the original iterator's items iff. the predicate `P` is true for that item.

The definition of the returned `Filter`:

```ignore
pub struct Filter<I, P> {
    iter: I,
    predicate: P,
}

impl<I: Iterator, P> Iterator for Filter<I, P>
    where P: FnMut(&I::Item) -> bool 
{
    type Item = I::Item;

    #[inline]
    fn next(&mut self) -> Option<I::Item> {
        for x in self.iter.by_ref() {
            if (self.predicate)(&x) {
                return Some(x);
            }
        }
        None
    }

    #[inline]
    fn size_hint(&self) -> (usize, Option<usize>) {
        let (_, upper) = self.iter.size_hint();
        (0, upper) // because of the predicate, the original lower bound can't be trusted
    }
}

```

---

```ignore
pub trait Iterator {
    fn filter_map<B, F>(self, f: F) -> FilterMap<Self, F>
        where Self: Sized, F: FnMut(Self::Item) -> Option<B>
    {
        FilterMap { iter: self, f: f }
    }
}
```

Like `map`, this method returns a mapping adapter. However, the functor taken does more than just mapping: by returning an `Option`, it also does the filtering.

The returned `FilterMap` is defined as follows:

```ignore
pub struct FilterMap<I, F> {
    iter: I,
    f: F,
}

impl<B, I: Iterator, F> Iterator for FilterMap<I, F>
    where F: FnMut(I::Item) 0> Option<B>
{
    type Item = B;
    #[inline]
    fn next(&mut self) -> B {
        for x in self.iter.by_ref() {
            if let Some(ret) = (self.f)(x) {
                return Some(ret);
            }
        }
        None
    }

    #[inline]
    fn size_hint(&self) -> (usize, Option<usize>) {
        let (_, upper) = self.iter.size_hint();
        (0, upper) // can't know a lower bound, due to the predicate
    }
}
```

---

```ignore
fn enumerate(self) -> Enumerate<Self> where Self: Sized {
    Enumerate { iter: self, count: 0 }
}
```

The method creates an adapter yielding the item as well as a `count: usize` of the current iteration, in a tuple as `(count, item)`. The is useful when counting is needed. Enumerating an iterator which can yield more `usize::MAX` might either produces the wrong result, or panics.

The returned adapter is defined as follows:

```ignore
pub struct Enumerate<T> {
    iter: I,
    count: usize,
}

impl<I> Iterator for Enumerate<I> where I: Iterator {
    type Item = (usize, <I as Iterator>::Item);

    #[inline]
    #[rustc_inherit_overflow_checks] // The attribute ensures that in debug builds the method would always be checked against overflows.
    fn next(&mut self) -> Option<(usize, <I as Iterator>::Item)> {
        self.iter.next().map(|a| {
            let ret = (self.count, a);
            // possible overflows
            self.count += 1;
            ret
        })
    }

    #[inline]
    fn size_hint(&self) -> (usize, Option<usize>) {
        self.iter.size_hint()
    }

    #[inline]
    #[rustc_inherit_overflow_checks]
    fn nth(&mut self, n: usize) -> Option<(usize, I::Item)> {
        self.iter.nth(n).map(|a| {
            let i = self.count + n;
            self.count = i + 1;
            (i, a)
        })
    }

    #[inline]
    fn count(self) -> usize {
        self.iter.count()
    }
}
```

---

```ignore
pub trait Iterator {
    fn peekable(self) -> Peekable<Self> where Self: Sized {
        Peekable { iter: self, peeked: None }
    }
}
```

The method creates an adapter that can `peek` into the net element of the iterator, without consuming it.

The returned adapter's type is defined as

```ignore
pub struct Peekable<I: Iterator> {
    iter: I,
    peeked: Option<Option<I::Item>>,
}

impl<I: Iterator> Iterator for Peekable<I> {
    type Item = I::Item;

    #[inline]
    fn next(&mut self) -> Option<I::Item> {
        match self.peeked.take() {
            Some(v) => v,
            Non => self.iter.next(),
        }
    }
}
```