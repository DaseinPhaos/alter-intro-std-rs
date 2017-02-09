% std::fmt

> https://doc.rust-lang.org/std/fmt/

The module contains the supporting traits for the `format!` series of macros.

# Formatting Syntax

First let's check out the syntax of formatting.

At its minimum, the macro requires a *format string*, which must be a string literal:

```rust
format!("Hey there!"); // => "Hey there!"
```

Additionally, it can take one or more formatting arguments. These additional arguments would be captured by `{}`s in the format string. Quick examples:

```rust
format!("No.{}", 1); // => "No.1"
format!("{}, {}!", "Hello", "world"); => "Hello, world!"
```

In the format string, a `{` character can be escaped with `{{`, similarly, a `}` can be escaped with `}}`:

```rust
format!("{{}}"); // => "{}"
```

A format string is required to use *all* of its arguments. Otherwise, it is a compile-time error.

By default, a `{}` would capture the "next" formatting argument. However, one can put integer index into the braces to jump out of the default rules and directly index into the formatting arguments it want to capture:

```rust
format!("{1} {0}", 1, 2); // => "2 1"
```

These "positional captures" don't participate the iterating progress of default captures, thus:

```rust
format!("{1} {} {0} {}", 1, 2); // => "2 1 1 2"
```

Formatting parameters can be explicitly named. When named, format string can capture them by their name:

```rust
format!("{name}", name = "test"); // => "test"
format!("{two} {} {}", 1, two="2"); // => "2 1 2"
```

Named formatting parameters behave just like normal formatting parameters. There is one restriction though: it is not valid to put normal parameters after any named parameters.

## Argument Types

Each formatting argument have a "type", dictated by the format string. These "types" are specified in the capturing braces. When a same parameter is captured for multiple times, the format string has to make sure that all those captures refer to the argument by *one* type. These types are actually mapped to traits defined in this module. Currently, the mapping is as follows:

- *empty* => `Display`
- `?` => `Debug`
- `o` => `Octal`
- `x` => `LowerHex`
- `X` => `UpperHex`
- `p` => `Pointer`
- `b` => `Binary`
- `e` => `LowerExp`
- `E` => `UpperExp`

Similar to Rust's syntax when introducing a variable, inside the capturing braces, a `:` is optionally used to separate the formatting argument's name/index(if any) from its type.

That is to say, any type of formatting argument implementing `Display` can then be captured in the format string with `{}`,  any type of argument implementing `Binary` can be captured with `{:b}`...

We will come to how to actually implement these traits for custom types later. For now all we have to know is that the standard library has implemented these traits for built-in types nicely and properly for us. Thus, we can write:

```rust
format!("{}", 10); // => "10"
format!("{:o}", 10); // => "12"
format!("{:x}", 10); // => "a"
format!("{:X}", 10); // => "A"
format!("{:b}", 10); // => "1010"
```

## Additional Specifications

After the `:` and before the type specification, additional formatting specifications can be used. Firt comes the width:

```rust
format!("{:2x}, 10); // => " a"
format!("{:2}", "a"); // => "a "
```

This specifies the minimum width that the formatted argument should take up. Besides literal values, the width can also be specified by another formatting parameter. The specification can be done either by name or by index, but should be suffixed with a `$`:

```rust
format!("{:width$x}", 10, width=2); // => " a"
format!("{:0$x}", 2); // => " 2"
```

If the formatted argument doesn't fill up, then padding is added.

For non-numerics, the default padding strategy is to fill with space character, left-aligned. For numerics, the default padding strategy is to fill with space character, right-aligned.

The padding strategy can be customized. The alignment is specified by a preceding `<`(left-aligned), `^`(center-aligned) or `>`(right-aligned):

```rust
format!("{:^4}", 10); // => " 10 "
```

Additionally, a customized filling character can be provided before the alignment specification:

```rust
format!("{:*^4}, 10); // => "*10*"
```

For numeric types, additional specifications can be inserted between padding specification and width specification. Namely:

- `+` can be used to specify that signs should always be printed.
- `#` specifies some additional printing forms:
    - `#?` would pretty-print the `Debug` formatting trait
    - `#x` precedes arguments with `0x`
    - `#X` precedes with `0x`
    - `#b` precedes with `0b`
    - `#o` precedes with `0o`
- `0` indicates that integer padding should both be done with `0` as well as be sign-aware. Thuse, `format!({:03}, 1)` yields `"001`; `format!({:03}, -1)` yields `-01`.


For non-numeric types, after the width specification, an `.precision` can be used to specify the "maximum width" of the formatted string. Longer results would be truncated. For integer types, this is ignored. For floating-point types, this indicates how many digits after thte decimal points should be printed.

To specify the `precision`, one can use:
- An integer
- An indiced/named argument, preceding with `$`(just like the width specification)
- An asterisk(`.*`). This will consume two argument slots. The first input should be of type `usize`, representing the precision; the second input should be the actual argument to be printed with this precision.

That's all about syntax. Next we are going to cover how to format custom defined types.

# Formatter

Before directly diving into the forming traits, however, we should take a look at the `Formatter` struct first, as this is the key component of custom type formattting.

Every formatting traits defines a `fmt` method which takes a `&mut Formatter` as an argument. When a trait is implemented, the implementing type can then be used as the corresponding type during formatting.

```ignored
type Result = Result<(), Err>;
// ...
pub trait Display {
    fn fmt(&self, f:&mut Formatter) -> Result;
}
```

The formatter has lots of methods exposed. However, normally we don't need to use any of it: it implements `std::io::Write`, thus, we can use standard macro `write!` and `writeln!` to deal with it instead. An example:

```rust
use std::fmt;

struct Point {
    x: i32,
    y: i32,
}

impl fmt::Display for Point {
    fn fmt(&self, f:&mut fmt::Formatter) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}

let p = Point { x: 0, y: 1};
let p_display = format!("{}", p);
assert_eq!(p_display, "(0, 1)");
```

Complete listing of methods can be found in the [reference](https://doc.rust-lang.org/std/fmt/struct.Formatter.html).

> TODO: introduce `format_args!`.