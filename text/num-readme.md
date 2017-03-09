% std::num

This module contains structs and enums that provide additional numerical functionalities.

It mainly provides:

```ignore
pub struct Wrappng<T>(pub T);
```

A wrapper type around numerical value types, such that standard arithmetic operations on the underlying value have wrapping semantics.

---

```ignore
pub enum FpCategory {
    Nan, // Not a number
    Infinite, // positive or negative infinity
    Zero, // positive or negative zero
    Subnormal, // de-normalized floating point representations
    Normal, // a regular floating point number
}
```

A classification of floating point values, used as the return type for `f32::classify()` and `f6::classify()`.

