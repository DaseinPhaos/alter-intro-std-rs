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

The required `next` method defines the actual mechanism of iteration. Every invocation of it means another round of iteration. It returns an `Option<Item>`. If the returned value is a `Some`, that element is the result of this round of iteration. Returning a `None`, on the other hand, means that the iterator has finished the whole iteration process. The iterator is permitted to resume iteration after such a `None` is returned, however. Subsequent calls to `next` *can* start returning `Some` again at some point. it is up to the implementation to decide how the returned value should be obtained. However, the general philosophy of iterators is to apply some sort of "lazy evaluation", that is, we don't perform any evaluation unless we have to.

The module also defines some other more specific kinds of iterators on top of this trait. This trait itself also has a bunch of default methods for utility usage. Lots of these methods deal with iterator composition, the module also contains corresponding generic struct definitions for composited iterators (aka. adapters). We will come to these definitions later in this post.

For now let's just check out how to define a simple iterator:

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

---

```ignore
pub trait Iterator {
    fn skip_while<P>(self, predicate: P) -> SkipWhile<Self, P>
        where Self: Sized, P: FnMut(&Self::Item) -> bool
    {
        SkipWhile{ iter: self, flag: false, predicate: predicate }
    }
}
```

This method returns an adapter that skips through the original iterator's yielding items until an invocation to the predicate returns `false`. After that, *every* succeeding element would be returned as is. The predicate would by then be done for. This is where this method differs from a negative `filter`.

The returned adapter is defined as follows:

```ignore
pub struct SkipWhile<I, P> {
    iter: I,
    flag: bool,
    predicate: P,
}

impl<I: Iterator, P> Iterator for SkipWhile<I, P>
    where P: FnMut(&I::Item) -> bool
{
    type Item = I::Item;

    #[inline]
    fn next(&mut self) -> Option<I::Item> {
        for x in self.iter.by_ref() {
            if self.flag || !(self.predicate)(&x) {
                self.flag = true;
                return Some(x);
            }
        }
        None
    }

    #[inline]
    fn size_hint(&self) -> (usize, Option<usize) {
        let (_, upper) = self.iter.size_hint();
        (0, upper) // lower bound can't be trusted due to skipping
    }
}
```

Note that unlike most other adapters, this adapter doesn't persist a `ExactNumberAdapter` or `DoubleEndnedAdapter`.

---

```ignore
pub trait Iterator {
    fn take_while<P>(self, predicate: P) -> TakeWhile<Self, P> { /* ... */}
}
```

Creates an adapter that yields original iterator's elements until the `predicate` when applying on the element, returns a `false`. This is quiet the opposite of `skip_while`. We omit implementation details here.

---

```ignore
pub trait Iterator {
    fn skip(self, n: usize) -> Skip<Self> { /* ... */}
}
```

Creates an adapter that skips the next `n` elements of the original iterator.

---

```ignore
pub trait Iterator {
    fn take(self, n: usize) -> Take<Self> { /* ... */}
}
```

Creates an adapter that only takes the next `n` elements of the original iterator.

---

```ignore
pub trait Iterator {
    fn scan<St, B, F>(self, initial_state: St, f: F) -> Scan<Self, St, F>
        where Self: Sized, F: FnMut(&mut St, Self::Item) -> Option<B>
    {
        Scan { iter: self, f: f, state: initial_state }
    }
}
```

Like `fold`, this method takes and accumulates a `state` object. Unlike `fold` which only returns the final result, `scan` here returns another iterator adapter, yielding out some "intermediate results" upon every iteration of the original iterator.

The definition of the returned structure is as follows:

```ignore
pub struct Scan<I, St, F> {
    iter: I,
    f: F,
    state: St,
}

impl<B, I, St, F> Iterator for Scan<I, St, F>
    where I: Iterator,
          F: FnMut(&mut St, I::Item) -> Option<B>
{
    type Item = B;

    #[inline]
    fn next(&mut self) -> Option<B> {
        self.iter.next().and_then(|a| (self.f)(&mut self.state, a))
    }

    #[inline]
    fn size_hint(&self) -> (usize, Option<usize>) {
        let (_, upper) = self.iter.size_hint();
        (0, upper)
    }
}
```

