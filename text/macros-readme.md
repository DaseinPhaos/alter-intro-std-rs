% Macros in std

Lists all the macros defined in `std`.

TODO:

- [x] categorize them.
- [ ] add examples for the formatting macros, maybe referencing a standalone post on formatting.
- [ ] introduce unstable macros.

# Panic!

First comes the Mighty Panic.

> [`panic!`](https://doc.rust-lang.org/std/macro.panic.html)
>
> Injects panic into the calling thread, causing it to panic entirely.
>
> Panic can be reaped as a `Box<Any>`.
>
> Single-argument form transmits the argument, multi-argument form effectively forwards the argument to a `format!` call and transmits the returned `String`.


# Formatting

These macros are used to `format` a `String`; `write` the result to some buffer; specifically, `print` to the standard output.

These are among the most widely used and well-known macros defined in `std` rust, thus should be covered here first. However, the actual formatting syntax they depend on is somewhat involved, and thus will be introduced in a separate post about [`std::fmt`](fmt-readme.md).[^rbef]

[^rbef]: reference to the correpsonding chapter in [Rust by Example](http://rustbyexample.com/hello/print.html)

Some other macros(such as `assert!`) also utilize the formatting syntax, but as formatting is not their primary concern, they are not put under this category.

> [`format!`](https://doc.rust-lang.org/std/macro.format.html)
>
> Use the syntax described in [`std::fmt`](fmt-readme.md) to create a value of `String`.
>
> [`format_args!`]
>
> The internal macro for formatted string creation. Also described above.

> [`write!`](https://doc.rust-lang.org/std/macro.write.html)
>
> Write formatted data into a buffer.
>
> The macro accepts a 'writer' (any value with a `write_fmt` method).
>
> The `write_fmt` method usually comes from implementation of `std::fmt::Write` or `std::io::Write` traits.
>
> Additional arguments will be formatted as presented to a `format!`.
>
> Returns whatever the `write_fmt` method returns, common return values include `std::fmt::Result` of `std::io::Result`.

> [`writeln!`](https://doc.rust-lang.org/std/macro.writeln.html)
>
> Similar to `write!`, but appends a newline character.
>
> The newline character is the LINE FEED character(`\n`, `U+000A`).

> [`print!`](https://doc.rust-lang.org/std/macro.print.html)
>
> Prints to the standard output using `format!` to format the input.
>
> Panics if writing fails.

> [`println!`](https://doc.rust-lang.org/std/macro.println.html)
>
> Similar to `print!`, but appends a newline character.
>
> The newline character is the LINE FEED character(`\n`, `U+000A`).

# Assertions

Assertions are runtime checks that once failed would `panic` the calling thread. They are used to check core invariants that shouldn't be violated. Also useful during testing.

> [`assert!`](https://doc.rust-lang.org/std/macro.assert.html)
>
> Ensures that a boolean expression evaluates to `true` at runtime. If not, `panic!`.
>
> Always checked, no matter what the build settings are. `debug_assert!` is for assertions that are not enabled in release builds by default.
>
> Additional arguments to the macro will be sent to `format!` to provide a custom panic message.
>
> Usage:
>
> - Enforce runtime invariants (esp. for unsafe code).
> - [testing](https://doc.rust-lang.org/book/testing.html)

>[`assert_eq!`](https://doc.rust-lang.org/std/macro.assert_eq.html)
>
>Similar to `assert!($left_expr != $right_expr)`.

>[`assert_ne!`](https://doc.rust-lang.org/std/macro.assert_ne.html)
>
>Similar to `assert!($left_expr == $right_expr)`.

> [`debug_assert!`](https://doc.rust-lang.org/std/macro.concat.html)
>
> [`debug_assert_eq!`](https://doc.rust-lang.org/std/macro.debug_assert_eq.html)
>
> [`debug_assert_ne!`](https://doc.rust-lang.org/std/macro.debug_assert_ne.html)
>
> Analogies for `assert!`s, but are only evaluated in debug builds by default.
>
> If `-C debug-assertions` is passed to the compiler, these assertions won't be optimized.

# Threading

> [`thread_local`](https://doc.rust-lang.org/std/macro.thread_local.html)
>
> Declares a new thread local storage key of type `std::thread::LocalKey`.
>
> The macro wraps any number of static delcarations and makes them thread local.
>
> ```rust
> use std::cell::RefCell;
> thread_local! {
>   pub static FOO: RefCell<u32> = RefCell::new(1);
>   #[allow(unused)]
>   static BAR: RefCell<f32> = RefCell::new(1.0);
> }
> ```

# Vector Construction

> [`vec!`](https://doc.rust-lang.org/std/macro.vec.html)
>
> Creates a `Vec` containing the arguments, allowing `Vec` to be defined with the same syntax as array exprs. It can
>
> - Create a `Vec` containing a given list of elements.
>
>   ```rust
>   let v = vec![1, 2, 3];
>   assert_eq!(v[1], 2);
>   ```
>
> - Create a `Vec` from a given element and size:
>
>   ```rust
>   let v = vec![1; 3];
>   assert_eq!(v, [1, 1, 1]);
>   ```
>
> Unlike array exprs, this supports all elements implementing `Clone`.
>
> This will use `clone()` to duplicate an expression.

# The Once Mighty `try!`

> [`try!`](https://doc.rust-lang.org/std/macro.try.html)
>
> Matches a `Result`. If `Ok`, expands to the wrapped value. If `Err`, retrieves the inner error, and returned it. Thus it can only be used in functions returning a `Result`.
>
> This was once a popular macro. In fact, it was so popular, that the Rust team decided to introduce it into the core syntax, using `?`. Now we should always prefer using `?`.

# Panicing Markers

> [`unimplemented!`](https://doc.rust-lang.org/std/macro.unimplemented.html)
>
> A standardized placeholder for marking unfinished code.
>
> When executed, panics with msg `"not yet implemented"`.

> [`unreachable!`](https://doc.rust-lang.org/std/macro.unreachable.html)
>
> Indicating unreachable code.
>
> Useful when compiler can't determine if the code is unreachable. e.g. when:
>
> - Match arms with guard conditions.
> - Loops that dynamically terminate.
> - Iterators that dynamically terminate.
>
> When executed, always panics.


# Compile-time str generation

These macros can take some input tokens and turns them to some specific `&'static str`s during compile time.

> [`concat!`](https://doc.rust-lang.org/std/macro.concat.html)
>
> Concatenates literals into a `&'static str`. Integers and floating points would be stringified.
>
> ```rust
> let s = concat!("test", 10, 'b', true);
> assert_eq!(s, "test10btrue");
> ```

> [`stringify!`](https://doc.rust-lang.org/std/macro.stringify.html)
>
> Stringifies the argument, yielding an expression of type `&'static str`, which is the stringification of all the input tokens.
>
> The expanded result is subjected to changes in the future, thus should not be relied on.
>
> ```rust
> // That said, we do rely on its output in the example.
> let one_p_one = stringify!(1 + 1);
> assert_eq!(one_p_one, "1 + 1");
> ```

> [`env!`](https://doc.rust-lang.org/std/macro.env.html)
>
> Inspect environment variables at compile time, yielding an `&'static str`.
>
> ```rust
> let path = env!("PATH");
> println!("The PATH variable at the time of compiling was: {}", path);
> ```

> [`option_env!`](https://doc.rust-lang.org/std/macro.option_env.html)
>
> Similar to `env!`, but returns an `Option<&'static str>` instead, `None` if the environment variable is not presented.

> [`file!`](https://doc.rust-lang.org/std/macro.file.html)
>
> Expands to the file name from which it was invoked, yielding an `&'static str`.
>
> ```rust
> let this_filename = file!();
> println!("defined in file: {}", this_filename);
> ```

# Meta Information

> [`cfg!`](https://doc.rust-lang.org/std/macro.cfg.html)
>
> Boolean evaluation of configuration flags.
>
> Syntax given to this macro is the same syntax as [the `cfg` attribute](https://doc.rust-lang.org/reference.html#conditional-compilation).
>
> ```rust
> let my_dir = if cfg!(windows) {
>   "windows-specified-dir"
> } else {
>   "unix-dir"
> };
> ```

> [`column!`](https://doc.rust-lang.org/std/macro.column.html)
>
> Expands to the colunmn number on which it was invoked, yielding a `u32`.

> [`line!`](https://doc.rust-lang.org/std/macro.line.html)
>
> Expands to the line number on which it was invoked, yielding a `u32`.

> [`module_path!`](https://doc.rust-lang.org/std/macro.module_path.html)
>
> Expands to a string that represents the current module path, which is a hierarchy of modules leading back up to the crate root.
>
> The first component of the path returned is the name of the crate currently being compiled.
>
> ```rust
> mod test {
>   pub fn foo() {
>     assert!(module_path!().ends_with("test"));
>   }
> }
>
> test::foo();
> ```

# Includes

> [`include!`](https://doc.rust-lang.org/std/macro.include.html)
>
> Parse a file as an expression or an item according to context.
>
> Path is located relative to the current file.
>
> Using this macro is often considered a bad idea.

> [`include_bytes!`](https://doc.rust-lang.org/std/macro.include_bytes.html)
>
> Include a file as a reference to a byte array, yielding a `&'static [u8; N]` which is the contents of the referenced file.
>
> Path is located relative to the current file.

> [`include_str!`](https://doc.rust-lang.org/std/macro.include_str.html)
>
> Include a utf8 encoded file as a string, yielding a `&'static str` which is the contents of the referenced file.
>
> Path is located relative to the current file.