% std::prelude

libstd is large. Some of the features it provides feels essential, so the compiler packs these things up in a prelude module.

Technically speaking, every Rust crate implicitly have

```ignore
extern crate std;
use std::prelude::v1::*;
```

in the crate root.

The currently used version of preludes, `v1`, exports the following items:

- `std::marker::{Copy, Send, Sized, Sync}`
- `std::ops::{Drop, Fn, FnMut, FnOnce}`
- `std::mem::drop`
- `std::boxed::Box`
- `std::borrow::ToOwned`
- `std::clone::Clone`
- `std::cmp::{PartialEq, PartialOrd, Eq, Ord}`
- `std::convert::{AsRef, AsMut, Into, From}`
- `std::default::Default`
- `std::iter::{Iterator, Extend, IntoIterator, DoubleEndedIterator, ExactSizeInterator}`
- `std::option::Option::{self, Some, None}`
- `std::result::Result::{self, Ok, Err}`
- `std::slice::SliceConcatExt`
- `std::string::{String, ToString}`
- `std::vec::Vec`