Note from the implementation that, the adapter can terminate when:

- original iterator terminates,
- scanning function decides that it should return `None`.

---

```ignore
pub trait Iterator {
    fn flat_map<U, F>(self, f: F) -> FlatMap<Self, U, F>
        where Self: Sized, U: IntoIterator, F: FnMut(Self::Item) -> U,
    {
        FlatMap { iter: self, f: f, frontiter: None, backiter: None }
    }
}
```

Creates a "flattened mapping". While `map` is certainly helpful, for deeply nested data, it can't do much.

Say we are to total number of `'a'`'s in a collection of strings. This method captures such need precisely and efficiently:

```rust
let words = ["alpha", "beta", "gamma"];
let count_a = words.iter().flat_map(|s| s.chars()) // Now a iterator over all characters
    .filter(|c| *c=='a').count();
assert_eq!(count_a, 5);
```

Let's take a look at how this is implemented:

```ignore
pub struct FlatMap<I, U: IntoIterator, F> {
    iter: I,
    f: F,
    frontiter: Option<U::IntoIter>,
    backiter: Option<U::IntoIter>, // for double ended iterators
}

impl<I: Iterator, U: IntoIterator, F> Iterator for FlatMap<I, U, F>
    where F: FnMut(I::Item) -> U
{
    type Item = U::Item;

    #[inline]
    fn next(&mut self) -> Option<U::Item> {
        loop {
            if let Some(ref mut inner) = self.frontiter {
                if let Some(x) = inner.by_ref().next() {
                    return Some(x)
                }
            }
            match self.iter.next().map(&mut self.f) {
                None => return self.backiter.as_mut().and_then(|it| it.next()),
                next => self.frontiter = next.map(IntoIterator::into_iter),
            }
        }
    }

    #[inline]
    fn size_hint(&self) -> (usize, Option<usize>) {
        let (flo, fhi) = self.froniter.as_ref().map_or(0, Some(0)), |it| it.size_hint());
        let (blo, bhi) = self.backiter.as_ref().map_or(0, Some(0)), |it| it.size_hint());
        let lo = flo.saturating_add(blo);
        match (self.iter.size_hint(), fhi, bhi) {
            (0, Some(0), Some(a), Some(b)) => (lo, a.checked_add(b)),
            _ => (lo, None)
        }
    }
}
```

Note that to support `DoubleEndedIterator`s, the structure adds some additional checking, which would otherwise be redundant. I think when specialization kicks in `stable` this tiny overhead should be gone.

---

```ignore
pub trait Iterator {
    fn fuse(self) -> Fuse<Self> {/* ... */}
}
```

The `Iterator` trait allows its implementation to somewhat resume iterating after yielding a `None` item. For those that once returned `None` always returning `None` from then on, the module provides another wrapper trait, `FusedIterator`. Currently it is unstable, however this method do let us create such a iterator adapter that implements it.

---

```ignore
pub trait Iterator {
    fn inspect<F>(self, f: F) -> Inspect<Self, F>
        where Self: Sized, F: FnMut(&Self::Item) { /* ... */ }
}
```

Creates an adapter inspects the original iterator's elements with `f: F`, and passing the value on.

When working with iterators, usually we'd have to do a lot of chaining. This is where inspection gets useful.

Continuing with our last example, we can now add some inspections:

```rust
let words = ["alpha", "beta", "gamma"];
let count_a = words.iter()
    .inspect(|s| println!("before flat mapping: {}", s))
    .flat_map(|s| s.chars()) // Now a iterator over all characters
    .inspect(|c| println!("before filtering: {}", c))
    .filter(|c| *c=='a')
    .inspect(|c| println!("after filtering: {}", c))
    .count();
assert_eq!(count_a, 5);
```

