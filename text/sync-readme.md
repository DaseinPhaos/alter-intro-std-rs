% std::sync

Synchronization primitives.

> https://doc.rust-lang.org/std/sync/

# Atomics

Submodule `atomic` contains atomic data types, including `AtomicBool`, `AtomicIsize`, `AtomicPtr` and `AtomicUsize`, along with constants `ATOMIC_BOOL_INIT` (`==false`), `ATOMIC_ISIZE_INIT` (`==0`) and `ATOMIC_USIZE_INIT` (`==0`).

All of these types are `Sync` (of course). All of them have an associated `new` method for data initialization, a `get_mut` method for single threaded mutation, and an `into_inner` method consuming them into its non-atomic counterparts. These methods are essentially single threaded.

Other methods of them, taking an [`enum Ordering`](file:///C:/Users/mlv/.rustup/toolchains/nightly-x86_64-pc-windows-msvc/share/doc/rust/html/std/sync/atomic/enum.Ordering.html) specifying the atomic memory ordering requirements of the operation, are intended for multithreaded scenarios. These methods performs loading, storing, swapping, compare and swapping, fetching and other bitwise operations.

The submodule also contains a `fence` function to prevents the compiler and CPU from reordering certain types of memory ops around it.

# Sync Primitives

On top of these atomic data types and operations, `sync` provides several useful synchronization primitives.


