% std::hash

> https://doc.rust-lang.org/std/hash/

The module provides generic hashing support.

A [hash function](https://en.wikipedia.org/wiki/Hash_function) is a function that maps an arbitrary sized datum to a fixed-sized datum. The first trait defined in this module represents such a function object:

```ignore
pub trait Hasher {
    fn write(&mut self, bytes: &[u8]);
    fn finish(&self) -> u64;
}
```

The trait requires the above two methods to be implemented. `write` writes some input into the `Hasher`. Such writing can be accumulated, until a `finish` is invoked, yielding the final hashed value of the preceding data stream, as a `u64`.

With the `write` method defined, the trait provides implementations for writing numeric type datas:

```ignore
pub trait Hasher {
    // ...
    fn write_u8(&mut self, i: u8) { ... }
    fn write_u16(&mut self, i: u16) { ... }
    fn write_u32(&mut self, i: u32) { ... }
    fn write_u64(&mut self, i: u64) { ... }
    fn write_usize(&mut self, i: usize) { ... }
    fn write_i8(&mut self, i: i8) { ... }
    fn write_i16(&mut self, i: i16) { ... }
    fn write_i32(&mut self, i: i32) { ... }
    fn write_i64(&mut self, i: i64) { ... }
    fn write_isize(&mut self, i: isize) { ... }
}
```

Internally, these functions make use of the `write` method, as we'd expect.

With hashing functions abstracted as `Hasher`s, to actually define a hashable type, one simply implements `Hash`:

```ignore
pub trait Hash {
    fn hash<H: Hasher>(&self, state: &mut H);

    fn hash_slice<H: Hasher>(data: &[Self], state: &mut H) where Self: Sized {
        for piece in data {
            piece.hash(state);
        }
    }
}
```

The required method `hash` should be implemented such that, given a hashing function object `state`, the value of the object is written into it. One can also use the `#[derive]` attribute with this trait to implement an default hashing strategy, if all fields of the type imeplement `Hash`. The resulting hash will be the combination of calling `hash` on each field. Most of the primitive types implements `Hash`.

The defaultly implemented method `hash_slice` hashes a slice of `Self`.

Most of the primitive types and a lot of other types in the `std` implement `Hash`.


Additionally, the module also defines a `BuuldHasher`, which can be considered as a factory-like interface for building hashing function objects(i.e. `Hasher`s). It is defined as:

```ignore
pub trait BuuldHasher {
    type Hasher: Hasher;
    fn build_hasher(&self) -> Self::Hasher;
}
```

Note that, for each instance of a `BuildHasher`, the hashers it create should be identical in the hashing behavior.

The module also provides `BuildHasherDefault`:

```ignore
pub struct BuildHasherDefault<H>(marker::PhantomData<H>);

impl<H> BuildHasher for BuildHasherDefault<H> where H: Default + Hasher {
    type Hasher = H;
    fn build_hasher(&self) -> H {
        H::default()
    }
}
```

for `H: Hasher` that also implements `Default`. The method simply returns that default value.

The struct also implements `Debug`, `Clone` and `Default`.