---

```ignore
pub trait Iterator {
    fn by_ref(&mut self) -> &mut Self { /* ... */ }
}
```

Borrows the iterator by **mutable** reference. Note that naming convention of rust methods is violated here. This method is useful with adapter chaining, when we don't want the original iterator to be consumed.

---

```ignore
pub trait Iterator {
    fn collect<B>(self) -> B where B: FromIterator { /* ... */ }
}
```

Collects all the items of the iterator into a `B: FromIterator`. Such a `B` is often some sort of collections:

```ignore
pub trait FromIterator<A> {
    fn from_iter<T>(iter: T) -> Self
        where T: IntoIterator<Item=A>;
}
```

---

```ignore
pub trait Iterator {
    fn partition<B, F>(self, mut f: F) -> (B, B)
        where Self: Sized,
              B: Default + Extend<Self::Item>,
              F: FnMut(&Self::Item) -> bool
    {
        let mut left: B = Default::default();
        let mut right: B = Default::default();
        for x in self {
            if f(&x) {
                let.extend(Some(x))
            } else {
                right.extend(Some(x))
            }
        }
        (left, right)
    }
}
```

Consumes the iterator, and creates two collections from it according to the given predicate.

The collection type should be `Default` constructible, and `Extend`able over the item type.

---

```ignore
pub trait Iterator {
    fn all<F>(&mut self, mut f: F) -> bool
        where Self: Sized, // for taking by value
              F: FnMut(Self::Item) -> bool
    {
        for x in self {
            if !f(x) {
                return false;
            }
        }
        true
    }

    fn any<F>(&mut self, mut f: F) -> bool
        where Self: Sized,
              F: FmMut(Self::Item) -> bool
    {
        for x in self {
            if f(x) {
                return true;
            }
        }
        false
    }
}
```

These two are straightforward.

---

```ignore
pub trait Iterator {
    fn find<F>(&mut self, mut f: F) -> Option<Self::Item>
        where Self: Sized,
              P: FnMut(&Self::Item) -> bool
    {
        for x in self {
            if f(&x) { return Some(x) }
        }
        None
    }
}
```

Returns the first item matching the predicate, if any.

---

```ignore
pub trait Iterator {
    fn position<P>(&mut self, mut predicate: P) -> Option<usize>
        where Self: Sized,
              P: FnMut(Self::Item) -> bool,
    {
        // `enumerate` might overflow.
        for (i, x) in self.enumerate() {
            if predicate(x) {
                return Some(i);
            }
        }
        None
    }
}
```

Returns the index ( from the current position) of the first item matching the predicate. Now that this predicate consumes items by value, unlike the one used by `find`.

---

For `ExactSizedIterator` and `DoubleEndedIterator`, we also have:

```ignore
pub trait Iterator {
    fn rposition<P>(&mut self, mut predicate: P) -> Option<usize>
        where P: FnMut(Self::Item) -> bool,
              Self: Sized + ExactSizeIterator + DoubleEndedIterator
    {
        let mut i = self.len();

        while let Some(v) = self.next_back() {
            if predicate(v) {
                return Some(i - 1);
            }
            // No need for an overflow check here, because `ExactSizeIterator`
            // implies that the number of elements fits into a `usize`.
            i -= 1;
        }
        None
    }
}
```

---

For ordered item types, we have:

```ignore
pub trait Iterator {
    fn max(self) -> Option<Self::Item>
        where Self: Sized,
              Self::Item: Ord
    {
        // ...
    }

    fn min(self) -> Option<Self::Item>
        where Self: Sized,
              Self::Item: Ord
    {
        // ...
    }
}
```

These functions returns the value closest to the current position of the iterator, among equally `max`/`min`s.

Additionally, we have:

```ignore
pub trait Iterator {
    fn max_by_key<B, F>(self, f: F) -> Option<Self::Item>
        where B: Ord,
              F: FnMut(&Self::Item) -> B { ... }
    fn max_by<F>(self, compare: F) -> Option<Self::Item> where F: FnMut(&Self::Item, &Self::Item) -> Ordering { ... }
    fn min_by_key<B, F>(self, f: F) -> Option<Self::Item> where B: Ord, F: FnMut(&Self::Item) -> B { ... }
    fn min_by<F>(self, compare: F) -> Option<Self::Item> where F: FnMut(&Self::Item, &Self::Item) -> Ordering { ... }
}
```

that can cooperate with customized ordering functions.

---

For `DoubleEndedIterator`s, we have a reverse adapter:

```ignore
pub trait Iterator {
    fn rev(self) -> Rev<Self>
        where Self: Sized + DoubleEndedIterator
    { Rev {iter: self} }
}
```

---

For a "zipped" iterator, we can unzip them into separate collections:

```ignore
pub trait Iterator {
    fn uzip<A, B, FromA, FromB>(self) -> (FromA, FromB)
        where FromA: Default + Extend<A>,
              FromB: Default + Extend<B>,
              Self: Sized + Iterator<Item=(A, B)>,
    {
        let mut as: FromA = Default::default();
        let mut bs: FromB = Default::default();

        for(a, b) in self {
            as.extend(Some(a)); // pattern matching here
            bs.extend(Some(b));
        }

        (as, bs)
    }
}
```

---

For an iterator that operates on some `& T`, where `T: Clone`, we have an adapter yielding each of its item `clone`d:

```ignore
fn cloned<'a, T>(self) -> Cloned<Self>
    where Self: Sized + Iterator<Item=&'a T>,
          T: Clone
{
    Cloned { it: self }
}
```

The returned adapter is defined as

```ignore
pub struct Cloned<T> {
    it: I,
}

impl<'a, I, T> Iterator for Cloned<T>
    where I: Iterator<Item=&'a T>,
          T: 'a + Clone
{
    type Item = T;

    fn next(&mut self) -> Option<T> {
        self.it.next().cloned()
    }
     fn size_hint(&self) -> (usize, Option<usize>) {
         self.it.size_hint()
     }

     fn fold<Acc, F>(self, init: Acc, mut f: F) -> Acc
        where F: FnMut(Acc, Self::Item) -> Acc
    {
        self.it.fold(init, move |acc, elt| f(acc, elt.clone()))
    }
}
```

Note that the adapter has a customized version of `fold` implemented. This can be considered as an optimization, because when calling `fold` we presumes that we don't care about the intermediate items. However, this also means that if the `clone` method of `Item` has any kind of side-effects, they won't appear either (as one would reasonably assume) when calling `fold`.

---

```ignore
pub trait Iterator {
    fn cycle(self) -> Cycle<Self> where Self: Sized + Clone {
        Cycle{orig: self.clone(), iter: self}
    }
}
```

Creates an adapter that, upon termination of the original iterator, *cycle*s the iteration process again. Looking into its implementation, we can see that it relies on the original iterator to be `Clone`able to achieve this.

```ignore
pub struct Cycle<I> {
    orig: I,
    iter: I,
}

impl<I> Iterator for Cycle<I> where I: Clone + Iterator {
    type Item = <I as Iterator>::Item;

    #[inline]
    fn next(&mut self) -> Option<<I as Iterator>::Item> {
        match self.iter.next() {
            None => { self.iter = self.orig.clone(); self.iter.next() }
            y => y
        }
    }

    #[inline]
    fn size_hint(&self) -> (usize, Option<usize>) {
        // the cycle iterator is either empty or infinite
        match self.orig.size_hint() {
            sz @ (0, Some(0)) => sz,
            (0, _) => (0, None),
            _ => (usize::MAX, None)
        }
    }
}
```

---

For summable item types, we have

```ignore
pub trait Iterator {
    fn sum<S>(self) -> S
        where Self: Sized,
              S: Sum<Self::Item>
    {
        Sum::sum(self)
    }
}
```

`Sum` here is a trait that represents a type that can be created by summing up another type:

```ignore
pub trait Sum<A = Self> {
    fn sum<I>(iter: I) -> Self
        where I: Iterator<Item=A>;
}
```

The required method describes how the summing would be performed.

For built-in numeric types (as well as their `Wrapping`s) this trait is implemented by `libstd`.

---

Similarly, we have

```ignore
pub trait Iterator {
    fn product<P>(self) -> P
        where Self: Sized,
              P: Product<Self::Item>,
    {
        Product::product(self)
    }
}
```

where

```ignore
pub trait Product<A = Self> {
    fn product<I>(iter: I) -> Self
        where I: Iterator<Item=A>;
}
```

to deal with products. The trait is also implemented properly for built-in numeric types.

---

Finally, for comparable item types, we have a bunch of methods to deal with lexicographical comparisons of iterators (actually comparing against their items):

```ignore
pub trait Iterator {
    fn cmp<I>(self, other: I) -> Ordering where I: IntoIterator<Item=Self::Item>, Self::Item: Ord { ... }
    fn partial_cmp<I>(self, other: I) -> Option<Ordering> where I: IntoIterator, Self::Item: PartialOrd<I::Item> { ... }
    fn eq<I>(self, other: I) -> bool where I: IntoIterator, Self::Item: PartialEq<I::Item> { ... }
    fn ne<I>(self, other: I) -> bool where I: IntoIterator, Self::Item: PartialEq<I::Item> { ... }
    fn lt<I>(self, other: I) -> bool where I: IntoIterator, Self::Item: PartialOrd<I::Item> { ... }
    fn le<I>(self, other: I) -> bool where I: IntoIterator, Self::Item: PartialOrd<I::Item> { ... }
    fn gt<I>(self, other: I) -> bool where I: IntoIterator, Self::Item: PartialOrd<I::Item> { ... }
    fn ge<I>(self, other: I) -> bool where I: IntoIterator, Self::Item: PartialOrd<I::Item> { ... }
}
```

# Other iterators

```ignore
pub trait DoubleEndedIterator: Iterator {
    fn next_back(&mut self) -> Option<Self::Item>;
}

pub trait ExactSizeIterator: Iterator {
    #[inline]
    fn len(&self) -> usize {
        let (lower, upper) = self.size_hint();
        // Note: This assertion is overly defensive, but it checks the invariant
        // guaranteed by the trait. If this trait were rust-internal,
        // we could use debug_assert!; assert_eq! will check all Rust user
        // implementations too.
        assert_eq!(upper, Some(lower));
        lower
    }

    /// Returns whether the iterator is empty.
    #[inline]
    #[unstable(feature = "exact_size_is_empty", issue = "35428")]
    fn is_empty(&self) -> bool {
        self.len() == 0
    }
}

#[unstable(feature = "fused", issue = "35602")]
pub trait FusedIterator: Iterator {}

#[unstable(feature = "trusted_len", issue = "37572")]
pub unsafe trait TrustedLen : Iterator {}
```

TODO: introduce them a bit.

# Utility Functions

```ignore
/// Creates an iterator that yields nothing.
pub fn empty<T>() -> Empty<T> {
    Empty(marker::PhantomData)
}
```

```ignore
/// Creates an iterator that yields an element exactly once.
#[stable(feature = "iter_once", since = "1.2.0")]
pub fn once<T>(value: T) -> Once<T> {
    Once { inner: Some(value).into_iter() }
}
```

```ignore
/// Creates a new iterator that endlessly repeats a single element.
#[inline]
#[stable(feature = "rust1", since = "1.0.0")]
pub fn repeat<T: Clone>(elt: T) -> Repeat<T> {
    Repeat{element: elt}
}
```